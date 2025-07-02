```
@Library('jenkins-devops-cicd-library') _

// --- Helper functions from shared library ---
def ListNugetSourcesFromArtifactory() {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: 'dotnet nuget list source')
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def RemoveNugetSourcesFromArtifactory(name) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: "dotnet nuget remove source ${name}")
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def UpdateNugetSourcesFromArtifactory(source, name) {
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            bat(returnStdout: true, script: "dotnet nuget update source ${name} --source ${source} --username ${USERNAME} --password ${PASSWORD} --store-password-in-clear-text")
        }
    } catch (err) {
        echo "Caught: ${err}"
    }
}

def AddNugetSourcesFromArtifactory(source, name) {
    RemoveNugetSourcesFromArtifactory(name)
    try {
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
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
        withCredentials(bindings: [usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            def buildInfo = Artifactory.newBuildInfo()
            buildInfo.env.capture = false
            buildInfo.name   = config.name
            buildInfo.number = config.version

            def server = Artifactory.newServer url: ARTIFACTORY_URL, credentialsId: ARTIFACTORY_CREDS_ID
            def uploadSpec = "{\"files\":[${GenerateArtifactoryUploadSpecification(config)}]}"

            timeout(time: 10, unit: 'MINUTES') {
                retry(3) {
                    server.upload       spec: uploadSpec, buildInfo: buildInfo
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
            label           'windows_2022_VS_2022'
            customWorkspace 'workspace/star/reportingapi'
        }
    }

    options { timestamps() }

    parameters {
        string(name: 'VersionNumberPrefix',
               defaultValue: '1.0.0',
               description: 'Prefix for the version number (will combine with BUILD_NUMBER)')
        choice(name: 'Config',
               choices: ['Release', 'Debug'],
               description: 'Build configuration')
    }

    environment {
        // Git & project
        PROJECT_REPOSITORY        = 'ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git'
        PROJECT_BRANCH            = 'Dev-StarTrends'
        PROJECT_NAME              = 'bnpp.star.reporting.api'
        SOLUTION_FILE             = 'bnpp.reporting.api.sln'
        CONFIG_FILE               = 'build/Nuget.config'

        // Versioning
        ARTIFACTORY_CREDS_ID      = 'STAR-ARTIFACTORY'
        ARTIFACTORY_URL           = 'https://artifactory.cib.echonet/artifactory'
        ARTIFACTORY_REPO          = ResolveArtifactoryRepository()
        VERSION_NUMBER            = "${VersionNumberPrefix}.${BUILD_NUMBER}"

        // NuGet feeds
        REPO_STAR_NUGET           = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        REPO_EXTERNAL_NUGET       = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        NUGET_REMOTE_CACHE        = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'

        // Fortify
        FORTIFY_URL               = 'https://fortifyssc.cib.echonet/ssc'
        SECURITY_CHAMPION_MAIL    = 'harpreet.nanra@uk.bnpparibas.com'
        APPLICATION_REPORTING_API = 'STAR.HUDSON'
        NEW_VERSION               = "0.0.${BUILD_NUMBER}"

        // Tools
        MSBUILD_17                = tool 'MSBuild_17.0'
        MSBUILD                   = "${MSBUILD_17}"
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    def gitDetails = checkout scmGit(
                        branches: [[name: "*/${PROJECT_BRANCH}"]],
                        userRemoteConfigs: [[
                            credentialsId: 'STAR-BITBUCKET-SSH',
                            url: PROJECT_REPOSITORY
                        ]]
                    )
                    env.GIT_URL    = gitDetails.GIT_URL
                    env.GIT_BRANCH = gitDetails.GIT_BRANCH
                    env.GIT_COMMIT = gitDetails.GIT_COMMIT
                }
            }
        }

        stage('Artifactory source setup') {
            steps {
                script {
                    AddNugetSourcesFromArtifactory(REPO_STAR_NUGET,    'star-nuget')
                    AddNugetSourcesFromArtifactory(REPO_EXTERNAL_NUGET,'external-nuget-local')
                    AddNugetSourcesFromArtifactory(NUGET_REMOTE_CACHE, 'nuget-remote-cache')
                }
            }
        }

        stage('Clean & Build') {
            steps {
                bat "dotnet clean --configuration ${params.Config}"
                bat "dotnet build ${SOLUTION_FILE} -c ${params.Config} --property:Version=${VERSION_NUMBER} --source ${REPO_STAR_NUGET} --source ${REPO_EXTERNAL_NUGET} --source ${NUGET_REMOTE_CACHE}"
            }
        }

        stage('Run unit tests') {
            steps {
                bat "dotnet test -c ${params.Config} --no-build --no-restore"
            }
        }

        stage('Publish EXEs') {
            steps {
                script {
                    env.ZIP_NAME = "${PROJECT_NAME}-${VERSION_NUMBER}"
                    ALL_PROJECTS.tokenize(',').each { proj ->
                        dir(proj) {
                            bat "dotnet publish --configuration ${params.Config} --self-contained true --use-current-runtime --output ../output/${proj}"
                        }
                    }
                }
            }
        }

        stage('Zip') {
            steps {
                bat "IF EXIST ${ZIP_NAME}.zip DEL /F ${ZIP_NAME}.zip"
                zip(zipFile: "${ZIP_NAME}.zip", dir: 'output/')
            }
        }

        stage('Promote to Artifactory') {
            steps {
                script {
                    def cfg = [
                        repository: ARTIFACTORY_REPO,
                        name:       PROJECT_NAME,
                        branch:     GIT_BRANCH,
                        version:    VERSION_NUMBER,
                        fullname:   "${ZIP_NAME}.zip",
                        path:       "${WORKSPACE}/${ZIP_NAME}.zip"
                    ]
                    PromotePackageToArtifactory(cfg)
                }
            }
        }

        stage('Fortify Scan') {
            steps {
                node('Bnpp-Maven3-SecOps') {
                    withCredentials([
                        string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN'),
                        string(credentialsId: 'fortify-report-token-star',   variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
                    ]) {
                        script {
                            bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -clean -b ${APPLICATION_REPORTING_API}"
                            bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -Xmx5G -b ${APPLICATION_REPORTING_API} -logfile fortify.log -verbose \"${MSBUILD}\" ${SOLUTION_FILE} /p:Configuration=${params.Config}"
                            bat "scancentral -sscurl ${FORTIFY_URL} -ssctoken ${FORTIFY_SCAN_CENTRAL_TOKEN} start -b ${APPLICATION_REPORTING_API} -f ${APPLICATION_REPORTING_API}.fpr -upload -application ${APPLICATION_REPORTING_API} -version ${NEW_VERSION} -uptoken ${FORTIFY_SCAN_CENTRAL_TOKEN}"
                        }
                    }
                }
            }
        }
    }
}
```
