```
@Library('jenkins-devops-cicd-library') _

def ListNugetSourcesFromArtifactory() {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: 'STAR-ARTIFACTORY', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: 'dotnet nuget list source')
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def RemoveNugetSourcesFromArtifactory(name) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: 'STAR-ARTIFACTORY', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: "dotnet nuget remove source ${name}")
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def UpdateNugetSourcesFromArtifactory(source, name) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: 'STAR-ARTIFACTORY', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: "dotnet nuget update source ${name} --source ${source} --username ${USERNAME} --password ${PASSWORD} --store-password-in-clear-text")
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def AddNugetSourcesFromArtifactory(source, name) {
    RemoveNugetSourcesFromArtifactory(name)
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: 'STAR-ARTIFACTORY', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: "dotnet nuget add source ${source} --name ${name} --username ${USERNAME} --password ${PASSWORD} --store-password-in-clear-text")
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def GenerateArtifactoryUploadSpecification(config) {
    def spec = [
        pattern: "${config.path}",
        target: "${config.repository}/com/bnpparibas/${config.name}/${config.branch}/${config.version}/${config.fullname}",
        props: "build.commit=${GIT_COMMIT};build.version=${config.version};build.branch=${GIT_BRANCH}"
    ]
    return writeJSON(returnText: true, json: spec)
}

def PromotePackageToArtifactory(config) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: 'STAR-ARTIFACTORY', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            def buildInfo = Artifactory.newBuildInfo()
            buildInfo.env.capture = false
            buildInfo.name = config.name
            buildInfo.number = config.version
            def server = Artifactory.newServer(url: 'https://artifactory.cib.echonet/artifactory', credentialsId: 'STAR-ARTIFACTORY')
            def uploadSpec = """{"files":[${GenerateArtifactoryUploadSpecification(config)}]}"""
            timeout(time: 10, unit: 'MINUTES') {
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
    parameters {
        string(name: 'VersionNumberPrefix', defaultValue: '1.0.0', description: '')
        choice(name: 'Config', choices: ['Release', 'Debug'], description: '')
    }
    environment {
        PROJECT_REPOSITORY = 'ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git'
        PROJECT_BRANCH     = 'Dev-StarTrends'
        PROJECT_NAME       = 'star.trends.dashboard'
        SOLUTION_FILE      = './StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln'
        VERSION_NUMBER     = "${VersionNumberPrefix}.${BUILD_NUMBER}"
        STAR_NUGET_REPO_SOURCE      = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        STAR_NUGET_REPO_NAME        = 'star-nuget'
        EXTERNAL_NUGET_REPO_SOURCE  = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        EXTERNAL_NUGET_REPO_NAME    = 'external-nuget-local'
        NUGET_REMOTE_CACHE_SOURCE   = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'
        NUGET_REMOTE_CACHE_NAME     = 'nuget-remote-cache'
    }
    stages {
        stage('Setup Build Name') {
            steps {
                script { currentBuild.displayName = "${VERSION_NUMBER}" }
            }
        }
        stage('Checkout') {
            steps {
                script {
                    def gitDetails = checkout scmGit(
                        branches: [[name: "*/${PROJECT_BRANCH}"]],
                        userRemoteConfigs: [[credentialsId: 'STAR-BITBUCKET-SSH', url: "${PROJECT_REPOSITORY}"]]
                    )
                    env.GIT_URL    = gitDetails.GIT_URL
                    env.GIT_BRANCH = gitDetails.GIT_BRANCH
                    env.GIT_COMMIT = gitDetails.GIT_COMMIT
                }
            }
        }
        stage('Artifactory Source Setup') {
            steps {
                script {
                    AddNugetSourcesFromArtifactory(STAR_NUGET_REPO_SOURCE, STAR_NUGET_REPO_NAME)
                    AddNugetSourcesFromArtifactory(EXTERNAL_NUGET_REPO_SOURCE, EXTERNAL_NUGET_REPO_NAME)
                    AddNugetSourcesFromArtifactory(NUGET_REMOTE_CACHE_SOURCE, NUGET_REMOTE_CACHE_NAME)
                }
            }
        }
        stage('Clean Previous Results') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    bat returnStdout: true, script: "dotnet clean --configuration ${params.Config}"
                }
            }
        }
        stage('Build Solution') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    bat returnStdout: true, script: "dotnet build --configuration ${params.Config} --property:Version=${VERSION_NUMBER} --source ${STAR_NUGET_REPO_SOURCE} --source ${EXTERNAL_NUGET_REPO_SOURCE} --source ${NUGET_REMOTE_CACHE_SOURCE}"
                }
            }
        }
        stage('Publish Packages') {
            steps {
                script {
                    def projects = ['StarTrendsDashboard', 'StarTrendsDashboard.Shared']
                    projects.each {
                        dir('StarTrends/StarTrendsDashboard') {
                            bat returnStdout: true, script: "dotnet publish ${it} --configuration ${params.Config} --property:Version=${VERSION_NUMBER} --source ${STAR_NUGET_REPO_SOURCE} --source ${EXTERNAL_NUGET_REPO_SOURCE} --source ${NUGET_REMOTE_CACHE_SOURCE} --self-contained true --use-current-runtime true --output .\\Bin\\${VERSION_NUMBER}\\${it}"
                        }
                    }
                }
            }
        }
        stage('Package Zip') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    zip archive: true, dir: ".\\Bin\\${VERSION_NUMBER}", zipFile: "${PROJECT_NAME}.${VERSION_NUMBER}.zip"
                }
            }
        }
        stage('Promote to Artifactory') {
            steps {
                script {
                    def config = [
                        path: "${WORKSPACE}\\${PROJECT_NAME}.${VERSION_NUMBER}.zip",
                        repository: ResolveArtifactoryRepository(),
                        name: PROJECT_NAME,
                        branch: GIT_BRANCH,
                        version: VERSION_NUMBER,
                        fullname: "${PROJECT_NAME}.${VERSION_NUMBER}.zip"
                    ]
                    PromotePackageToArtifactory(config)
                }
            }
        }
    }
}
```
