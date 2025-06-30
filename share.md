@Library('jenkins-devops-cicd-library') _
def ListNugetSourcesFromArtifactory() {
    try {
        withCredentials(bindings:[usernamePassword(credentialsId:'STAR-ARTIFACTORY',usernameVariable:'USERNAME',passwordVariable:'PASSWORD')]) {
            bat returnStdout:true, script:'dotnet nuget list source'
        }
    } catch(err) { echo "Caught: ${err}" }
}
def RemoveNugetSourcesFromArtifactory(name) {
    try {
        withCredentials(bindings:[usernamePassword(credentialsId:'STAR-ARTIFACTORY',usernameVariable:'USERNAME',passwordVariable:'PASSWORD')]) {
            bat returnStdout:true, script:"dotnet nuget remove source ${name}"
        }
    } catch(err) { echo "Caught: ${err}" }
}
def AddNugetSourcesFromArtifactory(source,name) {
    RemoveNugetSourcesFromArtifactory(name)
    try {
        withCredentials(bindings:[usernamePassword(credentialsId:'STAR-ARTIFACTORY',usernameVariable:'USERNAME',passwordVariable:'PASSWORD')]) {
            bat returnStdout:true, script:"dotnet nuget add source ${source} --name ${name} --username ${USERNAME} --password ${PASSWORD} --store-password-in-clear-text"
        }
    } catch(err) { echo "Caught: ${err}" }
}
def GenerateArtifactoryUploadSpecification(config) {
    def spec=[pattern:config.path,target:"${config.repository}/com/bnpparibas/${config.name}/${config.branch}/${config.version}/${config.fullname}",props:"build.commit=${env.GIT_COMMIT};build.version=${config.version};build.branch=${env.GIT_BRANCH}"]
    writeJSON returnText:true, json:spec
}
def PromotePackageToArtifactory(config) {
    try {
        withCredentials(bindings:[usernamePassword(credentialsId:'STAR-ARTIFACTORY',usernameVariable:'USERNAME',passwordVariable:'PASSWORD')]) {
            def buildInfo=Artifactory.newBuildInfo()
            buildInfo.env.capture=false
            buildInfo.name=config.name
            buildInfo.number=config.version
            def server=Artifactory.newServer(url:'https://artifactory.cib.echonet/artifactory',credentialsId:'STAR-ARTIFACTORY')
            def uploadSpec="{\"files\":[${GenerateArtifactoryUploadSpecification(config)}]}"
            timeout(time:10,unit:'MINUTES'){ retry(3){ server.upload spec:uploadSpec,buildInfo:buildInfo; server.publishBuildInfo buildInfo } }
        }
    } catch(err) { echo "Caught: ${err}" }
}
def ResolveArtifactoryRepository() {
    (env.BRANCH_NAME=='master')?'star-generic-local-release':'star-generic-local-dev'
}
def ResolveVersion() {
    if(!fileExists('.\\build\\version.txt')) {
        def c=readFile('.\\StarTrends\\StarTrendsDashboard\\StarTrendsDashboard.csproj')
        def m=(c=~/<AssemblyVersion>([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)<\/AssemblyVersion>/)
        if(!m) error 'AssemblyVersion not found'
        return m[0][1].tokenize('.').take(3).join('.')
    } else {
        return readFile('.\\build\\version.txt').trim()
    }
}
pipeline {
    agent { node{ label 'windows_2022_VS_2022'; customWorkspace 'workspace/star/StarTrends' } }
    options{ timestamps() }
    parameters{
        string(name:'VersionNumberPrefix',defaultValue:'1.0.0',description:'')
        choice(name:'Config',choices:['Release','Debug'],description:'')
    }
    environment{
        PROJECT_REPOSITORY='ssh://git@bitbucket.cib.echonet:7999/star/StarTrends.git'
        PROJECT_BRANCH='Dev-StarTrends'
        PROJECT_NAME='star.trends.dashboard'
        SOLUTION_FILE='StarTrends/StarTrendsDashboard/StarTrendsDashboard.sln'
        CONFIG_FILE='build\\Nuget.config'
        STAR_NUGET_REPO_SOURCE='https://artifactory.cib.echonet/artifactory/api/nuget/star-nuget'
        STAR_NUGET_REPO_NAME='star-nuget'
        EXTERNAL_NUGET_REPO_SOURCE='https://artifactory.cib.echonet/artifactory/api/nuget/external-nuget-local'
        EXTERNAL_NUGET_REPO_NAME='external-nuget-local'
        NUGET_REMOTE_CACHE_SOURCE='https://artifactory.cib.echonet/artifactory/api/nuget/nuget-remote-cache'
        NUGET_REMOTE_CACHE_NAME='nuget-remote-cache'
        MSBuild17=tool 'MSBuild_17.0'
        MSBUILD="${MSBuild17}"
    }
    stages{
        stage('Setup current build name'){
            steps{ script{ env.BRANCH_NAME=PROJECT_BRANCH; env.VERSION_NUMBER="${ResolveVersion()}.${BUILD_NUMBER}"; currentBuild.displayName=VERSION_NUMBER; env.ZIP_NAME="${PROJECT_NAME}-${VERSION_NUMBER}.zip"; env.ARTIFACTORY_REPO=ResolveArtifactoryRepository() } }
        }
        stage('Checkout'){
            steps{
                git url:PROJECT_REPOSITORY,branch:PROJECT_BRANCH,credentialsId:'STAR-BITBUCKET-SSH'
                script{ env.GIT_COMMIT=bat(returnStdout:true,script:'git rev-parse --short HEAD').trim() }
            }
        }
        stage('Create needed folders'){ steps{ bat 'if not exist "D:\\data\\reportingapi" mkdir "D:\\data\\reportingapi"' } }
        stage('Get commitId and latest tag'){ steps{ script{ env.LATEST_TAG=bat(returnStdout:true,script:'git describe --always --abbrev=0').trim() } } }
        stage('Get artifact version'){ steps{ /* merged into Setup build */ } }
        stage('Delete build directory'){ steps{ script{ ['StarTrendsDashboard','StarTrendsDashboard.Shared'].each{ p-> bat "if exist ${p}\\bin rmdir /S /Q ${p}\\bin" } } } }
        stage('Set version for all dlls and exes'){ steps{ powershell''' Get-ChildItem . -Recurse -Filter *.csproj | ForEach-Object { (Get-Content $_.FullName) -replace '<Version>1.1.1.1</Version>',"<Version>$env:VERSION_NUMBER</Version><RuntimeIdentifier>win-x64</RuntimeIdentifier>" | Set-Content $_.FullName } ''' } }
        stage('Add Nuget credentials'){ steps{ script{ AddNugetSourcesFromArtifactory(STAR_NUGET_REPO_SOURCE,STAR_NUGET_REPO_NAME); AddNugetSourcesFromArtifactory(EXTERNAL_NUGET_REPO_SOURCE,EXTERNAL_NUGET_REPO_NAME); AddNugetSourcesFromArtifactory(NUGET_REMOTE_CACHE_SOURCE,NUGET_REMOTE_CACHE_NAME) } } }
        stage('Dotnet clean'){ steps{ bat "dotnet clean --configuration ${params.Config}" } }
        stage('Restore with Nuget'){ steps{ bat "\"${MSBUILD}\" -version\""; bat "dotnet restore ${SOLUTION_FILE} -r win-x64 --configfile ${CONFIG_FILE}" } }
        stage('Remove all override config file settings'){ steps{ powershell''' Get-ChildItem . -Recurse -Filter *.json | ForEach-Object { (Get-Content $_.FullName) -replace 'FileOverrideLocation','FileOverrideLocation_' | Set-Content $_.FullName } ''' } }
        stage('Build'){ steps{ bat "dotnet build ${SOLUTION_FILE} -c ${params.Config}" } }
        stage('Run unit tests'){ steps{ bat "dotnet test -c ${params.Config} --no-build --no-restore" } }
        stage('Create Fortify Version'){ steps{ node('Bnpp-Maven3-SecOps'){ withCredentials([string(credentialsId:'fortify-rest-api-token-star',variable:'FORTIFY_REST_API_TOKEN')]){ createNewFortifyApplicationVersionBasedOnLastScan('https://fortifyssc.cib.echonet/ssc',FORTIFY_REST_API_TOKEN,'STAR.HUDSON',"0.0.${BUILD_NUMBER}") } } } }
        stage('Fortify Scan/Analysis'){ steps{ withCredentials([string(credentialsId:'fortify-rest-api-token-star',variable:'FORTIFY_REST_API_TOKEN'),string(credentialsId:'fortify-report-token-star',variable:'FORTIFY_SCAN_CENTRAL_TOKEN')]){ bat 'fortifyupdate -url https://fortifyssc.cib.echonet/ssc -acceptSSLCertificate -acceptKey'; bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -clean -b STAR.HUDSON"; bat "sourceanalyzer -Dcom.fortify.sca.ProjectRoot=.fortify -Xmx5G -b STAR.HUDSON -logfile reporting-api-fortify.log -verbose \"${MSBUILD}\" ${SOLUTION_FILE} /p:Configuration=${params.Config}"; bat """scancentral -sscurl https://fortifyssc.cib.echonet/ssc -ssctoken ${FORTIFY_SCAN_CENTRAL_TOKEN} start -projroot .fortify -b STAR.HUDSON -email harpreet.nanra@uk.bnpparibas.com -f STAR.HUDSON.fpr -log STAR.HUDSON-scan.log -upload -application STAR.HUDSON -version 0.0.${BUILD_NUMBER} -uptoken ${FORTIFY_SCAN_CENTRAL_TOKEN} -scan""" } } }
        stage('build EXE binaries'){ steps{ script{ bat "if exist output rmdir /S /Q output"; ['StarTrendsDashboard','StarTrendsDashboard.Shared'].each{ p-> dir(p){ bat "dotnet publish --no-restore -c ${params.Config} --self-contained true --use-current-runtime true -o ../output/${p}" } } } } }
        stage('Zip'){ steps{ bat "if exist ${ZIP_NAME} del /F ${ZIP_NAME}"; zip zipFile:ZIP_NAME,archive:false,dir:'output' } }
        stage('Deploy to Artifactory'){ steps{ script{ def cfg=[repository:env.ARTIFACTORY_REPO,name:PROJECT_NAME,branch:env.BRANCH_NAME,version:env.VERSION_NUMBER,fullname:env.ZIP_NAME,path:"${env.WORKSPACE}\\StarTrends\\StarTrendsDashboard\\${env.ZIP_NAME}"]; PromotePackageToArtifactory(cfg) } } }
    }
}

