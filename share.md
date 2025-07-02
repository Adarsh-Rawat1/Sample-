```
@Library('jenkins-devops-cicd-library@master') _

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
        timeout(time: 60, unit: 'MINUTES')
        gitConfig('git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git', 'STAR-BITBUCKET-SSH')
    }    

    environment {
        // Project configuration
        PROJECT_NAME = "star.trends.dashboard"
        PROJECT_REPOSITORY = "ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git"
        PROJECT_BRANCH = "${BRANCH_NAME}"
        
        // Build configuration
        ALL_PROJECTS = "StarTrendsDashboard,StarTrendsDashboard.Shared"
        BINARY_PUBLISH_PATH = "\\bin\\Release\\net8.0\\win-x64\\publish"
        SOLUTION_FILE = "StarTrends\\StarTrendsDashboard\\StarTrendsDashboard.sln"
        CONFIG_FILE = "build\\Nuget.config"
        
        // Artifactory configuration
        ARTIFACTORY_CREDS_ID = 'STAR-ARTIFACTORY'
        ARTIFACTORY_URL = "https://artifactory.cib.echonet/artifactory"
        ARTIFACTORY_REPO = ResolveArtifactoryRepository()
        
        // NuGet sources
        REPO_STAR_NUGET = "https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget"
        REPO_EXTERNAL_NUGET = "https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local"
        REPO_NUGET_CACHE = "https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache"
        
        VERSION_NUMBER = "${ResolveVersion()}.${env.BUILD_NUMBER}"
        
        // Fortify configuration
        FORTIFY_URL = "https://fortifyssc.cib.echonet/ssc"
        SECURITY_CHAMPION_MAIL = "security.champion@uk.bnpparibas.com"
        APPLICATION_NAME = "STAR.TRENDS.DASHBOARD"
        NEW_VERSION = "0.0.${env.BUILD_NUMBER}"
        
        // Build tools
        MSBuild17 = tool 'MSBuild_17.0'
        MSBUILD = "${MSBuild17}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.BRANCH_NAME}"]],
                    extensions: [
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: '.'],
                        [$class: 'CloneOption', depth: 1, noTags: false, shallow: true],
                        [$class: 'LocalBranch', localBranch: "${env.BRANCH_NAME}"],
                        [$class: 'UserIdentity', email: 'jenkins@ci.bnpparibas.com', name: 'Jenkins CI'],
                        [$class: 'CleanBeforeCheckout']
                    ],
                    userRemoteConfigs: [[
                        credentialsId: 'STAR-BITBUCKET-SSH',
                        url: "${PROJECT_REPOSITORY}"
                    ]]
                ])
                
                script {
                    def gitCommit = bat(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.GIT_COMMIT = gitCommit
                    def gitTag = bat(script: 'git describe --tags --always', returnStdout: true).trim()
                    env.GIT_TAG = gitTag
                }
            }
        }

        stage('Initialize') {
            steps {
                script {
                    currentBuild.displayName = "${VERSION_NUMBER}"
                    env.ZIP_NAME = "${PROJECT_NAME}-${VERSION_NUMBER}"
                    
                    // Create needed directories
                    bat """
                        if not exist "D:\\data\\trendsdashboard" mkdir "D:\\data\\trendsdashboard"
                        if exist "output" rmdir /s /q "output"
                    """
                }
            }
        }

        stage('Clean & Restore') {
            steps {
                script {
                    // Clean build directories
                    ALL_PROJECTS.tokenize(',').each {
                        bat "IF EXIST ${it}\\bin RMDIR /S /Q ${it}\\bin"
                    }
                    
                    // Set version in project files
                    powershell '''
                        $filePath = "./StarTrends/StarTrendsDashboard/*.csproj"
                        Get-ChildItem $filePath -Recurse | ForEach-Object {
                            $newVersionStr = "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>"
                            (Get-Content $_).Replace('<Version>1.1.1.1</Version>',$newVersionStr) | Set-Content $_
                        }
                    '''
                    
                    // Configure NuGet sources
                    withCredentials([usernamePassword(credentialsId: "${ARTIFACTORY_CREDS_ID}", usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASS')]) {
                        bat """
                            nuget Sources Add -Name star-nuget -Source ${REPO_STAR_NUGET} -UserName %ARTIFACTORY_USER% -Password %ARTIFACTORY_PASS% -ConfigFile ${CONFIG_FILE}
                            nuget Sources Add -Name external-nuget -Source ${REPO_EXTERNAL_NUGET} -UserName %ARTIFACTORY_USER% -Password %ARTIFACTORY_PASS% -ConfigFile ${CONFIG_FILE}
                            nuget Sources Add -Name nuget-cache -Source ${REPO_NUGET_CACHE} -UserName %ARTIFACTORY_USER% -Password %ARTIFACTORY_PASS% -ConfigFile ${CONFIG_FILE}
                        """
                    }
                    
                    // Restore packages
                    bat "dotnet restore ${SOLUTION_FILE} -r win-x64 --configfile ${CONFIG_FILE}"
                }
            }
        }

        stage('Build') {
            steps {
                bat "dotnet build ${SOLUTION_FILE} -c Release --no-restore"
            }
        }

        stage('Test') {
            steps {
                bat "dotnet test -c Release --no-build --no-restore"
            }
        }

        stage('Fortify Scan') {
            when {
                expression { env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'release' }
            }
            steps {
                withCredentials([
                    string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN'),
                    string(credentialsId: 'fortify-report-token-star', variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
                ]) {
                    script {
                        try {
                            bat """
                                fortifyupdate -url ${FORTIFY_URL} -acceptSSLCertificate -acceptKey
                                sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -clean -b ${APPLICATION_NAME}
                                sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -Xmx5G -b ${APPLICATION_NAME} -logfile trends-dashboard-fortify.log -debug -verbose \"${MSBUILD}\" ${SOLUTION_FILE} /p:Configuration=Release
                                scancentral -sscurl ${FORTIFY_URL} -ssctoken %FORTIFY_SCAN_CENTRAL_TOKEN% start -projroot .fortify -b ${APPLICATION_NAME} -email ${SECURITY_CHAMPION_MAIL} -f ${APPLICATION_NAME}.fpr -log ${APPLICATION_NAME}-scan.log -upload -application ${APPLICATION_NAME} -version ${NEW_VERSION} -uptoken %FORTIFY_SCAN_CENTRAL_TOKEN% -scan
                            """
                        } catch (error) {
                            bat "type trends-dashboard-fortify.log"
                            error "Fortify scan failed: ${error}"
                        }
                    }
                }
            }
        }

        stage('Publish') {
            steps {
                script {
                    ALL_PROJECTS.tokenize(',').each {
                        dir("StarTrends\\StarTrendsDashboard\\${it}") {
                            bat """
                                dotnet publish --no-restore -c Release --self-contained true --use-current-runtime true -o ..\\..\\..\\output\\${it}
                            """
                        }
                    }
                }
            }
        }

        stage('Package') {
            steps {
                script {
                    bat "IF EXIST \"${ZIP_NAME}.zip\" DEL /F \"${ZIP_NAME}.zip\""
                    zip zipFile: "${ZIP_NAME}.zip", archive: false, dir: "output"
                }
            }
        }

        stage('Deploy to Artifactory') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${ARTIFACTORY_CREDS_ID}", usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASS')]) {
                        def buildInfo = Artifactory.newBuildInfo()
                        buildInfo.env.capture = false
                        buildInfo.name = "${PROJECT_NAME}"
                        buildInfo.number = "${VERSION_NUMBER}"

                        def server = Artifactory.server('ARTIFACTORY')
                        server.url = "${ARTIFACTORY_URL}"
                        server.credentialsId = "${ARTIFACTORY_CREDS_ID}"

                        def uploadSpec = """{
                            "files": [{
                                "pattern": "${ZIP_NAME}.zip",
                                "target": "${ARTIFACTORY_REPO}/com/bnpparibas/${PROJECT_NAME}/${BRANCH_NAME}/${VERSION_NUMBER}/${ZIP_NAME}.zip",
                                "props": "build.commit=${GIT_COMMIT};build.version=${VERSION_NUMBER};build.branch=${BRANCH_NAME}"
                            }]
                        }"""

                        try {
                            server.upload(uploadSpec, buildInfo)
                            server.publishBuildInfo(buildInfo)
                        } catch (err) {
                            error "Failed to upload artifacts to Artifactory: ${err}"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Build ${VERSION_NUMBER} completed successfully"
        }
        failure {
            echo "Build ${VERSION_NUMBER} failed"
        }
    }
}
```
