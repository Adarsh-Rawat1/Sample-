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
    if (!fileExists('build/version.txt')) {
        def csprojFile = 'StarTrends/StarTrendsDashboard/StarTrendsDashboard.csproj'
        def content = readFile(csprojFile)
        def matcher = (content =~ /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/)
        if (matcher) {
            def ver = matcher[0][1].tokenize('.').take(3).join('.')
            return ver
        } else {
            error "AssemblyVersion not found"
        }
    } else {
        def ver = readFile('build/version.txt').trim()
        return ver
    }
}

pipeline {
    agent {
        node {
            label 'windows_2016_VS_2022'
            customWorkspace 'workspace/star'
        }
    }
    options {
        timestamps()
    }
    environment {
        PROJECT_NAME = 'star.trends.dashboard'
        ALL_PROJECTS = 'StarTrendsDashboard,StarTrendsDashboard.Shared'
        SOLUTION_FILE = 'StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln'
        CONFIG_FILE = 'StarTrends/StarTrendsDashboard/Nuget.config'
        BINARY_PUBLISH_PATH = '\\bin\\Release\\net8.0\\win-x64\\publish'
        ARTIFACTORY_CREDS_ID = 'STAR-ARTIFACTORY'
        ARTIFACTORY_CREDS = credentials("${ARTIFACTORY_CREDS_ID}")
        ARTIFACTORY_USER = "${ARTIFACTORY_CREDS_USR}"
        ARTIFACTORY_PASS = "${ARTIFACTORY_CREDS_PSW}"
        ARTIFACTORY_URL = 'https://artifactory.cib.echonet/artifactory'
        ARTIFACTORY_REPO = ResolveArtifactoryRepository()
        REPO_STAR_NUGET = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        VERSION_NUMBER = "${ResolveVersion()}.${BUILD_NUMBER}"
        NEW_VERSION = "${VERSION_NUMBER}"
        MSBUILD_PATH = tool('MSBuild_17.0')
        MSBUILD = "${MSBUILD_PATH}"
        COMMIT_ID = ''
        LATEST_TAG = ''
        APPLICATION_ID = 'STAR.TRENDS'
    }
    stages {
        stage('Prepare Environment') {
            steps {
                bat 'if not exist "D:\\data\\StarTrends" mkdir D:\\data\\StarTrends'
                script {
                    COMMIT_ID = bat(returnStdout: true, script: '@git rev-parse --short HEAD').trim()
                    LATEST_TAG = bat(returnStdout: true, script: '@git describe --always --abbrev=0').trim()
                    currentBuild.displayName = "${VERSION_NUMBER}"
                    env.ZIP_NAME = "${PROJECT_NAME}-${VERSION_NUMBER}"
                }
            }
        }
        stage('Clean Build Folders') {
            steps {
                script {
                    ALL_PROJECTS.tokenize(',').each {
                        bat "IF EXIST ${it}\\bin RMDIR /S /Q ${it}\\bin"
                        bat "IF EXIST ${it}\\obj RMDIR /S /Q ${it}\\obj"
                    }
                }
            }
        }
        stage('Update Project Version') {
            steps {
                powershell '''
                    Get-ChildItem .\\StarTrends\\StarTrendsDashboard -Recurse -Filter *.csproj | ForEach-Object {
                        (Get-Content $_) -replace '<Version>.*</Version>', "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>" |
                        Set-Content $_
                    }
                '''
            }
        }
        stage('Configure NuGet') {
            steps {
                bat "nuget sources add -Name star-nuget -Source ${REPO_STAR_NUGET} -UserName ${ARTIFACTORY_USER} -Password ${ARTIFACTORY_PASS} -ConfigFile ${CONFIG_FILE}"
            }
        }
        stage('Clean Solution') {
            steps {
                bat "dotnet clean ${SOLUTION_FILE} -c Release"
            }
        }
        stage('Restore Packages') {
            steps {
                bat "\"${MSBUILD}\" -version"
                bat "dotnet restore ${SOLUTION_FILE} -r win-x64 --configfile ${CONFIG_FILE}"
            }
        }
        stage('Remove JSON Overrides') {
            steps {
                powershell '''
                    Get-ChildItem .\\StarTrends\\StarTrendsDashboard -Recurse -Filter *.json | ForEach-Object {
                        (Get-Content $_) -replace 'FileOverrideLocation','FileOverrideLocation_' | Set-Content $_
                    }
                '''
            }
        }
        stage('Build') {
            steps {
                bat "dotnet build ${SOLUTION_FILE} -c Release"
            }
        }
        stage('Test') {
            steps {
                bat "dotnet test ${SOLUTION_FILE} -c Release --no-build --no-restore"
            }
        }
        stage('Fortify Setup') {
            steps {
                node('Bnpp-Maven3-SecOps') {
                    withCredentials([string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN')]) {
                        sh "createNewFortifyApplicationVersionBasedOnLastScan(\"${ARTIFACTORY_URL}\",\"${FORTIFY_REST_API_TOKEN}\",\"${APPLICATION_ID}\",\"${NEW_VERSION}\")"
                    }
                }
            }
        }
        stage('Fortify Scan') {
            steps {
                withCredentials([
                    string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN'),
                    string(credentialsId: 'fortify-report-token-star', variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
                ]) {
                    bat "sourceanalyzer -clean -b ${APPLICATION_ID}"
                    bat "sourceanalyzer -b ${APPLICATION_ID} -Xmx5G \"${MSBUILD}\" ${SOLUTION_FILE} /p:Configuration=Release"
                    bat "scancentral -sscurl ${ARTIFACTORY_URL} -ssctoken ${FORTIFY_SCAN_CENTRAL_TOKEN} start -b ${APPLICATION_ID} -email security.champion@bnpparibas.com -f ${APPLICATION_ID}.fpr -upload -application ${APPLICATION_ID} -version ${NEW_VERSION}"
                }
            }
        }
        stage('Publish Binaries') {
            steps {
                script {
                    ALL_PROJECTS.tokenize(',').each {
                        dir(it) {
                            bat "dotnet publish -c Release --self-contained true --use-current-runtime -o ../output/${it}"
                        }
                    }
                }
            }
        }
        stage('Package Artifacts') {
            steps {
                bat "IF EXIST ${ZIP_NAME}.zip DEL /F ${ZIP_NAME}.zip"
                zip zipFile: "${ZIP_NAME}.zip", dir: 'output'
            }
        }
        stage('Upload to Artifactory') {
            steps {
                script {
                    def buildInfo = Artifactory.newBuildInfo()
                    buildInfo.env.capture = false
                    buildInfo.name = "${ZIP_NAME}"
                    buildInfo.number = "${VERSION_NUMBER}"
                    def server = Artifactory.newServer url: "${ARTIFACTORY_URL}", credentialsId: "${ARTIFACTORY_CREDS_ID}"
                    def uploadSpec = """{"files":[{"pattern":"${ZIP_NAME}.zip","target":"${ARTIFACTORY_REPO}/com/bnpparibas/${PROJECT_NAME}/${BRANCH_NAME}/${VERSION_NUMBER}/${ZIP_NAME}.zip","props":"build.commit=${COMMIT_ID};build.version=${VERSION_NUMBER};build.branch=${BRANCH_NAME}"}]}"""
                    server.upload spec: uploadSpec, buildInfo: buildInfo
                    server.publishBuildInfo buildInfo
                }
            }
        }
    }
}

```
