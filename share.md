@Library('jenkins-devops-cicd-library') _

// --- Helper functions from shared library ---
def ListNugetSourcesFromArtifactory() {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat returnStdout: true, script: 'dotnet nuget list source'
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def RemoveNugetSourcesFromArtifactory(name) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat returnStdout: true, script: "dotnet nuget remove source ${name} --configfile ${CONFIG_FILE}"
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def UpdateNugetSourcesFromArtifactory(source, name) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat returnStdout: true, script: "dotnet nuget update source ${name} --source ${source} --username ${USERNAME} --password ${PASSWORD} --store-password-in-clear-text --configfile ${CONFIG_FILE}"
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def AddNugetSourcesFromArtifactory(source, name) {
    RemoveNugetSourcesFromArtifactory(name)
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat returnStdout: true, script: "dotnet nuget add source ${source} --name ${name} --username ${USERNAME} --password ${PASSWORD} --store-password-in-clear-text --configfile ${CONFIG_FILE}"
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def GenerateArtifactoryUploadSpecification(config) {
    def spec = [
        pattern: "${config.path}",
        target : "${config.repository}/com/bnpparibas/${config.name}/${config.branch}/${config.version}/${config.fullname}",
        props  : "build.commit=${GIT_COMMIT};build.version=${config.version};build.branch=${GIT_BRANCH}"
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
        def matcher        = (csprojContent =~ /<AssemblyVersion>([0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+)<\\/AssemblyVersion>/)
        if (matcher) {
            def assemblyVersion = matcher[0][1]
            return assemblyVersion.tokenize('.').take(3).join('.')
        } else {
            error 'AssemblyVersion not found in the .csproj file.'
        }
    } else {
        return readFile('.\\build\\version.txt').trim()
    }
}

pipeline {
    agent {
        node {
            label           'windows_2016_VS_2022'
            customWorkspace 'workspace/star'
        }
    }

    options {
        timestamps()
    }

    parameters {
        string(
            name: 'VersionNumberPrefix',
            defaultValue: '1.0.0',
            description: 'Prefix for version (combined with BUILD_NUMBER)'
        )
        choice(
            name: 'Config',
            choices: ['Release', 'Debug'],
            description: 'Build configuration'
        )
    }

    environment {
        // Project settings
        PROJECT_REPOSITORY         = 'ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git'
        PROJECT_BRANCH             = 'Dev-StarTrends'
        PROJECT_NAME               = 'star.trends.dashboard'

        // Solution file
        SOLUTION_FILE              = './StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln'
        CONFIG_FILE                = '.\\build\\Nuget.config'

        // Version
        ARTIFACTORY_CREDS_ID       = 'STAR-ARTIFACTORY'
        ARTIFACTORY_URL            = 'https://artifactory.cib.echonet/artifactory'
        ARTIFACTORY_REPO           = ResolveArtifactoryRepository()
        VERSION_NUMBER             = "${params.VersionNumberPrefix}.${BUILD_NUMBER}"

        // NuGet feeds
        STAR_NUGET_REPO_SOURCE     = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        STAR_NUGET_REPO_NAME       = 'star-nuget'
        EXTERNAL_NUGET_REPO_SOURCE = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        EXTERNAL_NUGET_REPO_NAME   = 'external-nuget-local'
        NUGET_REMOTE_CACHE_SOURCE  = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'
        NUGET_REMOTE_CACHE_NAME    = 'nuget-remote-cache'

        // Credentials
        BITBUCKET_CREDS_ID         = 'STAR-BITBUCKET-SSH'

        // Git info (filled in stages)
        GIT_COMMIT                 = ''
        GIT_BRANCH                 = ''
    }

    stages {
        stage('Clone Repository') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: PROJECT_BRANCH]],
                    userRemoteConfigs: [[
                        url: PROJECT_REPOSITORY,
                        credentialsId: BITBUCKET_CREDS_ID
                    ]]
                ])
            }
        }

        stage('Get Git Info') {
            steps {
                script {
                    env.GIT_COMMIT = bat(returnStdout: true, script: '@git rev-parse --short HEAD').trim()
                    env.GIT_BRANCH = env.BRANCH_NAME
                }
            }
        }

        stage('Clean & Restore') {
            steps {
                bat "dotnet clean --configuration ${params.Config}"
                bat "dotnet restore ${SOLUTION_FILE} --configfile ${CONFIG_FILE}"
            }
        }

        stage('Build & Test') {
            steps {
                bat "dotnet build ${SOLUTION_FILE} -c ${params.Config}"
                bat "dotnet test ${SOLUTION_FILE} -c ${params.Config} --no-build --no-restore"
            }
        }

        stage('Publish & Zip') {
            steps {
                script {
                    def zipName = "${PROJECT_NAME}.${VERSION_NUMBER}.zip"
                    bat "dotnet publish ${SOLUTION_FILE} -c ${params.Config} -o publish_output"
                    zip archive: false, dir: 'publish_output', zipFile: zipName
                    env.PACKAGE_ZIP = zipName
                }
            }
        }

        stage('Promote to Artifactory') {
            steps {
                script {
                    def cfg = [
                        repository: ARTIFACTORY_REPO,
                        name      : PROJECT_NAME,
                        branch    : env.GIT_BRANCH,
                        version   : VERSION_NUMBER,
                        fullname  : PACKAGE_ZIP,
                        path      : "${WORKSPACE}/${PACKAGE_ZIP}"
                    ]
                    PromotePackageToArtifactory(cfg)
                }
            }
        }
    }
}
