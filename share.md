```

@Library('jenkins-devops-cicd-library') _

def ResolveArtifactoryRepository() {
    String repos = 'star-generic-local-dev'
    if (BRANCH_NAME == 'master') {        
        repos = 'star-generic-local-release'
    }
    return repos
}

def ResolveVersion() {
    if (fileExists('.\\build\\version.txt') == false) { 
        def csprojFilePath = '.\\StarTrends\\StarTrendsDashboard\\StarTrendsDashboard.csproj'
        def csprojContent = readFile "${csprojFilePath}"
        def pattern = /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/
        def matcher = (csprojContent =~ pattern)

        if (matcher) {
            def assemblyVersion = matcher[0][1]
            def formattedVersion = assemblyVersion.tokenize('.').take(3).join('.')
            echo "Version info from AssemblyVersion: ${formattedVersion}"
            return formattedVersion
        } else {
            error "AssemblyVersion not found in the .csproj file."
        }
    } else {
        def version = readFile '.\\build\\version.txt'
        echo "Reading Version from file version.txt ${version}"
        return version
    }
}

pipeline {
    agent {
        node {
            label 'windows_2022_VS_2022'
            customWorkspace 'workspace/star/trendsdashboard'
        }
    }
    
    options {
        timestamps()
    }    

    environment {
        PROJECT_NAME = "star.trends.dashboard"
        ALL_PROJECTS = "StarTrendsDashboard,StarTrendsDashboard.Shared"
        BINARY_PUBLISH_PATH = "\\bin\\Release\\net8.0\\win-x64\\publish"
        SOLUTION_FILE = "StarTrends\\StarTrendsDashboard\\StarTrendsDashboard.sln"
        CONFIG_FILE = "build\\Nuget.config"
        
        ARTIFACTORY_CREDS_ID = 'STAR-ARTIFACTORY'
        ARTIFACTORY_CREDS = credentials("${ARTIFACTORY_CREDS_ID}")
        ARTIFACTORY_USER = "${ARTIFACTORY_CREDS_USR}"
        ARTIFACTORY_PASS = "${ARTIFACTORY_CREDS_PSW}"
        ARTIFACTORY_URL = "https://artifactory.cib.echonet/artifactory"
        ARTIFACTORY_REPO = ResolveArtifactoryRepository()
        
        REPO_STAR_NUGET = "https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget"
        REPO_EXTERNAL_NUGET = "https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local"
        REPO_NUGET_CACHE = "https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache"
        
        VERSION_NUMBER = "${ResolveVersion()}.${env.BUILD_NUMBER}"
        
        FORTIFY_URL = "https://fortifyssc.cib.echonet/ssc"
        SECURITY_CHAMPION_MAIL = "security.champion@uk.bnpparibas.com"
        APPLICATION_NAME = "STAR.TRENDS.DASHBOARD"
        NEW_VERSION = "0.0.${env.BUILD_NUMBER}"
        
        MSBuild17 = tool 'MSBuild_17.0'
        MSBUILD = "${MSBuild17}"
    }

    stages {
        stage('Create needed folders') {
            steps {
                bat "dir"
                bat "if not exist \"D:\\data\\trendsdashboard\" (mkdir D:\\data\\trendsdashboard) else (echo Folder already exists!)"
            }
        }
        
        stage('Get commitId and latest tag') {
            steps {
                script {
                    def commitId = bat(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.COMMIT_ID = commitId

                    def latestTag = bat(returnStdout: true, script: 'git describe --always --abbrev=0').trim()
                    env.LATEST_TAG = latestTag

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
                    ALL_PROJECTS.tokenize(',').each {
                        bat "IF EXIST ${it}\\bin RMDIR /S /Q ${it}\\bin"
                    }
                }
            }
        }

        stage('Set version for all projects') {
            steps {
                powershell ''' 
                    $filePath = "./StarTrends/StarTrendsDashboard/*.csproj"
                    Get-ChildItem $filePath -Recurse | ForEach-Object {
                        $newVersionStr = "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>"
                        (Get-Content $_).Replace('<Version>1.1.1.1</Version>',$newVersionStr) | Set-Content $_
                    }
                '''
            }
        }

        stage('Add Nuget credentials') {
            steps {
                bat """
                    nuget Sources Add -Name star-nuget -Source ${REPO_STAR_NUGET} -UserName ${ARTIFACTORY_USER} -Password ${ARTIFACTORY_PASS} -ConfigFile ${CONFIG_FILE}
                    nuget Sources Add -Name external-nuget -Source ${REPO_EXTERNAL_NUGET} -UserName ${ARTIFACTORY_USER} -Password ${ARTIFACTORY_PASS} -ConfigFile ${CONFIG_FILE}
                    nuget Sources Add -Name nuget-cache -Source ${REPO_NUGET_CACHE} -UserName ${ARTIFACTORY_USER} -Password ${ARTIFACTORY_PASS} -ConfigFile ${CONFIG_FILE}
                """
            }
        }
        
        stage('Dotnet clean') {
            steps {
                bat "dotnet clean --configuration Release"
            }
        }
        
        stage('Restore with Nuget') {
            steps {
                bat "\"${MSBUILD}\" -version"
                powershell ''' Get-Content ./build/Nuget.config '''
                bat "dotnet restore ${SOLUTION_FILE} -r win-x64 --configfile ${CONFIG_FILE}"
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
                    withCredentials([
                        string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN')
                    ])  {
                        script {
                            try {
                                sh "echo APPLICATION: ${APPLICATION_NAME}"
                                sh "echo NEW_VERSION: ${NEW_VERSION}" 
                            
                                createNewFortifyApplicationVersionBasedOnLastScan("${FORTIFY_URL}", "${FORTIFY_REST_API_TOKEN}", "${APPLICATION_NAME}", "${NEW_VERSION}")
                            } catch (error) {
                                throw error
                            }
                        }
                    }
                }
            }
        }
        
        stage('Fortify Scan/Analysis') {
            steps {
                withCredentials([
                    string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN'),
                    string(credentialsId: 'fortify-report-token-star', variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
                ])  {
                    script {
                        try {
                            bat "echo APPLICATION: ${APPLICATION_NAME}"
                            bat "echo NEW_VERSION: ${NEW_VERSION}"
                                                        
                            bat 'fortifyupdate -url https://fortifyssc.cib.echonet/ssc -acceptSSLCertificate -acceptKey'
                            bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -clean -b ${APPLICATION_NAME}"
                            bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -Xmx5G -b ${APPLICATION_NAME} -logfile trends-dashboard-fortify.log -debug -verbose \"${MSBUILD}\" ${SOLUTION_FILE} /p:Configuration=Release"
                            bat "scancentral -sscurl ${FORTIFY_URL} -ssctoken %FORTIFY_SCAN_CENTRAL_TOKEN% start -projroot .fortify -b ${APPLICATION_NAME} -email ${SECURITY_CHAMPION_MAIL} -f ${APPLICATION_NAME}.fpr -log ${APPLICATION_NAME}-scan.log -upload -application ${APPLICATION_NAME} -version ${NEW_VERSION} -uptoken %FORTIFY_SCAN_CENTRAL_TOKEN% -scan"
                            bat "echo trends dashboard scan complete"
                            
                        } catch (error) {
                            bat "type trends-dashboard-fortify.log"
                            throw error
                        }
                    }
                }
            }
        }

        stage('Publish Blazor App') {
            steps {
                script {
                    bat "IF EXIST output RMDIR /S /Q output"
                    ALL_PROJECTS.tokenize(',').each {
                        dir("StarTrends\\StarTrendsDashboard\\${it}") {
                            bat "dotnet publish --no-restore -c Release --self-contained true --use-current-runtime true -o ..\\..\\..\\output\\${it}"
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
                    env.INPUT_DIR = "output\\"
                }
                zip zipFile: ZIP_FILEPATH, archive: false, dir: INPUT_DIR
            }
        }

        stage('Deploy to Artifactory') {
            steps {
                script {
                    def buildInfo = Artifactory.newBuildInfo()
                    buildInfo.env.capture = false
                    buildInfo.name = "${ZIP_NAME}"
                    buildInfo.number = "${VERSION_NUMBER}"

                    def server = Artifactory.newServer url: "${ARTIFACTORY_URL}", credentialsId: "${ARTIFACTORY_CREDS_ID}"
                    def uploadSpec = """{
                        "files": [{
                            "pattern": "${ZIP_NAME}.zip",
                            "target": "${ARTIFACTORY_REPO}/com/bnpparibas/${PROJECT_NAME}/${BRANCH_NAME}/${VERSION_NUMBER}/${ZIP_NAME}.zip",
                            "props": "build.commit=${COMMIT_ID};build.version=${VERSION_NUMBER};build.branch=${BRANCH_NAME}"
                        }]}"""
                        
                    try {
                        server.upload spec: uploadSpec, buildInfo: buildInfo
                        server.publishBuildInfo buildInfo
                    }
                    catch (err) {
                        error ("Failed to upload artifacts to Artifactory:" + err)
                    }                    
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}

```
