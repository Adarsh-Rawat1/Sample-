```
@Library('jenkins-devops-cicd-library') _

// pick correct Artifactory repo based on branch
def ResolveArtifactoryRepository() {
    return (env.BRANCH_NAME == 'master')
      ? 'star-generic-local-release'
      : 'star-generic-local-dev'
}

// read version from file or from .csproj AssemblyVersion
def ResolveVersion() {
    if (!fileExists('.\\build\\version.txt')) {
        def csproj = readFile('.\\bnpp.star.openapi\\bnpp.star.openapi.csproj')
        def matcher = (csproj =~ /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/)
        if (!matcher) error "AssemblyVersion not found"
        return matcher[0][1].tokenize('.').take(3).join('.')
    } else {
        return readFile('.\\build\\version.txt').trim()
    }
}

pipeline {
  agent {
    node {
      label 'windows_2022_VS_2022'
      customWorkspace 'workspace/star/reportingapi'
    }
  }
  options { timestamps() }
  parameters {
    string(name: 'PROJECT_REPOSITORY',
           defaultValue: 'ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.reporting.api.git',
           description: 'Git URL of your reporting-api repo')
    string(name: 'PROJECT_BRANCH',
           defaultValue: 'Dev-StarTrends',
           description: 'Branch to build')
  }
  environment {
    PROJECT_NAME              = "bnpp.star.reporting.api"
    ALL_PROJECTS              = "bnpp.star.data.extractor,bnpp.star.web.service"
    SOLUTION_FILE             = "bnpp.reporting.api.sln"
    CONFIG_FILE               = "build\\Nuget.config"

    ARTIFACTORY_CREDS_ID      = 'STAR-ARTIFACTORY'
    ARTIFACTORY_URL           = "https://artifactory.cib.echonet/artifactory"

    REPO_STAR_NUGET           = "https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget"
    REPO_EXTERNAL_NUGET       = "https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local"
    REPO_REMOTE_CACHE         = "https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache"

    FORTIFY_URL               = "https://fortifyssc.cib.echonet/ssc"
    SECURITY_CHAMPION_MAIL    = "harpreet.nanra@uk.bnpparibas.com"
    APPLICATION_REPORTING_API = "STAR.HUDSON"

    MSBuild17                 = tool 'MSBuild_17.0'
    MSBUILD                   = "${MSBuild17}"
  }

  stages {
    stage('Checkout') {
      steps {
        script {
          // explicit clone — no checkout scm
          git url: params.PROJECT_REPOSITORY,
              branch: params.PROJECT_BRANCH,
              credentialsId: 'STAR-BITBUCKET-SSH'
          // now stash branch into env for later logic
          env.BRANCH_NAME = params.PROJECT_BRANCH
        }
      }
    }

    stage('Initialize') {
      steps {
        script {
          env.ARTIFACTORY_REPO = ResolveArtifactoryRepository()
          def baseVer = ResolveVersion()
          env.VERSION_NUMBER = "${baseVer}.${BUILD_NUMBER}"
          currentBuild.displayName = env.VERSION_NUMBER
          env.ZIP_NAME       = "${PROJECT_NAME}-${VERSION_NUMBER}"
        }
      }
    }

    stage('Create folders') {
      steps {
        bat 'if not exist "D:\\data\\reportingapi" mkdir D:\\data\\reportingapi'
      }
    }

    stage('Clean previous bins') {
      steps {
        script {
          ALL_PROJECTS.tokenize(',').each { proj ->
            bat "if exist ${proj}\\bin rmdir /S /Q ${proj}\\bin"
          }
        }
      }
    }

    stage('Set csproj Version') {
      steps {
        powershell '''
          Get-ChildItem . -Recurse -Filter *.csproj | ForEach-Object {
            (Get-Content $_.FullName) `
              -replace '<Version>1.1.1.1</Version>', "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>" `
              | Set-Content $_.FullName
          }
        '''
      }
    }

    stage('Add NuGet feeds') {
      steps {
        script {
          // fresh skeleton
          writeFile file: CONFIG_FILE, text: '''<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources><clear/></packageSources>
  <packageSourceCredentials/>
</configuration>'''

          // bind Artifactory creds just for this step
          withCredentials([usernamePassword(credentialsId: ARTIFACTORY_CREDS_ID,
                                            usernameVariable: 'ART_USER',
                                            passwordVariable: 'ART_PSW')]) {
            def feeds = [
              [name:'star-nuget',           url:REPO_STAR_NUGET],
              [name:'external-nuget-local', url:REPO_EXTERNAL_NUGET],
              [name:'nuget-remote-cache',    url:REPO_REMOTE_CACHE]
            ]
            feeds.each { f ->
              bat """
                nuget sources Remove -Name ${f.name} -ConfigFile ${CONFIG_FILE} || echo skip
                nuget sources Add -Name ${f.name} ^
                  -Source ${f.url} ^
                  -UserName %ART_USER% ^
                  -Password %ART_PSW% ^
                  -ConfigFile ${CONFIG_FILE}
              """
            }
          }
        }
      }
    }

    stage('Clean & Restore') {
      steps {
        bat "dotnet clean --configuration Release"
        bat "\"${MSBUILD}\" -version"
        bat "dotnet restore ${SOLUTION_FILE} -r win-x64 --configfile ${CONFIG_FILE}"
      }
    }

    stage('Strip overrides') {
      steps {
        powershell '''
          Get-ChildItem . -Recurse -Filter *.json | ForEach-Object {
            (Get-Content $_.FullName) -replace 'FileOverrideLocation','FileOverrideLocation_' | Set-Content $_.FullName
          }
        '''
      }
    }

    stage('Build & Test') {
      steps {
        bat "dotnet build ${SOLUTION_FILE} -c Release"
        bat "dotnet test  ${SOLUTION_FILE} -c Release --no-build --no-restore"
      }
    }

    stage('Fortify Version') {
      steps {
        node('Bnpp-Maven3-SecOps') {
          withCredentials([string(credentialsId: 'fortify-rest-api-token-star',
                                 variable: 'FORTIFY_REST_API_TOKEN')]) {
            script {
              createNewFortifyApplicationVersionBasedOnLastScan(
                FORTIFY_URL, FORTIFY_REST_API_TOKEN,
                APPLICATION_REPORTING_API, "0.0.${BUILD_NUMBER}"
              )
            }
          }
        }
      }
    }

    stage('Fortify Scan') {
      steps {
        withCredentials([
          string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_REST_API_TOKEN'),
          string(credentialsId: 'fortify-report-token-star',     variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
        ]) {
          script {
            try {
              bat "fortifyupdate -url ${FORTIFY_URL} -acceptSSLCertificate -acceptKey"
              bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -clean -b ${APPLICATION_REPORTING_API}"
              bat """sourceanalyzer \
                    -Dcom.fortify.sca.ProjectRoot=.fortify \
                    -Xmx5G \
                    -b ${APPLICATION_REPORTING_API} \
                    -logfile reporting-api-fortify.log \
                    -debug -verbose \
                    "${MSBUILD}" ${SOLUTION_FILE} /p:Configuration=Release"""
              bat """scancentral \
                    -sscurl ${FORTIFY_URL} \
                    -ssctoken ${FORTIFY_SCAN_CENTRAL_TOKEN} \
                    start -projroot .fortify \
                    -b ${APPLICATION_REPORTING_API} \
                    -email ${SECURITY_CHAMPION_MAIL} \
                    -f ${APPLICATION_REPORTING_API}.fpr \
                    -log ${APPLICATION_REPORTING_API}-scan.log \
                    -upload -application ${APPLICATION_REPORTING_API} \
                    -version 0.0.${BUILD_NUMBER} \
                    -uptoken ${FORTIFY_SCAN_CENTRAL_TOKEN} \
                    -scan"""
            } catch (e) {
              bat "type reporting-api-fortify.log"
              error "Fortify failed: ${e}"
            }
          }
        }
      }
    }

    stage('Publish EXEs') {
      steps {
        script {
          bat "if exist output rmdir /S /Q output"
          ALL_PROJECTS.tokenize(',').each { proj ->
            dir(proj) {
              bat "dotnet publish --no-restore -c Release --self-contained true --use-current-runtime true -o ../output/${proj}"
            }
          }
        }
      }
    }

    stage('Zip & Deploy') {
      steps {
        bat "if exist ${ZIP_NAME}.zip del /F ${ZIP_NAME}.zip"
        zip zipFile: "${ZIP_NAME}.zip", archive: false, dir: 'output'
        script {
          def buildInfo = Artifactory.newBuildInfo()
          buildInfo.env.capture = false
          buildInfo.name   = ZIP_NAME
          buildInfo.number = VERSION_NUMBER

          def server = Artifactory.newServer url: ARTIFACTORY_URL, credentialsId: ARTIFACTORY_CREDS_ID
          def spec = """{
            "files":[{
              "pattern":"${ZIP_NAME}.zip",
              "target":"${ARTIFACTORY_REPO}/com/bnpparibas/${PROJECT_NAME}/${BRANCH_NAME}/${VERSION_NUMBER}/${ZIP_NAME}.zip",
              "props":"build.commit=${COMMIT_ID};build.version=${VERSION_NUMBER};build.branch=${BRANCH_NAME}"
            }]
          }"""
          server.upload spec: spec, buildInfo: buildInfo
          server.publishBuildInfo buildInfo
        }
      }
    }
  }
}


```
