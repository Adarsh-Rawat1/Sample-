```
@Library('jenkins-devops-cicd-library') _

// Selects the correct Artifactory repository based on branch
def ResolveArtifactoryRepository() {
    String repos = 'star-generic-local-dev'
    if (env.BRANCH_NAME == 'master') {
        repos = 'star-generic-local-release'
    }
    return repos
}

// Reads version from build/version.txt or from the .csproj AssemblyVersion
def ResolveVersion() {
    if (!fileExists('.\\build\\version.txt')) {
        def csprojFilePath = '.\\bnpp.star.openapi\\bnpp.star.openapi.csproj'
        def csprojContent = readFile csprojFilePath
        def pattern = /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/
        def matcher = (csprojContent =~ pattern)
        if (matcher) {
            def assemblyVersion = matcher[0][1]
            def formattedVersion = assemblyVersion.tokenize('.').take(3).join('.')
            echo "Version info from AssemblyVersion: ${formattedVersion}"
            return formattedVersion
        } else {
            error "AssemblyVersion not found in ${csprojFilePath}"
        }
    } else {
        def version = readFile('.\\build\\version.txt').trim()
        echo "Reading Version from file version.txt: ${version}"
        return version
    }
}

pipeline {
    agent {
        node {
            label 'windows_2022_VS_2022'
            customWorkspace 'workspace/star/reportingapi'
        }
    }
    options {
        timestamps()
    }
    environment {
        // project & build
        PROJECT_NAME               = "bnpp.star.reporting.api"
        ALL_PROJECTS               = "bnpp.star.data.extractor,bnpp.star.web.service"
        SOLUTION_FILE              = "bnpp.reporting.api.sln"
        CONFIG_FILE                = "build\\Nuget.config"

        // Artifactory credentials
        ARTIFACTORY_CREDS_ID       = 'STAR-ARTIFACTORY'
        ARTIFACTORY_CREDS          = credentials("${ARTIFACTORY_CREDS_ID}")
        ARTIFACTORY_USER           = "${ARTIFACTORY_CREDS_USR}"
        ARTIFACTORY_PASS           = "${ARTIFACTORY_CREDS_PSW}"
        ARTIFACTORY_URL            = "https://artifactory.cib.echonet/artifactory"

        // NuGet feeds
        REPO_STAR_NUGET            = "https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget"
        REPO_EXTERNAL_NUGET        = "https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local"
        REPO_REMOTE_CACHE          = "https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache"

        // Fortify
        FORTIFY_URL                = "https://fortifyssc.cib.echonet/ssc"
        SECURITY_CHAMPION_MAIL     = "harpreet.nanra@uk.bnpparibas.com"
        APPLICATION_REPORTING_API  = "STAR.HUDSON"

        // MSBuild tool
        MSBuild17                  = tool 'MSBuild_17.0'
        MSBUILD                    = "${MSBuild17}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Initialize') {
            steps {
                script {
                    // capture branch, version, artifact names
                    env.BRANCH_NAME      = env.GIT_BRANCH ?: env.BRANCH_NAME
                    env.ARTIFACTORY_REPO = ResolveArtifactoryRepository()
                    env.VERSION_NUMBER   = "${ResolveVersion()}.${BUILD_NUMBER}"
                    currentBuild.displayName = env.VERSION_NUMBER
                    env.ZIP_NAME         = "${PROJECT_NAME}-${VERSION_NUMBER}"
                }
            }
        }

        stage('Create needed folders') {
            steps {
                bat 'if not exist "D:\\data\\reportingapi" mkdir D:\\data\\reportingapi'
            }
        }

        stage('Delete build directories') {
            steps {
                script {
                    ALL_PROJECTS.tokenize(',').each { proj ->
                        bat "if exist ${proj}\\bin rmdir /S /Q ${proj}\\bin"
                    }
                }
            }
        }

        stage('Set version in csproj') {
            steps {
                powershell '''
                  Get-ChildItem . -Recurse -Filter *.csproj | ForEach-Object {
                    (Get-Content $_.FullName) -replace '<Version>1.1.1.1</Version>', "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>" | Set-Content $_.FullName
                  }
                '''
            }
        }

        stage('Add NuGet credentials') {
            steps {
                script {
                    // start with a fresh NuGet.config
                    writeFile file: "${CONFIG_FILE}", text: '''<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
  </packageSources>
  <packageSourceCredentials>
  </packageSourceCredentials>
</configuration>'''
                    // define and add all feeds
                    def feeds = [
                      [name:'star-nuget',           url:REPO_STAR_NUGET],
                      [name:'external-nuget-local', url:REPO_EXTERNAL_NUGET],
                      [name:'nuget-remote-cache',    url:REPO_REMOTE_CACHE]
                    ]
                    feeds.each { feed ->
                        bat """
                          nuget sources Remove -Name ${feed.name} -ConfigFile ${CONFIG_FILE} || echo 'no existing ${feed.name}'
                          nuget sources Add    -Name ${feed.name} ^
                               -Source ${feed.url} ^
                               -UserName ${ARTIFACTORY_USER} ^
                               -Password ${ARTIFACTORY_PASS} ^
                               -ConfigFile ${CONFIG_FILE}
                        """
                    }
                }
            }
        }

        stage('Dotnet clean') {
            steps {
                bat "dotnet clean --configuration Release"
            }
        }

        stage('Restore with NuGet') {
            steps {
                bat "\"${MSBUILD}\" -version"
                bat "dotnet restore ${SOLUTION_FILE} -r win-x64 --configfile ${CONFIG_FILE}"
            }
        }

        stage('Remove override settings') {
            steps {
                powershell '''
                  Get-ChildItem . -Recurse -Filter *.json | ForEach-Object {
                    (Get-Content $_.FullName) -replace 'FileOverrideLocation','FileOverrideLocation_' | Set-Content $_.FullName
                  }
                '''
            }
        }

        stage('Build') {
            steps {
                bat "dotnet build ${SOLUTION_FILE} -c Release"
            }
        }

        stage('Run unit tests') {
            steps {
                bat "dotnet test -c Release --no-build --no-restore"
            }
        }

        stage('Create Fortify Version') {
            steps {
                node('Bnpp-Maven3-SecOps') {
                    withCredentials([string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN')]) {
                        script {
                            createNewFortifyApplicationVersionBasedOnLastScan(
                                "${FORTIFY_URL}",
                                "${FORTIFY_REST_API_TOKEN}",
                                "${APPLICATION_REPORTING_API}",
                                "0.0.${BUILD_NUMBER}"
                            )
                        }
                    }
                }
            }
        }

        stage('Fortify Scan/Analysis') {
            steps {
                withCredentials([
                  string(credentialsId: 'fortify-rest-api-token-star',   variable: 'FORTIFY_REST_API_TOKEN'),
                  string(credentialsId: 'fortify-report-token-star',     variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
                ]) {
                    script {
                        try {
                            bat "fortifyupdate -url ${FORTIFY_URL} -acceptSSLCertificate -acceptKey"
                            bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -clean -b ${APPLICATION_REPORTING_API}"
                            bat """sourceanalyzer \
                                -Dcom.fortify.sca.ProjectRoot=.fortify \
                                -Xmx5G \
                                -b ${APPLICATION_REPORTING_API} \
                                -logfile reporting-api-fortify.log \
                                -debug -verbose \
                                "${MSBUILD}" ${SOLUTION_FILE} /p:Configuration=Release"""
                            bat """scancentral \
                                -sscurl ${FORTIFY_URL} \
                                -ssctoken ${FORTIFY_SCAN_CENTRAL_TOKEN} \
                                start -projroot .fortify \
                                -b ${APPLICATION_REPORTING_API} \
                                -email ${SECURITY_CHAMPION_MAIL} \
                                -f ${APPLICATION_REPORTING_API}.fpr \
                                -log ${APPLICATION_REPORTING_API}-scan.log \
                                -upload -application ${APPLICATION_REPORTING_API} \
                                -version 0.0.${BUILD_NUMBER} \
                                -uptoken ${FORTIFY_SCAN_CENTRAL_TOKEN} \
                                -scan"""
                        } catch (err) {
                            bat "type reporting-api-fortify.log"
                            error "Fortify scan failed: ${err}"
                        }
                    }
                }
            }
        }

        stage('Build EXE binaries') {
            steps {
                script {
                    bat "if exist output rmdir /S /Q output"
                    ALL_PROJECTS.tokenize(',').each { proj ->
                        dir(proj) {
                            bat "dotnet publish --no-restore -c Release --self-contained true --use-current-runtime true -o ../output/${proj}"
                        }
                    }
                }
            }
        }

        stage('Zip') {
            steps {
                bat "if exist ${ZIP_NAME}.zip del /F ${ZIP_NAME}.zip"
                script {
                    env.ZIP_FILEPATH = "${ZIP_NAME}.zip"
                    env.INPUT_DIR     = "output\\"
                }
                zip zipFile: ZIP_FILEPATH, archive: false, dir: INPUT_DIR
            }
        }

        stage('Deploy to Artifactory') {
            steps {
                script {
                    def buildInfo = Artifactory.newBuildInfo()
                    buildInfo.env.capture = false
                    buildInfo.name   = "${ZIP_NAME}"
                    buildInfo.number = "${VERSION_NUMBER}"

                    def server = Artifactory.newServer(
                        url: "${ARTIFACTORY_URL}",
                        credentialsId: "${ARTIFACTORY_CREDS_ID}"
                    )

                    def uploadSpec = """{
                      "files": [{
                        "pattern": "${ZIP_NAME}.zip",
                        "target": "${ARTIFACTORY_REPO}/com/bnpparibas/${PROJECT_NAME}/${BRANCH_NAME}/${VERSION_NUMBER}/${ZIP_NAME}.zip",
                        "props": "build.commit=${COMMIT_ID};build.version=${VERSION_NUMBER};build.branch=${BRANCH_NAME}"
                      }]
                    }"""

                    server.upload spec: uploadSpec, buildInfo: buildInfo
                    server.publishBuildInfo buildInfo
                }
            }
        }
    }
}
```
