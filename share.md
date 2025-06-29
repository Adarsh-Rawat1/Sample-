```
@Library('jenkins-devops-cicd-library') _

// ——————————————————————————————————————————————————————
// Helpers to manage your Artifactory NuGet feeds
// ——————————————————————————————————————————————————————
def ListNugetSourcesFromArtifactory() {
    try {
        withCredentials([usernamePassword(credentialsId: 'STAR-ARTIFACTORY',
                                          usernameVariable: 'USERNAME',
                                          passwordVariable: 'PASSWORD')]) {
            bat returnStdout: true, script: 'dotnet nuget list source'
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def RemoveNugetSourcesFromArtifactory(name) {
    try {
        withCredentials([usernamePassword(credentialsId: 'STAR-ARTIFACTORY',
                                          usernameVariable: 'USERNAME',
                                          passwordVariable: 'PASSWORD')]) {
            bat returnStdout: true, script: "dotnet nuget remove source ${name}"
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def AddNugetSourcesFromArtifactory(source, name) {
    // ensure old entry is gone
    RemoveNugetSourcesFromArtifactory(name)
    try {
        withCredentials([usernamePassword(credentialsId: 'STAR-ARTIFACTORY',
                                          usernameVariable: 'USERNAME',
                                          passwordVariable: 'PASSWORD')]) {
            bat returnStdout: true, script: """
                dotnet nuget add source ${source} \
                    --name ${name} \
                    --username ${USERNAME} \
                    --password ${PASSWORD} \
                    --store-password-in-clear-text
            """
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

// ——————————————————————————————————————————————————————
// Helpers for Artifactory upload & promotion
// ——————————————————————————————————————————————————————
def GenerateArtifactoryUploadSpecification(config) {
    def spec = [
        pattern:  config.path,
        target:   "${config.repository}/com/bnpparibas/${config.name}/${config.branch}/${config.version}/${config.fullname}",
        props:    "build.commit=${env.GIT_COMMIT};build.version=${config.version};build.branch=${env.GIT_BRANCH}"
    ]
    return writeJSON(returnText: true, json: spec)
}

def PromotePackageToArtifactory(config) {
    try {
        withCredentials([usernamePassword(credentialsId: 'STAR-ARTIFACTORY',
                                          usernameVariable: 'USERNAME',
                                          passwordVariable: 'PASSWORD')]) {
            def buildInfo = Artifactory.newBuildInfo()
            buildInfo.env.capture = false
            buildInfo.name   = config.name
            buildInfo.number = config.version

            def server = Artifactory.newServer(
                url: 'https://artifactory.cib.echonet/artifactory',
                credentialsId: 'STAR-ARTIFACTORY'
            )
            def uploadSpec = """{
                "files":[${GenerateArtifactoryUploadSpecification(config)}]
            }"""

            timeout(time:10, unit:'MINUTES') {
                retry(3) {
                    server.upload spec: uploadSpec, buildInfo: buildInfo
                    server.publishBuildInfo buildInfo
                }
            }
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

// ——————————————————————————————————————————————————————
// Star Trends Dashboard Pipeline
// ——————————————————————————————————————————————————————
pipeline {
    agent {
        node {
            label 'windows_2016_VS_2022'
            customWorkspace 'workspace/star'
        }
    }

    parameters {
        string(name: 'VersionNumberPrefix',
               defaultValue: '1.0.0',
               description: 'Version prefix (will be appended with build number)')
        choice(name: 'Config',
               choices: ['Release','Debug'],
               description: 'Build configuration')
    }

    environment {
        // Git
        PROJECT_REPOSITORY         = 'ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git'
        PROJECT_BRANCH             = 'Dev-StarTrends'

        // Build
        PROJECT_NAME               = 'star.trends.dashboard'
        SOLUTION_FILE              = './StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln'
        VERSION_NUMBER             = "${VersionNumberPrefix}.${BUILD_NUMBER}"

        // NuGet feeds
        STAR_NUGET_REPO_SOURCE     = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        STAR_NUGET_REPO_NAME       = 'star-nuget'
        EXTERNAL_NUGET_REPO_SOURCE = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        EXTERNAL_NUGET_REPO_NAME   = 'external-nuget-local'
        NUGET_REMOTE_CACHE_SOURCE  = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'
        NUGET_REMOTE_CACHE_NAME    = 'nuget-remote-cache'
    }

    stages {
        stage('Setup Build Name') {
            steps {
                script { currentBuild.displayName = VERSION_NUMBER }
            }
        }

        stage('Checkout') {
            steps {
                script {
                    def git_details = checkout(
                        [$class: 'GitSCM',
                         branches: [[name: "*/${PROJECT_BRANCH}"]],
                         userRemoteConfigs: [[
                             url: PROJECT_REPOSITORY,
                             credentialsId: 'STAR-BITBUCKET-SSH'
                         ]]
                        ]
                    )
                    env.GIT_URL    = git_details.GIT_URL
                    env.GIT_BRANCH = git_details.GIT_BRANCH
                    env.GIT_COMMIT = git_details.GIT_COMMIT
                }
            }
        }

        stage('Artifactory Source Setup') {
            steps {
                script {
                    def feeds = [
                        [source: STAR_NUGET_REPO_SOURCE,     name: STAR_NUGET_REPO_NAME],
                        [source: EXTERNAL_NUGET_REPO_SOURCE, name: EXTERNAL_NUGET_REPO_NAME],
                        [source: NUGET_REMOTE_CACHE_SOURCE,  name: NUGET_REMOTE_CACHE_NAME]
                    ]
                    feeds.each { f ->
                        AddNugetSourcesFromArtifactory(f.source, f.name)
                    }
                    env.NUGET_SOURCE_ARGS = feeds.collect { "--source ${it.source}" }.join(' ')
                }
            }
        }

        stage('Show Workspace Contents') {
            steps {
                script {
                    def listing = bat(returnStdout:true, script:'dir /B /S')
                    echo "Workspace files:\n${listing}"
                }
            }
        }

        stage('Clean & Build') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    bat returnStdout:true, script: "dotnet clean --configuration ${params.Config}"
                    bat returnStdout:true, script: """
                        dotnet build \
                          --configuration ${params.Config} \
                          --property:Version=${VERSION_NUMBER} \
                          ${env.NUGET_SOURCE_ARGS}
                    """
                }
            }
        }

        stage('Publish Solution') {
            steps {
                script {
                    def projects = ['StarTrendsDashboard','StarTrendsDashboard.Shared']
                    projects.each { proj ->
                        dir('StarTrends/StarTrendsDashboard') {
                            bat returnStdout:true, script: """
                                dotnet publish ${proj} \
                                  --configuration ${params.Config} \
                                  --property:Version=${VERSION_NUMBER} \
                                  ${env.NUGET_SOURCE_ARGS} \
                                  --self-contained true \
                                  --use-current-runtime \
                                  --output .\\Bin\\${VERSION_NUMBER}\\${proj}
                            """
                        }
                    }
                }
            }
        }

        stage('Package ZIP') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    bat "if exist .\\Bin\\${VERSION_NUMBER} rmdir /S /Q .\\Bin\\${VERSION_NUMBER}\\temp"
                    zip archive: true,
                        dir: ".\\Bin\\${VERSION_NUMBER}",
                        zipFile: "${PROJECT_NAME}.${VERSION_NUMBER}.zip",
                        overwrite: true
                }
            }
        }

        stage('Promote to Artifactory') {
            steps {
                script {
                    def cfg = [
                        repository: 'star-generic-local-dev',
                        name:       PROJECT_NAME,
                        branch:     GIT_BRANCH,
                        version:    VERSION_NUMBER,
                        fullname:   "${PROJECT_NAME}.${VERSION_NUMBER}.zip",
                        path:       "${WORKSPACE}\\${PROJECT_NAME}.${VERSION_NUMBER}.zip"
                    ]
                    PromotePackageToArtifactory(cfg)
                }
            }
        }
    }
}

```
