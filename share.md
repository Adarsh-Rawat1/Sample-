```
@Library('jenkins-devops-cicd-library@master') _

// helpers
def ResolveArtifactoryRepository() {
    def repos = 'star-generic-local-dev'
    if (env.BRANCH_NAME == 'master') {
        repos = 'star-generic-local-release'
    }
    return repos
}

def ResolveVersion() {
    if (!fileExists('build/version.txt')) {
        def csproj = readFile('StarTrends/StarTrendsDashboard/StarTrendsDashboard.csproj')
        def matcher = (csproj =~ /<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/)
        if (matcher) {
            def ver = matcher[0][1].tokenize('.').take(3).join('.')
            echo "Version from AssemblyVersion: ${ver}"
            return ver
        }
        error "AssemblyVersion not found"
    }
    def txt = readFile('build/version.txt').trim()
    echo "Version from file: ${txt}"
    return txt
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
        timeout(time: 60, unit: 'MINUTES')
        skipDefaultCheckout()
    }

    environment {
        // static config
        PROJECT_NAME        = "star.trends.dashboard"
        PROJECT_REPO        = "ssh://git@bitbucket.cib.echonet:7999/star/bnpp.star.utilities.git"
        ALL_PROJECTS        = "StarTrendsDashboard,StarTrendsDashboard.Shared"
        SOLUTION_FILE       = "StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln"
        CONFIG_FILE         = "build/Nuget.config"
        REPO_STAR_NUGET     = "https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget"
        REPO_EXTERNAL_NUGET = "https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local"
        REPO_NUGET_CACHE    = "https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache"
        ARTIFACTORY_CREDS_ID= 'STAR-ARTIFACTORY'
        ARTIFACTORY_URL     = "https://artifactory.cib.echonet/artifactory"
        FORTIFY_URL         = "https://fortifyssc.cib.echonet/ssc"
        SECURITY_CHAMPION_MAIL = "security.champion@uk.bnpparibas.com"
        APPLICATION_NAME    = "STAR.TRENDS.DASHBOARD"
        MSBUILD             = tool('MSBuild_17.0')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.BRANCH_NAME}"]],
                    userRemoteConfigs: [[
                        url: env.PROJECT_REPO,
                        credentialsId: 'STAR-BITBUCKET-SSH'
                    ]],
                    extensions: [
                        [$class: 'CleanBeforeCheckout'],
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: '.']
                    ]
                ])
                script {
                    env.GIT_COMMIT = bat(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.GIT_TAG    = bat(returnStdout: true, script: 'git describe --tags --always').trim()
                    echo "Commit: ${env.GIT_COMMIT}, Tag: ${env.GIT_TAG}"
                }
            }
        }

        stage('Initialize') {
            steps {
                script {
                    // compute dynamic values now that workspace is ready
                    env.ARTIFACTORY_REPO = ResolveArtifactoryRepository()
                    def baseVer = ResolveVersion()
                    env.VERSION_NUMBER   = "${baseVer}.${env.BUILD_NUMBER}"
                    env.ZIP_NAME         = "${env.PROJECT_NAME}-${env.VERSION_NUMBER}"
                    currentBuild.displayName = env.VERSION_NUMBER
                    // prepare output dir
                    bat '''
                        if exist output rmdir /s /q output
                        mkdir output
                    '''
                }
            }
        }

        stage('Clean & Restore') {
            steps {
                script {
                    // clean bins
                    env.ALL_PROJECTS.split(',').each { proj ->
                        bat "if exist ${proj}\\bin rmdir /s /q ${proj}\\bin"
                    }
                    // set version
                    powershell '''
                        Get-ChildItem StarTrends/StarTrendsDashboard -Filter '*.csproj' |
                          ForEach-Object {
                            (Get-Content $_) -replace '<Version>1.1.1.1</Version>',
                              "<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>" |
                            Set-Content $_
                          }
                    '''
                    // add nuget sources
                    withCredentials([usernamePassword(credentialsId: env.ARTIFACTORY_CREDS_ID,
                                                      usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        bat """
                            nuget sources remove -Name star-nuget -ConfigFile ${env.CONFIG_FILE} || echo ok
                            nuget sources add -Name star-nuget -Source ${env.REPO_STAR_NUGET} -UserName %USER% -Password %PASS% -ConfigFile ${env.CONFIG_FILE}
                            nuget sources remove -Name external-nuget -ConfigFile ${env.CONFIG_FILE} || echo ok
                            nuget sources add -Name external-nuget -Source ${env.REPO_EXTERNAL_NUGET} -UserName %USER% -Password %PASS% -ConfigFile ${env.CONFIG_FILE}
                            nuget sources remove -Name nuget-cache -ConfigFile ${env.CONFIG_FILE} || echo ok
                            nuget sources add -Name nuget-cache -Source ${env.REPO_NUGET_CACHE} -UserName %USER% -Password %PASS% -ConfigFile ${env.CONFIG_FILE}
                        """
                    }
                    // restore
                    bat "dotnet restore ${env.SOLUTION_FILE} -r win-x64 --configfile ${env.CONFIG_FILE}"
                }
            }
        }

        stage('Build') {
            steps {
                bat "\"${env.MSBUILD}\" ${env.SOLUTION_FILE} /t:Build /p:Configuration=Release /p:Version=${env.VERSION_NUMBER} /p:RuntimeIdentifier=win-x64"
            }
        }

        stage('Test') {
            steps {
                bat "dotnet test ${env.SOLUTION_FILE} -c Release --no-build --no-restore"
            }
        }

        stage('Fortify Scan') {
            when { expression { env.BRANCH_NAME in ['master','release'] } }
            steps {
                withCredentials([
                    string(credentialsId: 'fortify-rest-api-token-star', variable: 'FORTIFY_TOKEN'),
                    string(credentialsId: 'fortify-report-token-star', variable: 'FORTIFY_SCAN_CENTRAL_TOKEN')
                ]) {
                    bat """
                        fortifyupdate -url ${env.FORTIFY_URL} -acceptSSLCertificate -acceptKey
                        sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -clean -b ${env.APPLICATION_NAME}
                        sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -Xmx5G -b ${env.APPLICATION_NAME} -logfile fortify.log -verbose "${env.MSBUILD}" ${env.SOLUTION_FILE} /p:Configuration=Release
                        scancentral -sscurl ${env.FORTIFY_URL} -ssctoken ${env.FORTIFY_SCAN_CENTRAL_TOKEN} start -projroot .fortify -b ${env.APPLICATION_NAME} -email ${env.SECURITY_CHAMPION_MAIL} -f ${env.APPLICATION_NAME}.fpr -upload -application ${env.APPLICATION_NAME} -version 0.0.${env.BUILD_NUMBER} -uptoken ${env.FORTIFY_SCAN_CENTRAL_TOKEN}
                    """
                }
            }
        }

        stage('Publish') {
            steps {
                script {
                    env.ALL_PROJECTS.split(',').each { proj ->
                        dir("StarTrends/StarTrendsDashboard/${proj}") {
                            bat "dotnet publish -c Release --self-contained true --use-current-runtime true -o ../../../output/${proj}"
                        }
                    }
                }
            }
        }

        stage('Package') {
            steps {
                bat "if exist \"${env.ZIP_NAME}.zip\" del /F \"${env.ZIP_NAME}.zip\""
                zip zipFile: "${env.ZIP_NAME}.zip", dir: 'output'
                archiveArtifacts artifacts: "${env.ZIP_NAME}.zip", fingerprint: true
            }
        }

        stage('Deploy to Artifactory') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.ARTIFACTORY_CREDS_ID,
                                                      usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        def server = Artifactory.newServer(
                            url: env.ARTIFACTORY_URL,
                            credentialsId: env.ARTIFACTORY_CREDS_ID
                        )
                        def buildInfo = Artifactory.newBuildInfo()
                        buildInfo.env.capture = false
                        buildInfo.name   = env.PROJECT_NAME
                        buildInfo.number = env.VERSION_NUMBER

                        def spec = """{
                          "files":[
                            {
                              "pattern":"${env.ZIP_NAME}.zip",
                              "target":"${env.ARTIFACTORY_REPO}/com/bnpparibas/${env.PROJECT_NAME}/${env.BRANCH_NAME}/${env.VERSION_NUMBER}/${env.ZIP_NAME}.zip",
                              "props":"build.commit=${env.GIT_COMMIT};build.version=${env.VERSION_NUMBER};build.branch=${env.BRANCH_NAME}"
                            }
                          ]
                        }"""
                        server.upload spec: spec, buildInfo: buildInfo
                        server.publishBuildInfo buildInfo
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Build ${env.VERSION_NUMBER} succeeded"
        }
        failure {
            echo "Build ${env.VERSION_NUMBER} failed"
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "See ${env.BUILD_URL} for details",
                to: env.SECURITY_CHAMPION_MAIL
            )
        }
    }
}
```
