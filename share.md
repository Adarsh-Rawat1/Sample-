```
@Library('jenkins-devops-cicd-library') _

// ——————————————————————————————————————————————————————
// Artifactory / NuGet helper functions (identical to Reporting-API)
// ——————————————————————————————————————————————————————
def RemoveNugetSourcesFromArtifactory(name) {
    try {
        withCredentials([usernamePassword(
            credentialsId: 'STAR-ARTIFACTORY',
            usernameVariable: 'USERNAME',
            passwordVariable: 'PASSWORD'
        )]) {
            bat returnStdout: true, script: "dotnet nuget remove source ${name}"
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def AddNugetSourcesFromArtifactory(source, name) {
    RemoveNugetSourcesFromArtifactory(name)
    try {
        withCredentials([usernamePassword(
            credentialsId: 'STAR-ARTIFACTORY',
            usernameVariable: 'USERNAME',
            passwordVariable: 'PASSWORD'
        )]) {
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
// Version & repo resolution (identical to Reporting-API)
// ——————————————————————————————————————————————————————
def ResolveVersion() {
    if (!fileExists('.\\build\\version.txt')) {
        def csproj = readFile('.\\StarTrends\\StarTrendsDashboard\\StarTrendsDashboard.csproj')
        def matcher = (csproj =~ /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/)
        if (!matcher) error "AssemblyVersion not found"
        return matcher[0][1].tokenize('.').take(3).join('.')
    } else {
        return readFile('.\\build\\version.txt').trim()
    }
}

def ResolveArtifactoryRepository() {
    return (env.BRANCH_NAME == 'master')
      ? 'star-generic-local-release'
      : 'star-generic-local-dev'
}

// ——————————————————————————————————————————————————————
// Pipeline
// ——————————————————————————————————————————————————————
pipeline {
    agent {
        node {
            label 'windows_2016_VS_2022'
            customWorkspace 'workspace/star/StarTrends'
        }
    }
    options { timestamps() }
    parameters {
        string(name: 'VersionNumberPrefix',
               defaultValue: '1.0.0',
               description: 'Prefix for your version (e.g. 1.0.0)')
        choice(name: 'Config',
               choices: ['Release','Debug'],
               description: 'Build configuration')
    }
    environment {
        // Dashboard specifics
        PROJECT_NAME       = 'star.trends.dashboard'
        SOLUTION_FILE      = 'StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln'
        CONFIG_FILE        = 'build\\Nuget.config'
        ALL_PROJECTS       = 'StarTrendsDashboard,StarTrendsDashboard.Shared'

        // NuGet feeds
        STAR_FEED_URL      = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        STAR_FEED_NAME     = 'star-nuget'
        EXT_FEED_URL       = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        EXT_FEED_NAME      = 'external-nuget-local'
        CACHE_FEED_URL     = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'
        CACHE_FEED_NAME    = 'nuget-remote-cache'

        // Artifactory
        ARTIFACTORY_CREDS_ID = 'STAR-ARTIFACTORY'
        ARTIFACTORY_URL      = 'https://artifactory.cib.echonet/artifactory'

        // MSBuild tool
        MSBuild17 = tool 'MSBuild_17.0'
        MSBUILD   = "${MSBuild17}"
    }

    stages {
        stage('Checkout') {
            steps {
                // clone your StarTrends repo
                git branch: 'Dev-StarTrends',
                    url: 'ssh://git@bitbucket.cib.echonet:7999/star/StarTrends.git',
                    credentialsId: 'STAR-BITBUCKET-SSH'
                script {
                    env.BRANCH_NAME = env.GIT_BRANCH
                }
            }
        }

        stage('Initialize Version') {
            steps {
                script {
                    def baseVer = ResolveVersion()
                    env.VERSION_NUMBER = "${baseVer}.${BUILD_NUMBER}"
                    currentBuild.displayName = env.VERSION_NUMBER
                    env.ZIP_NAME = "${PROJECT_NAME}-${VERSION_NUMBER}.zip"
                    env.ARTIFACTORY_REPO = ResolveArtifactoryRepository()
                }
            }
        }

        stage('NuGet Feed Setup') {
            steps {
                script {
                    AddNugetSourcesFromArtifactory(STAR_FEED_URL,  STAR_FEED_NAME)
                    AddNugetSourcesFromArtifactory(EXT_FEED_URL,   EXT_FEED_NAME)
                    AddNugetSourcesFromArtifactory(CACHE_FEED_URL, CACHE_FEED_NAME)
                    env.NUGET_ARGS = "--source ${STAR_FEED_URL} --source ${EXT_FEED_URL} --source ${CACHE_FEED_URL}"
                }
            }
        }

        stage('Clean & Build') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    bat "dotnet clean ${SOLUTION_FILE} -c ${params.Config}"
                    bat "dotnet build ${SOLUTION_FILE} -c ${params.Config} /p:Version=${VERSION_NUMBER} ${env.NUGET_ARGS}"
                }
            }
        }

        stage('Publish Projects') {
            steps {
                script {
                    def projs = ALL_PROJECTS.tokenize(',')
                    projs.each { proj ->
                        dir('StarTrends/StarTrendsDashboard') {
                            bat """
                              dotnet publish ${proj} \
                                -c ${params.Config} \
                                /p:Version=${VERSION_NUMBER} \
                                ${env.NUGET_ARGS} \
                                --self-contained true \
                                --use-current-runtime \
                                -o ..\\output\\${proj}
                            """
                        }
                    }
                }
            }
        }

        stage('Package ZIP') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    zip zipFile: "${ZIP_NAME}",
                        dir: "output",
                        archive: false
                }
            }
        }

        stage('Promote to Artifactory') {
            steps {
                script {
                    def cfg = [
                        repository: env.ARTIFACTORY_REPO,
                        name:       PROJECT_NAME,
                        branch:     BRANCH_NAME,
                        version:    VERSION_NUMBER,
                        fullname:   ZIP_NAME,
                        path:       "${env.WORKSPACE}\\StarTrends\\StarTrendsDashboard\\${ZIP_NAME}"
                    ]
                    PromotePackageToArtifactory(cfg)
                }
            }
        }
    }
}
```
