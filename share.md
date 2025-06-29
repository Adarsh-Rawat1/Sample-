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
        target:  "${config.repository}/com/bnpparibas/${config.name}/${config.branch}/${config.version}/${config.fullname}",
        props:   "build.commit=${env.GIT_COMMIT};build.version=${config.version};build.branch=${env.GIT_BRANCH}"
    ]
    return writeJSON(returnText: true, json: spec)
}
def PromotePackageToArtifactory(config) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: 'STAR-ARTIFACTORY', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            def buildInfo = Artifactory.newBuildInfo()
            buildInfo.env.capture = false
            buildInfo.name   = config.name
            buildInfo.number = config.version
            def server = Artifactory.newServer(url: 'https://artifactory.cib.echonet/artifactory', credentialsId: 'STAR-ARTIFACTORY')
            def uploadSpec = "{\"files\":[${GenerateArtifactoryUploadSpecification(config)}]}"
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
def ResolveArtifactoryRepository() {
    return (env.BRANCH_NAME == 'master') ? 'star-generic-local-release' : 'star-generic-local-dev'
}
def ResolveVersion() {
    if (!fileExists('.\\build\\version.txt')) {
        def csproj = readFile '.\\StarTrends\\StarTrendsDashboard\\StarTrendsDashboard.csproj'
        def m = csproj =~ /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/
        if (!m) error 'AssemblyVersion not found'
        return m[0][1].tokenize('.').take(3).join('.')
    } else {
        return readFile('.\\build\\version.txt').trim()
    }
}
pipeline {
    agent { node { label 'windows_2022_VS_2022'; customWorkspace 'workspace/star/StarTrends' } }
    options { timestamps() }
    parameters {
        string(name: 'VersionNumberPrefix', defaultValue: '1.0.0', description: '')
        choice(name: 'Config', choices: ['Release','Debug'], description: '')
    }
    environment {
        PROJECT_REPOSITORY         = 'ssh://git@bitbucket.cib.echonet:7999/star/StarTrends.git'
        PROJECT_BRANCH             = 'Dev-StarTrends'
        PROJECT_NAME               = 'star.trends.dashboard'
        SOLUTION_FILE              = 'StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln'
        CONFIG_FILE                = 'build\\Nuget.config'
        STAR_NUGET_REPO_SOURCE     = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        STAR_NUGET_REPO_NAME       = 'star-nuget'
        EXTERNAL_NUGET_REPO_SOURCE = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        EXTERNAL_NUGET_REPO_NAME   = 'external-nuget-local'
        NUGET_REMOTE_CACHE_SOURCE  = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'
        NUGET_REMOTE_CACHE_NAME    = 'nuget-remote-cache'
        MSBuild17                  = tool 'MSBuild_17.0'
        MSBUILD                    = "${MSBuild17}"
    }
    stages {
        stage('Setup build') {
            steps {
                script {
                    env.BRANCH_NAME       = PROJECT_BRANCH
                    env.VERSION_NUMBER    = "${ResolveVersion()}.${BUILD_NUMBER}"
                    currentBuild.displayName = VERSION_NUMBER
                    env.ZIP_NAME          = "${PROJECT_NAME}-${VERSION_NUMBER}.zip"
                    env.ARTIFACTORY_REPO  = ResolveArtifactoryRepository()
                }
            }
        }
        stage('Checkout') {
            steps {
                script {
                    def gd = checkout(scmGit(branches: [[name:"*/${PROJECT_BRANCH}"]],
                                              userRemoteConfigs:[[url:PROJECT_REPOSITORY,credentialsId:'STAR-BITBUCKET-SSH']]]))
                    env.GIT_BRANCH = gd.GIT_BRANCH
                    env.GIT_COMMIT = gd.GIT_COMMIT
                }
            }
        }
        stage('Artifactory source setup') {
            steps {
                script {
                    AddNugetSourcesFromArtifactory(STAR_NUGET_REPO_SOURCE, STAR_NUGET_REPO_NAME)
                    AddNugetSourcesFromArtifactory(EXTERNAL_NUGET_REPO_SOURCE, EXTERNAL_NUGET_REPO_NAME)
                    AddNugetSourcesFromArtifactory(NUGET_REMOTE_CACHE_SOURCE, NUGET_REMOTE_CACHE_NAME)
                }
            }
        }
        stage('Clean & Build') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    bat(returnStdout:true, script:"dotnet clean --configuration ${params.Config}")
                    bat(returnStdout:true, script:"dotnet build --configuration ${params.Config} --property:Version=${VERSION_NUMBER} --source ${STAR_NUGET_REPO_SOURCE} --source ${EXTERNAL_NUGET_REPO_SOURCE} --source ${NUGET_REMOTE_CACHE_SOURCE}")
                }
            }
        }
        stage('Publish projects') {
            steps {
                script {
                    ['StarTrendsDashboard','StarTrendsDashboard.Shared'].each { p ->
                        dir('StarTrends/StarTrendsDashboard') {
                            bat(returnStdout:true, script:"dotnet publish ${p} --configuration ${params.Config} --property:Version=${VERSION_NUMBER} --source ${STAR_NUGET_REPO_SOURCE} --source ${EXTERNAL_NUGET_REPO_SOURCE} --source ${NUGET_REMOTE_CACHE_SOURCE} --self-contained true --use-current-runtime true --output .\\Bin\\${VERSION_NUMBER}\\${p}")
                        }
                    }
                }
            }
        }
        stage('Package ZIP') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    zip archive:true, dir:".\\Bin\\${VERSION_NUMBER}", overwrite:true, zipFile:ZIP_NAME
                }
            }
        }
        stage('Promote package to Artifactory') {
            steps {
                script {
                    def cfg = [repository:ARTIFACTORY_REPO,name:PROJECT_NAME,branch:GIT_BRANCH,version:VERSION_NUMBER,fullname:ZIP_NAME,path:"${WORKSPACE}\\StarTrends\\StarTrendsDashboard\\${ZIP_NAME}"]
                    PromotePackageToArtifactory(cfg)
                }
            }
        }
    }
}
```
