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
        gitConfig('git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git', 'STAR-BITBUCKET-SSH')
        timeout(time: 60, unit: 'MINUTES')
    }    

    environment {
        // Git configuration
        GIT_SSH_COMMAND = 'ssh -o StrictHostKeyChecking=no -i /path/to/ssh/key'
        
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
        ARTIFACTORY_CREDS = credentials("${ARTIFACTORY_CREDS_ID}")
        ARTIFACTORY_USER = "${ARTIFACTORY_CREDS_USR}"
        ARTIFACTORY_PASS = "${ARTIFACTORY_CREDS_PSW}"
        ARTIFACTORY_URL = "https://artifactory.cib.echonet/artifactory"
        ARTIFACTORY_REPO = ResolveArtifactoryRepository()
        
        // NuGet sources
        REPO_STAR_NUGET = "https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget"
        REPO_EXTERNAL_NUGET = "https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local"
        REPO_NUGET_CACHE = "https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache"
        
        VERSION_NUMBER = "${ResolveVersion()}.${env.BUILD_NUMBER}"
        
        // Fortify specific
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
                    branches: [[name: "*/${PROJECT_BRANCH}"]],
                    extensions: [
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: '.'],
                        [$class: 'CloneOption', depth: 1, noTags: false, shallow: true],
                        [$class: 'LocalBranch', localBranch: "${PROJECT_BRANCH}"],
                        [$class: 'UserIdentity', email: 'jenkins@ci.bnpparibas.com', name: 'Jenkins CI']
                    ],
                    userRemoteConfigs: [[
                        credentialsId: 'STAR-BITBUCKET-SSH',
                        url: "${PROJECT_REPOSITORY}"
                    ]]
                ])
                
                script {
                    def gitCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.COMMIT_ID = gitCommit
                    def gitTag = sh(script: 'git describe --tags --always', returnStdout: true).trim()
                    env.LATEST_TAG = gitTag
                }
            }
        }

        stage('Initialize') {
            steps {
                script {
                    currentBuild.displayName = "${VERSION_NUMBER}"
                    env.ZIP_NAME = "${PROJECT_NAME}-${VERSION_NUMBER}"
                }
                
                // Create needed directories
                bat """
                    if not exist "D:\\data\\trendsdashboard" mkdir "D:\\data\\trendsdashboard"
                    if exist "output" rmdir /s /q "output"
                """
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
                    bat """
                        nuget Sources Add -Name star-nuget -Source ${REPO_STAR_NUGET} -UserName ${ARTIFACTORY_USER} -Password ${ARTIFACTORY_PASS} -ConfigFile ${CONFIG_FILE}
                        nuget Sources Add -Name external-nuget -Source ${REPO_EXTERNAL_NUGET} -UserName ${ARTIFACTORY_USER} -Password ${ARTIFACTORY_PASS} -ConfigFile ${CONFIG_FILE}
                        nuget Sources Add -Name nuget-cache -Source ${REPO_NUGET_CACHE} -UserName ${ARTIFACTORY_USER} -Password ${ARTIFACTORY_PASS} -ConfigFile ${CONFIG_FILE}
                    """
                    
                    // Restore packages
                    bat "dotnet restore ${SOLUTION_FILE} -r win-x64 --configfile ${CONFIG_FILE}"
                }
            }
        }

        // Remaining stages (Build, Test, Fortify, Publish, Deploy) would follow here
        // ... [previous stages you had for build, test, fortify, etc.]
    }
}
