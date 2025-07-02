@Library('jenkins-devops-cicd-library') _

// --- Helper functions from shared library ---
def ListNugetSourcesFromArtifactory() {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: 'dotnet nuget list source')
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def RemoveNugetSourcesFromArtifactory(name) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: "dotnet nuget remove source ${name}")
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def UpdateNugetSourcesFromArtifactory(source, name) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: "dotnet nuget update source ${name} --source ${source} --username ${USERNAME} --password ${PASSWORD} --store-password-in-clear-text")
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def AddNugetSourcesFromArtifactory(source, name) {
    RemoveNugetSourcesFromArtifactory(name)
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: "dotnet nuget add source ${source} --name ${name} --username ${USERNAME} --password ${PASSWORD} --store-password-in-clear-text")
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def GenerateArtifactoryUploadSpecification(config) {
    def spec = [
        'pattern': "${config.path}",
        'target' : "${config.repository}/com/bnpparibas/${config.name}/${config.branch}/${config.version}/${config.fullname}",
        'props'  : "build.commit=${GIT_COMMIT};build.version=${config.version};build.branch=${GIT_BRANCH}"
    ]
    return writeJSON(returnText: true, json: spec)
}

def PromotePackageToArtifactory(config) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            def buildInfo = Artifactory.newBuildInfo()
            buildInfo.env.capture = false
            buildInfo.name   = config.name
            buildInfo.number = config.version

            def server = Artifactory.newServer url: ARTIFACTORY_URL, credentialsId: ARTIFACTORY_CREDS_ID
            def uploadSpec = "{\"files\": [${GenerateArtifactoryUploadSpecification(config)}]}"

            timeout(time: 10, unit: 'MINUTES') {
                retry(3) {
                    server.upload           spec: uploadSpec, buildInfo: buildInfo
                    server.publishBuildInfo buildInfo
                }
            }
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def ResolveArtifactoryRepository() {
    String repos = 'star-generic-local-dev'
    if (BRANCH_NAME == 'master') {
        repos = 'star-generic-local-release'
    }
    return repos
}

def ResolveVersion() {
    if (!fileExists('.\\build\\version.txt')) {
        def csprojFilePath = '.\\bnpp.star.openapi\\bnpp.star.openapi.csproj'
        def csprojContent  = readFile(csprojFilePath)
        def matcher        = (csprojContent =~ /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/)
        if (matcher) {
            def assemblyVersion = matcher[0][1]
            def formattedVersion = assemblyVersion.tokenize('.').take(3).join('.')
            echo "Version info from AssemblyVersion: ${formattedVersion}"
            return formattedVersion
        } else {
            error "AssemblyVersion not found in the .csproj file."
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
            label           'windows_2022_VS_2022'
            customWorkspace 'workspace/star/reportingapi'
        }
    }

    options {
        timestamps()
    }

    parameters {
        string(
            name: 'VersionNumberPrefix',
            defaultValue: '1.0.0',
            description: 'The prefix for the version number (will be concatenated with BUILD_NUMBER)'
        )
        choice(
            name: 'Config',
            choices: ['Release', 'Debug'],
            description: 'Build configuration to use'
        )
    }

    environment {
        // Project & Git
        PROJECT_REPOSITORY         = 'ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git'
        PROJECT_BRANCH             = 'Dev-StarTrends'
        PROJECT_NAME               = 'bnpp.star.reporting.api'
        ALL_PROJECTS               = 'bnpp.star.data.extractor,bnpp.star.web.service'

        // Solution & NuGet config
        SOLUTION_FILE              = 'bnpp.reporting.api.sln'
        CONFIG_FILE                = 'build\\Nuget.config'
        BINARY_PUBLISH_PATH        = '\\bin\\Release\\net8.0\\win-x64\\publish'

        // Versioning
        ARTIFACTORY_CREDS_ID       = 'STAR-ARTIFACTORY'
        ARTIFACTORY_CREDS          = credentials("${ARTIFACTORY_CREDS_ID}")
        ARTIFACTORY_USER           = "${ARTIFACTORY_CREDS_USR}"
        ARTIFACTORY_PASS           = "${ARTIFACTORY_CREDS_PSW}"
        ARTIFACTORY_URL            = 'https://artifactory.cib.echonet/artifactory'
        ARTIFACTORY_REPO           = ResolveArtifactoryRepository()
        VERSION_NUMBER             = "${ResolveVersion()}.${BUILD_NUMBER}"

        // NuGet feeds
        REPO_STAR_NUGET            = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        REPO_EXTERNAL_NUGET        = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        NUGET_REMOTE_CACHE         = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'

        // Fortify
        FORTIFY_URL                = 'https://fortifyssc.cib.echonet/ssc'
        SECURITY_CHAMPION_MAIL     = 'harpreet.nanra@uk.bnpparibas.com'
        APPLICATION_REPORTING_API  = 'STAR.HUDSON'
        NEW_VERSION                = "0.0.${BUILD_NUMBER}"

        // MSBuild tool
        MSBuild17                  = tool 'MSBuild_17.0'
        MSBUILD                    = "${MSBuild17}"
    }

    stages {
        stage('Create needed folders') {
            steps {
                bat 'dir'
                bat 'if not exist "D:\\data\\reportingapi" (mkdir D:\\data\\reportingapi) else (echo Folder already exists!)'
            }
        }

        stage('Get commitId and latest tag') {
            steps {
                script {
                    env.COMMIT_ID = bat(returnStdout: true, script: '@git rev-parse --short HEAD').trim()
                    env.LATEST_TAG = bat(returnStdout: true, script: '@git describe --always --abbrev=0').trim()
                    echo "Application Version: ${VERSION_NUMBER}"
                }
            }
        }

        stage('Get artifact version') {
            steps {
                script {
                    currentBuild.displayName = "${VERSION_NUMBER}"
                    env.ZIP_NAME = "${PROJECT_NAME}-${VERSION_NUMBER}"
                }
            }
        }

        stage('Delete build directory') {
            steps {
                script {
                    ALL_PROJECTS.tokenize(',').each { proj ->
                        bat "IF EXIST ${proj}\\bin RMDIR /S /Q ${proj}\\bin"
                    }
                }
            }
        }

        stage('Set version for all dlls and exes') {
            steps {
                powershell '''
                    Get-ChildItem -Path . -Filter *.csproj -Recurse |
                      ForEach-Object {
                        (Get-Content $_) |
                          ForEach-Object { $_ -replace '<Version>.*?</Version>',
                            "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>" } |
                          Set-Content $_
                      }
                '''
            }
        }

        stage('Add NuGet credentials') {
            steps {
                bat "nuget sources Add -Name star-nuget -Source ${REPO_STAR_NUGET} -UserName ${ARTIFACTORY_USER} -Password ${ARTIFACTORY_PASS} -ConfigFile ${CONFIG_FILE}"
            }
        }

        stage('Dotnet clean') {
            steps {
                bat "dotnet clean --configuration ${params.Config}"
            }
        }

        stage('Restore with NuGet') {
            steps {
                bat "\"${MSBUILD}\" -version"
                powershell 'Get-Content .\\build\\Nuget.config'
                bat "dotnet restore ${SOLUTION_FILE} -r win-x64 --configfile ${CONFIG_FILE}"
            }
        }

        stage('Remove all override config file settings') {
            steps {
                powershell '''
                    Get-ChildItem -Path . -Filter *.json -Recurse |
                      ForEach-Object {
                        (Get-Content $_) -replace 'FileOverrideLocation','FileOverrideLocation_' |
                          Set-Content $_
                      }
                '''
            }
        }

        stage('Build') {
            steps {
                bat "dotnet build ${SOLUTION_FILE} -c ${params.Config}"
            }
        }

        stage('Run unit tests') {
            steps {
                bat "dotnet test -c ${params.Config} --no-build --no-restore"
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
                              "${NEW_VERSION}"
                            )
                        }
                    }
                }
            }
        }

        stage('Fortify Scan/Analysis') {
            steps {
                withCredentials([
                    string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN'),
                    string(credentialsId: 'fortify-report-token-star',   variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
                ]) {
                    script {
                        bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -clean -b ${APPLICATION_REPORTING_API}"
                        bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -Xmx5G -b ${APPLICATION_REPORTING_API} -logfile reporting-api-fortify.log -verbose \"${MSBUILD}\" ${SOLUTION_FILE} /p:Configuration=${params.Config}"
                        bat "scancentral -sscurl ${FORTIFY_URL} -ssctoken ${FORTIFY_SCAN_CENTRAL_TOKEN} start -projroot .fortify -b ${APPLICATION_REPORTING_API} -email ${SECURITY_CHAMPION_MAIL} -f ${APPLICATION_REPORTING_API}.fpr -log ${APPLICATION_REPORTING_API}-scan.log -upload -application ${APPLICATION_REPORTING_API} -version ${NEW_VERSION} -uptoken ${FORTIFY_SCAN_CENTRAL_TOKEN} -scan"
                        bat "echo reporting api scan complete"
                    }
                }
            }
        }

        stage('Build EXE binaries') {
            steps {
                script {
                    bat "IF EXIST output RMDIR /S /Q output"
                    ALL_PROJECTS.tokenize(',').each { proj ->
                        dir(proj) {
                            bat "dotnet publish --no-restore -c ${params.Config} --self-contained true --use-current-runtime -o ../output/${proj}"
                        }
                    }
                }
            }
        }

        stage('Zip') {
            steps {
                bat "IF EXIST ${ZIP_NAME}.zip DEL /F ${ZIP_NAME}.zip"
                script {
                    env.ZIP_FILEPATH = "${ZIP_NAME}.zip"
                    env.INPUT_DIR    = "output\\"
                }
                zip archive: false, dir: env.INPUT_DIR, zipFile: env.ZIP_FILEPATH
            }
        }

        stage('Deploy to Artifactory') {
            steps {
                script {
                    def cfg = [
                        repository: "${ARTIFACTORY_REPO}",
                        name      : "${PROJECT_NAME}",
                        branch    : "${BRANCH_NAME}",
                        version   : "${VERSION_NUMBER}",
                        fullname  : "${ZIP_NAME}.zip",
                        path      : "${WORKSPACE}\\${ZIP_NAME}.zip"
                    ]
                    PromotePackageToArtifactory(cfg)
                }
            }
        }
    }
}
