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
        skipDefaultCheckout()
        preserveStashes() // Preserve stashes across builds
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
        
        // Version will be set in initialize stage
        VERSION = ""
        
        // Fortify configuration
        FORTIFY_URL = "https://fortifyssc.cib.echonet/ssc"
        SECURITY_CHAMPION_MAIL = "security.champion@uk.bnpparibas.com"
        APPLICATION_NAME = "STAR.TRENDS.DASHBOARD"
        
        // Build tools
        MSBuild17 = tool 'MSBuild_17.0'
        MSBUILD = "${MSBuild17}"
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    // Set version early to ensure it's available in post section
                    def baseVersion = ResolveVersion()
                    env.VERSION_NUMBER = "${baseVersion}.${env.BUILD_NUMBER}"
                    env.NEW_VERSION = "0.0.${env.BUILD_NUMBER}"
                    currentBuild.displayName = "${env.VERSION_NUMBER}"
                    env.ZIP_NAME = "${PROJECT_NAME}-${env.VERSION_NUMBER}"
                    
                    // Create needed directories
                    bat """
                        if not exist "D:\\data\\trendsdashboard" mkdir "D:\\data\\trendsdashboard"
                        if exist "output" rmdir /s /q "output"
                        mkdir output
                    """
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${BRANCH_NAME}"]],
                    extensions: [
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: '.'],
                        [$class: 'CloneOption', depth: 1, noTags: false, shallow: true],
                        [$class: 'LocalBranch', localBranch: "${BRANCH_NAME}"],
                        [$class: 'UserIdentity', email: 'jenkins@ci.bnpparibas.com', name: 'Jenkins CI'],
                        [$class: 'CleanBeforeCheckout'],
                        [$class: 'IgnoreNotifyCommit'],
                        [$class: 'DisableRemotePoll']
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
                    echo "Checked out commit: ${GIT_COMMIT} (tag: ${GIT_TAG})"
                }
            }
        }

        // ... (keep all your other stages exactly the same) ...
    }

    post {
        always {
            script {
                echo "Cleaning up workspace"
                node('windows_2022_VS_2022') {
                    cleanWs()
                }
            }
        }
        success {
            echo "Build ${currentBuild.displayName} completed successfully"
        }
        failure {
            script {
                def displayName = currentBuild.displayName ?: "UNKNOWN_VERSION"
                echo "Build ${displayName} failed"
                emailext (
                    subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                    body: """Check console output at ${env.BUILD_URL}console""",
                    to: "${SECURITY_CHAMPION_MAIL}"
                )
            }
        }
    }
}

```
