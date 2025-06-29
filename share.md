```
@Library('jenkins-devops-cicd-library') _

// ─────────────────────────────────────────────────────────────────────────────
// 1) NuGet feed management (list/remove/update/add)
// ─────────────────────────────────────────────────────────────────────────────
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
    // always start fresh
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

// ─────────────────────────────────────────────────────────────────────────────
// 2) Artifactory JSON‐spec & promotion
// ─────────────────────────────────────────────────────────────────────────────
def GenerateArtifactoryUploadSpecification(config) {
    def spec = [
      pattern:  config.path,
      target:   "${config.repository}/com/bnpparibas/${config.name}/${config.branch}/${config.version}/${config.fullname}",
      props:    "build.commit=${env.GIT_COMMIT};build.version=${config.version};build.branch=${config.branch}"
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
            def server    = Artifactory.newServer(
                url: config.base_url,
                credentialsId: 'STAR-ARTIFACTORY'
            )
            def buildInfo = Artifactory.newBuildInfo()
            buildInfo.env.capture = false
            buildInfo.name   = config.name
            buildInfo.number = config.version

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

// ─────────────────────────────────────────────────────────────────────────────
// 3) Version & repo resolution (from your original pipeline)
// ─────────────────────────────────────────────────────────────────────────────
def ResolveArtifactoryRepository() {
    // push to "release" only on master
    return (env.GIT_BRANCH == 'master')
      ? 'star-generic-local-release'
      : 'star-generic-local-dev'
}
def ResolveVersion() {
    // if no build/version.txt, read from csproj AssemblyVersion
    if (!fileExists('.\\build\\version.txt')) {
        def csproj = readFile('.\\bnpp.star.openapi\\bnpp.star.openapi.csproj')
        def m = (csproj =~ /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/)
        if (m) {
            return m[0][1].tokenize('.').take(3).join('.')
        }
        error "AssemblyVersion not found in the .csproj file."
    } else {
        def v = readFile('.\\build\\version.txt').trim()
        echo "Reading Version from file version.txt → ${v}"
        return v
    }
}

// ─────────────────────────────────────────────────────────────────────────────
// 4) The main pipeline
// ─────────────────────────────────────────────────────────────────────────────
pipeline {
    agent {
        node {
            label           'windows_2016_VS_2022'
            customWorkspace 'workspace/star'
        }
    }
    options { timestamps() }

    parameters {
        choice(name: 'Config',
               choices: ['Release','Debug'],
               description: 'Build configuration')
    }

    environment {
        // repo & branch info
        PROJECT_REPOSITORY = 'ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git'
        PROJECT_NAME       = 'star.trends.dashboard'
        SOLUTION_FILE      = './StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln'

        // computed at runtime
        GIT_BRANCH       = ''
        GIT_COMMIT       = ''
        LATEST_TAG       = ''
        COMMIT_ID        = ''
        ARTIFACTORY_REPO = ResolveArtifactoryRepository()
        VERSION_NUMBER   = "${ResolveVersion()}.${env.BUILD_NUMBER}"

        // Artifactory & NuGet endpoints
        ARTIFACTORY_URL            = 'https://artifactory.cib.echonet/artifactory'
        STAR_NUGET_REPO_SOURCE     = 'https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        EXTERNAL_NUGET_REPO_SOURCE = 'https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        NUGET_REMOTE_CACHE_SOURCE  = 'https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'
    }

    stages {
      stage('Init') {
        steps { script {
          currentBuild.displayName = VERSION_NUMBER
        }}
      }

      stage('Checkout & Git Info') {
        steps { script {
          def info = checkout scmGit(
            branches: [[name: "*/master"]],
            userRemoteConfigs: [[
              credentialsId: 'STAR-BITBUCKET-SSH',
              url: PROJECT_REPOSITORY
            ]]
          )
          env.GIT_BRANCH = info.GIT_BRANCH
          env.GIT_COMMIT = info.GIT_COMMIT
          COMMIT_ID      = bat(returnStdout: true, script: '@git rev-parse --short HEAD').trim()
          LATEST_TAG     = bat(returnStdout: true, script: '@git describe --always --abbrev=0').trim()
          echo "Branch: ${env.GIT_BRANCH}, Commit: ${env.GIT_COMMIT}, Tag: ${LATEST_TAG}"
        }}
      }

      stage('Setup NuGet feeds') {
        steps { script {
          AddNugetSourcesFromArtifactory(STAR_NUGET_REPO_SOURCE,     'star-nuget')
          AddNugetSourcesFromArtifactory(EXTERNAL_NUGET_REPO_SOURCE, 'external-nuget-local')
          AddNugetSourcesFromArtifactory(NUGET_REMOTE_CACHE_SOURCE,  'nuget-remote-cache')
        }}
      }

      stage('Clean & Remove Bins') {
        steps { script {
          // if you had multiple project folders, you could tokenize them here
          bat 'dotnet clean --configuration %Config%'
          bat 'IF EXIST .\\output RMDIR /S /Q .\\output'
        }}
      }

      stage('Stamp CSProjs') {
        steps {
          powershell '''
            $files = Get-ChildItem -Recurse -Filter *.csproj
            foreach ($f in $files) {
              (Get-Content $f) -replace '<Version>.*?</Version>', "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>" |
                Set-Content $f
            }
          '''
        }
      }

      stage('Restore & Build') {
        steps {
          bat "\"${tool 'MSBuild_17.0'}\" -version"
          bat "dotnet restore ${SOLUTION_FILE} -r win-x64"
          bat "dotnet build ${SOLUTION_FILE} -c %Config%"
        }
      }

      // ─── Optional: run your tests ─────────────────────────────────────────────────
      /*stage('Test') {
        steps {
          bat "dotnet test --configuration %Config% --no-build --logger:\"trx;LogFileName=test_results.trx\""
        }
      }*/

      stage('Fortify: Create Version') {
        steps {
          node('Bnpp-Maven3-SecOps') {
            withCredentials([string(
              credentialsId: 'fortify-rest-api-token-star',
              variable: 'FORTIFY_REST_API_TOKEN'
            )]) {
              sh """
                echo APPLICATION=STAR.HUDSON
                echo NEW_VERSION=0.0.${env.BUILD_NUMBER}
                createNewFortifyApplicationVersionBasedOnLastScan \
                  https://fortifyssc.cib.echonet/ssc \
                  ${FORTIFY_REST_API_TOKEN} \
                  STAR.HUDSON \
                  0.0.${env.BUILD_NUMBER}
              """
            }
          }
        }
      }

      stage('Fortify: Scan/Analysis') {
        steps {
          withCredentials([
            string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN'),
            string(credentialsId: 'fortify-report-token-star',      variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
          ]) {
            bat 'fortifyupdate -url https://fortifyssc.cib.echonet/ssc -acceptSSLCertificate -acceptKey'
            bat "sourceanalyzer -clean -b STAR.HUDSON -Xmx5G -logfile fortify.log msbuild ${SOLUTION_FILE} /p:Configuration=%Config%"
            bat """
              scancentral \
                -sscurl https://fortifyssc.cib.echonet/ssc \
                -ssctoken ${FORTIFY_SCAN_CENTRAL_TOKEN} \
                start -projroot .fortify -b STAR.HUDSON \
                -email harpreet.nanra@uk.bnpparibas.com \
                -f STAR.HUDSON.fpr \
                -log STAR.HUDSON-scan.log \
                -upload \
                -application STAR.HUDSON \
                -version 0.0.${env.BUILD_NUMBER} \
                -uptoken ${FORTIFY_SCAN_CENTRAL_TOKEN}
            """
          }
        }
      }

      stage('Publish & Zip') {
        steps { script {
          // publish two StarTrends projects
          def pubs = ['StarTrendsDashboard','StarTrendsDashboard.Shared']
          pubs.each { p ->
            bat """
              dotnet publish ${p} \
                -c %Config% \
                -o .\\output\\${p} \
                --self-contained true \
                --use-current-runtime true
            """
          }
          // zip entire output folder
          zip zipFile: "${PROJECT_NAME}-${VERSION_NUMBER}.zip",
              dir: 'output',
              archive: true
        }}
      }

      stage('Promote to Artifactory') {
        steps { script {
          def cfg = [
            base_url:   env.ARTIFACTORY_URL,
            repository: ResolveArtifactoryRepository(),
            name:       PROJECT_NAME,
            branch:     env.GIT_BRANCH,
            version:    VERSION_NUMBER,
            fullname:   "${PROJECT_NAME}-${VERSION_NUMBER}.zip",
            path:       "${WORKSPACE}\\${PROJECT_NAME}-${VERSION_NUMBER}.zip"
          ]
          PromotePackageToArtifactory(cfg)
        }}
      }
    } // stages
}
```
