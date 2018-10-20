import groovy.json.JsonSlurperClassic
def GIT_REPOSITORY_ONE  = "https://vjain@bitbucket.hsdp.io/scm/alcon/alcon-dhs-common-utilities.git"
def GIT_REPOSITORY_TWO  = "https://vjain@bitbucket.hsdp.io/scm/alcon/alcon-analyzor-server.git"
def BEHAVIOUR ="icataract-microservices/icataract-iam-services/.*"
def version, revision
def BRANCH = 'master'

// VERSION & RELEASE SPECIFIED HERE
def getVersion(def projectJson){
    def slurper = new JsonSlurperClassic()
    project = slurper.parseText(projectJson)
    slurper = null
    return project.version.split('-')[0]
}

// REPOSITORY CLONE FROM GIT
def CloneFromGit( REPOSITORY_NAME,BRANCH,BEHAVIOUR ){
    def version, revision
    try {
        checkout([$class: 'GitSCM', 
            branches: [[name: "${BRANCH}"]], 
            doGenerateSubmoduleConfigurations: false, 
            extensions: [[$class: 'DisableRemotePoll'],[$class: 'PathRestriction', excludedRegions: '', includedRegions: "${BEHAVIOUR}"]],
            submoduleCfg: [], 
            userRemoteConfigs: [[
                credentialsId: 'bitbucketcredentialID', 
                url: "${REPOSITORY_NAME }"
                ]]
            ])
    }
    catch (Exception e) {
        println 'Repo Cloning failed'
        throw e

    }
    finally {
        revision = version + "-" + sprintf("%04d", env.BUILD_NUMBER.toInteger())
        println "Start building revision $revision"

    }
    return this
}

// MAVEN BUILD FUNCTION
def MavenBuild(command){
    def mavenTools = tool name: 'alcon-maven', type: 'maven'
    def mvnCMD = "${mavenTools}/bin/mvn"
    sh "${mvnCMD} ${command}"
    
    return this
}

// PUSH TO NEXUS
def nexusPush(artifactID){
    withCredentials([file(credentialsId: 'nexuscredentialID', variable: 'nexusCredentials')]) {
            nexusPublisher nexusInstanceId: 'alcon-dhs-release', 
                           nexusRepositoryId: 'alcon-dhs-release', 
                           packages: [[$class: 'MavenPackage', 
                                    mavenAssetList: [[
                                        classifier: 'debug', 
                                        extension: '', 
                                        filePath: "${artifactID}/target/${artifactID}-1.0-SNAPSHOT.jar"
                                    ]], 
                                    mavenCoordinate: [
                                        artifactId: "${artifactID}", 
                                        groupId: 'com.alcon.dhs', 
                                        packaging: 'jar', 
                                        version: '1.0.0'
                                    ]
                           ]]
        
            }
            return this
}

// ANALYTSIS WITH SonarQube

// SONAR ANALYSIS (Utilities)
def sonarUtilAnalysis(){
    def sonarTools = tool name: 'AlconSonarQube_Scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
    withSonarQubeEnv('Alcon_SonarQube') {
        def sonarCMD="""sonar:sonar \
            -Dsonar.projectKey=alcon-dhs-common-utilities-nexus \
            -Dsonar.projectVersion=1.0 \
            -Dsonar.exclusions=**/dto/*.java,**/test/**/*.java,**/src/main/java/com/alcon/dhs/**/constants/*.java,**/src/main/java/**/*Application*.java,**/src/main/java/*Properties.java,*.xml,*.properties """
        MavenBuild(sonarCMD)
    }
    return this
}
// SONAR ANALYSIS (IAM Services)
def sonarIAMAnalysis(){
    def sonarTools = tool name: 'AlconSonarQube_Scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
    withSonarQubeEnv('Alcon_SonarQube') {
        def sonarCMD="""sonar:sonar \
            -Dsonar.projectKey=icataract-authentication-services \
            -Dsonar.projectVersion=1.0 \
            -Dsonar.exclusions=**/dto/*.java,**/src/main/java/**/*Constants*.java,**/src/main/java/**/*Application*.java,**/test/**/*.java,**/*.js,*.xml,*.properties"""
        MavenBuild(sonarCMD)
    }
    return this
}


// DEPLOY TO CLOUD FOUNDRY
def pushToCFoundry(){
    pushToCloudFoundry(
        target: 'https://api.cloud.pcftest.com',
        organization: 'suite-alcon',
        cloudSpace: 'dev-mobiquity', 
        credentialsId: 'cf-suite-alcon',
        manifestChoice: [
            value: 'jenkinsConfig',
            appName: 'icataract-iam-services', 
            hostname: 'icataract-iam-services', 
            instances: '1', 
            memory: '1024', 
            timeout: '60',
            appPath: 'target/icataract-iam-services-1.0-SNAPSHOT.jar'
            ]
    )     
         
    return this
}
// SEND EMAIL NOTIFICATION
def notifyBuild(String buildStatus = 'Jenkins Job Started'){
    buildStatus = buildStatus ?: 'Jenkins Job Successful'
    def changes = getChangString()
    def subject = "${buildStatus}: Job  ${env.JOB_NAME}[${env.BUILD_NUMBER}]" 
    def summery = "${subject}(${env.BUILD_URL})"
    def details = """Build Started : Job '${env.JOB_NAME}[${env.BUILD_NUMBER}]'\n
            Check Console Output at : ${env.BUILD_URL} \n
            ${changes}"""
    
    emailext(
        subject: subject,
        body: details,
        recipientProviders:  [[$class: 'DevelopersRecipientProvider']] 
        //to: 'uzzal2k5@gmail.com'
    )
    return this
}
// milestone label: 'BuildLabel'
def getChangString(){
    MAX_MSG_LEN = 100
    def changeString = ""
    echo "Gathering SCM chnages"
    def changeLogSets =  currentBuild.rawBuild.changeSets
    for (int i =0; i < changeLogSets.size(); i++){
        def entries = changeLogSets[i].items
        for(int j = 0; j < entries.length; j++){
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += "-  ${truncated_msg} [${entry.author}] \n"
        }
    }
    if(! changeString){
        changeString = " - No new changes"
    }
    return changeString
}
//def BEHAVIOUR = ''
// START NODE 
node {
    properties([
        pipelineTriggers([
            bitbucketPush()
            ])
        ])
    try{
        notifyBuild('Jenkins Job Started')
        def sonaBuild = "clean install -P sonar-build"
        
        // UTILITY CLONING FROM GIT STAGE
        stage(' UTILITY CLONE'){
            deleteDir()
            CloneFromGit(GIT_REPOSITORY_ONE, BRANCH,'')
        }
        dir(""){
            // BUILD COMMON UTILITIES STAGE
            stage('UTILITY BUILD') {
                MavenBuild(sonaBuild)
            }
            // PUSH TO NEXUS STAGE
            stage('PUSH TO NEXUS'){
                for ( artifactID in [
                    'dhs-commons-service',
                    'dhs-commons-filters',
                    'hsdp-auditing-integration',
                    'hsdp-cdr-integration',
                    'hsdp-iam-integration',
                    'hsdp-logging-integration',
                    'hsdp-rds-integration',
                    'hsdp-redis-integration'
                    ]){
                        nexusPush(artifactID)
                    }
            }
            // UTILITY SONAR ANALYSIS STAGE
            stage('UTILITY ANALYSIS') {          
                sonarUtilAnalysis()         
            }
            
        }
        
        // IAM CLONING FROM GIT STAGE
        stage('IAM CLONE') {
            //def BEHAVIOUR = "[$class: 'DisableRemotePoll'],[$class: 'PathRestriction', excludedRegions: '', includedRegions: 'icataract-microservices/icataract-iam-services/.*']"
            deleteDir()      
            CloneFromGit(GIT_REPOSITORY_TWO, BRANCH,BEHAVIOUR)       
        }
        dir("icataract-microservices/icataract-iam-services"){
            //BUILD IAM SERVICES STAGE
            stage('IAM BUILD') {    
                    MavenBuild(sonaBuild)       
            }
            // IAM SONAR ANALYSIS STAGE
            stage('IAM ANALYSIS') {                
                sonarIAMAnalysis()  
            }
            // IAM DEPLOY TO CF STAGE
            stage('DEPLOY TO CF') {                
                pushToCFoundry()  
            }
        }
    }catch(Exception e){
        currentBuild.result = "FAILED"
        throw e
    }finally{
        // POST NOTIFICATION SEND
        notifyBuild(currentBuild.result)      
    }
    
 // END NODE      
}
