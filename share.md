```
@Library('jenkins-devops-cicd-library') _

// ——————————————————————————————————————————————————————
// Artifactory / NuGet helper functions
// ——————————————————————————————————————————————————————
def ListNugetSourcesFromArtifactory() {
    try {
        withCredentials([usernamePassword(
            credentialsId: 'STAR-ARTIFACTORY',
            usernameVariable: 'USERNAME',
            passwordVariable: 'PASSWORD'
        )]) {
            bat returnStdout: true, script: 'dotnet nuget list source'
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

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

def UpdateNugetSourcesFromArtifactory(source, name) {
    try {
        withCredentials([usernamePassword(
            credentialsId: 'STAR-ARTIFACTORY',
            usernameVariable: 'USERNAME',
            passwordVariable: 'PASSWORD'
        )]) {
            bat returnStdout: true, script: """
                dotnet nuget update source ${name} \
                    --source ${source} \
                    --username ${USERNAME} \
                    --password ${PASSWORD} \
                    --store-password-in-clear-text
            """
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def AddNugetSourcesFromArtifactory(source, name) {
    // remove any stale entry first
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
// Artifactory promotion helpers
// ——————————————————————————————————————————————————————
def GenerateArtifactoryUploadSpecification(config) {
    def spec = [
        pattern:  config["path"],
        target:   "${config["repository"]}/com/bnpparibas/${config["name"]}/${config["branch"]}/${config["version"]}/${config["fullname"]}",
        props:    "build.commit=${env.GIT_COMMIT};build.version=${config["version"]};build.branch=${env.GIT_BRANCH}"
    ]
    return writeJSON(returnText: true, json: spec)
}

def PromotePackageToArtifactory(config) {
    try {
        withCredentials([usernamePassword(
            credentialsId: 'STAR-ARTIFACTORY',
            usernameVariable: 'USERNAME',
            passwordVariable: 'PASSWORD'
        )]) {
            def buildInfo = Artifactory.newBuildInfo()
            buildInfo.env.capture = false
            buildInfo.name   = config["name"]
            buildInfo.number = config["version"]

            def server = Artifactory.newServer(
                url: 'https://artifactory.cib.echonet/artifactory',
                credentialsId: 'STAR-ARTIFACTORY'
            )
            def uploadSpec = """{
                "files":[${GenerateArtifactoryUploadSpecification(config)}]
            }"""

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

// ——————————————————————————————————————————————————————
// Pipeline for Star Trends Dashboard Reporting API
// ——————————————————————————————————————————————————————
pipeline {
    agent {
        node {
            label 'windows_2022_VS_2022'
            customWorkspace 'workspace/star/reportingapi'
        }
    }

    options {
        timestamps()
    }

    parameters {
        string(name: 'VersionNumberPrefix',
               defaultValue: '1.0.0',
               description: 'e.g. 1.0.0')
        choice(name: 'Config',
               choices: ['Release', 'Debug'],
               description: 'Build configuration')
    }

    environment {
        // project
        PROJECT_NAME       = "bnpp.star.reporting.api"
        ALL_PROJECTS       = "bnpp.star.data.extractor,bnpp.star.web.service"
        SOLUTION_FILE      = "bnpp.reporting.api.sln"
        CONFIG_FILE        = "build\\Nuget.config"

        // Artifactory
        ARTIFACTORY_CREDS_ID   = 'STAR-ARTIFACTORY'
        ARTIFACTORY_URL        = "https://artifactory.cib.echonet/artifactory"
        // these three feeds:
        STAR_NUGET_REPO_SOURCE     = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        STAR_NUGET_REPO_NAME       = 'star-nuget'
        EXTERNAL_NUGET_REPO_SOURCE = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        EXTERNAL_NUGET_REPO_NAME   = 'external-nuget-local'
        NUGET_REMOTE_CACHE_SOURCE  = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'
        NUGET_REMOTE_CACHE_NAME    = 'nuget-remote-cache'

        // Fortify
        FORTIFY_URL             = "https://fortifyssc.cib.echonet/ssc"
        SECURITY_CHAMPION_MAIL  = "harpreet.nanra@uk.bnpparibas.com"
        APPLICATION_REPORTING_API = "STAR.HUDSON"

        // MSBuild
        MSBuild17 = tool 'MSBuild_17.0'
        MSBUILD   = "${MSBuild17}"
    }

    stages {

        stage('Create needed folders') {
            steps {
                bat 'if not exist "D:\\data\\reportingapi" mkdir "D:\\data\\reportingapi"'
            }
        }

        stage('Get commitId and latest tag') {
            steps {
                script {
                    env.COMMIT_ID = bat(returnStdout: true,
                                       script: '@git rev-parse --short HEAD').trim()
                    env.LATEST_TAG = bat(returnStdout: true,
                                        script: '@git describe --always --abbrev=0').trim()
                    echo "Application Version: ${VERSION_NUMBER}"
                }
            }
        }

        stage('Get artifact version') {
            steps {
                script {
                    // compute version
                    def base = ResolveVersion()
                    env.VERSION_NUMBER = "${base}.${BUILD_NUMBER}"
                    currentBuild.displayName = env.VERSION_NUMBER
                    env.ZIP_NAME = "${PROJECT_NAME}-${VERSION_NUMBER}"
                }
            }
        }

        stage('Delete build directories') {
            steps {
                script {
                    ALL_PROJECTS.tokenize(',').each { proj ->
                        bat "if exist ${proj}\\bin rmdir /S /Q ${proj}\\bin"
                    }
                }
            }
        }

        stage('Set dll/exe Version') {
            steps {
                powershell '''
                Get-ChildItem . -Recurse -Filter *.csproj | ForEach-Object {
                  (Get-Content $_.FullName) -replace '<Version>1.1.1.1</Version>',
                    "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>" |
                    Set-Content $_.FullName
                }
                '''
            }
        }

        stage('Artifactory source setup') {
            steps {
                script {
                    AddNugetSourcesFromArtifactory(STAR_NUGET_REPO_SOURCE,     STAR_NUGET_REPO_NAME)
                    AddNugetSourcesFromArtifactory(EXTERNAL_NUGET_REPO_SOURCE, EXTERNAL_NUGET_REPO_NAME)
                    AddNugetSourcesFromArtifactory(NUGET_REMOTE_CACHE_SOURCE,  NUGET_REMOTE_CACHE_NAME)
                }
            }
        }

        stage('Show content') {
            steps {
                script {
                    echo bat(returnStdout:true, script:'dir /B /S')
                }
            }
        }

        stage('Clean previous build results') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    bat returnStdout: true,
                        script: "dotnet clean --configuration ${params.Config}"
                }
            }
        }

        stage('Build projects in the solution') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    bat returnStdout: true, script: """
                      dotnet build --configuration ${params.Config} \
                        --property:Version=${VERSION_NUMBER} \
                        --source ${STAR_NUGET_REPO_SOURCE} \
                        --source ${EXTERNAL_NUGET_REPO_SOURCE} \
                        --source ${NUGET_REMOTE_CACHE_SOURCE}
                    """
                }
            }
        }

        stage('Publish package solution') {
            steps {
                script {
                    def projects = ['StarTrendsDashboard','StarTrendsDashboard.Shared']
                    projects.each { p ->
                        dir('StarTrends/StarTrendsDashboard') {
                            bat returnStdout: true, script: """
                              dotnet publish ${p} \
                                --configuration ${params.Config} \
                                --property:Version=${VERSION_NUMBER} \
                                --source ${STAR_NUGET_REPO_SOURCE} \
                                --source ${EXTERNAL_NUGET_REPO_SOURCE} \
                                --source ${NUGET_REMOTE_CACHE_SOURCE} \
                                --self-contained true \
                                --use-current-runtime true \
                                --output .\\Bin\\${VERSION_NUMBER}\\${p}
                            """
                        }
                    }
                }
            }
        }

        stage('Build deployable package') {
            steps {
                dir('StarTrends/StarTrendsDashboard') {
                    zip archive: true,
                        dir: ".\\Bin\\${VERSION_NUMBER}",
                        overwrite: true,
                        zipFile: "${PROJECT_NAME}.${VERSION_NUMBER}.zip"
                }
            }
        }

        stage('Promote package to Artifactory') {
            steps {
                script {
                    def cfg = [
                        repository: 'star-generic-local-dev',
                        name:       PROJECT_NAME,
                        branch:     env.GIT_BRANCH,
                        version:    VERSION_NUMBER,
                        fullname:   "${PROJECT_NAME}.${VERSION_NUMBER}.zip",
                        path:       "${env.WORKSPACE}\\${PROJECT_NAME}.${VERSION_NUMBER}.zip"
                    ]
                    PromotePackageToArtifactory(cfg)
                }
            }
        }
    }
}
```
