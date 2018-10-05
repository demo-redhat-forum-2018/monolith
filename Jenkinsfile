#!groovy
/*
 * This pipeline accepts the following parameters :
 *  - OPENSHIFT_IMAGE_STREAM: The ImageStream name to use to tag the built images
 *  - OPENSHIFT_BUILD_CONFIG: The BuildConfig name to use
 *  - OPENSHIFT_SERVICE: The Service object to update (either green or blue)
 *  - OPENSHIFT_DEPLOYMENT_CONFIG: The DeploymentConfig name to use
 *  - OPENSHIFT_BUILD_PROJECT: The OpenShift project in which builds are run
 *  - OPENSHIFT_TEST_ENVIRONMENT: The OpenShift project in which we will deploy the test version
 *  - OPENSHIFT_PROD_ENVIRONMENT: The OpenShift project in which we will deploy the prod version
 *  - NEXUS_REPO_URL: The URL of your Nexus repository. Something like http://<nexushostname>/repository/maven-snapshots/
 *  - NEXUS_MIRROR_URL: The URL of your Nexus public mirror. Something like http://<nexushostname>/repository/maven-all-public/
 *  - NEXUS_USER: A nexus user allowed to push your software. Usually 'admin'.
 *  - NEXUS_PASSWORD: The password of the nexus user. Usually 'admin123'.
 *  - OVH_URL
 *  - OVH_TOKEN
 *  - AZURE_URL
 *  - AZURE_TOKEN
 *  - QUAY_REGISTRY
 *  - QUAY_USER
 *  - QUAY_PASSWORD
 */

def newVersion
def azureURL = env.AZURE_URL
def azureToken = env.AZURE_TOKEN
def ovhURL = env.OVH_URL
def ovhToken = env.OVH_TOKEN
def openShiftBuildEnv = env.OPENSHIFT_BUILD_PROJECT
def openShiftTestEnv = env.OPENSHIFT_TEST_ENVIRONMENT
def openShiftProdEnv = env.OPENSHIFT_PROD_ENVIRONMENT

node('maven') {

    stage ("Get Source code"){
        echo '*** Build starting ***'
        def mvn = "mvn -s mvn-settings.xml"
        sh "mvn --version"
        git url : 'https://github.com/demo-redhat-forum-2018/monolith.git'
                  
        //write Nexus configfile
        writeFile file: 'mvn-settings.xml', text: """
<?xml version="1.0"?>
<settings>
  <mirrors>
    <mirror>
      <id>Nexus</id>
      <name>Nexus Public Mirror</name>
      <url>${params.NEXUS_MIRROR_URL}</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>nexus</id>
      <username>${params.NEXUS_USER}</username>
      <password>${params.NEXUS_PASSWORD}</password>
    </server>
  </servers>
</settings>
"""
    }

    def pom            = readMavenPom file: 'pom.xml'
    def packageName    = pom.name
    def version        = pom.version
    newVersion     = "${version}-${BUILD_NUMBER}"
    def artifactId     = pom.artifactId
    def groupId        = pom.groupId

    // Using Mav build the war file
    stage('Build war file') {
        sh "mvn -s mvn-settings.xml clean install -DskipTests=true"
    }         
    
    // Using Maven run the unit tests
    stage('Unit Tests') {
      //sh "${mvn} test"
      sh "sleep 5"
    }

    stage('Publish to Nexus') {
        sh "mvn -s mvn-settings.xml deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::${params.NEXUS_REPO_URL}"
    }

    stage('Build OpenShift Image') {

        // Determine the war filename that we need to use later in the process
        String warFileName = "${groupId}.${artifactId}"
        warFileName = warFileName.replace('.', '/')
        def WAR_FILE_URL = "${params.NEXUS_REPO_URL}/${warFileName}/${version}/${artifactId}-${version}.war"
        echo "Will use WAR at ${WAR_FILE_URL}"

        openshift.withCluster(ovhURL, ovhToken) {
            openshift.withProject(openShiftBuildEnv) {
                openshift.selector("bc", "coolstore").startBuild("-e WAR_FILE_URL=${WAR_FILE_URL}","--wait=true")
            }
        }
        
    }
}

node('jenkins-slave-skopeo') {

    stage('Tag Version') {
        promoteImage("latest", newVersion)
    }

    stage('Deploy to TEST') {

        promoteImage(newVersion,"promoteToTest")

        // Trigger a new deployment
        openshift.withCluster(azureURL, azureToken) {
            openshift.withProject(openShiftTestEnv) {
                def dc = openshift.selector('dc', 'coolstore')
                dc.rollout().latest();
                dc.rollout().status();
            }
        }

        // PLACEHOLDER SLACK (Send TEST Route URL)
    }

    stage('Integration Test') {
        sh "sleep 7"        
    }

    stage('Security Test') {
        sh "sleep 4" 
        // call Quay to get scan result        
    }

    stage('Promote to PROD') {
        promoteImage(newVersion,"promoteToProd")
    }

    stage('Deploy to PROD') {
        openshift.withCluster(ovhURL, ovhToken) {
            openshift.withProject(openShiftProdEnv) {
                
                def coolstorerouteweight = openshift.selector('route',"coolstore").object().spec.to.weight
                sh "echo $coolstorerouteweight > weight"
                def newTarget = getNewTarget()
                def dcname = env.OPENSHIFT_DEPLOYMENT_CONFIG + '-' + newTarget

                def dc = openshift.selector('dc', dcname)
                dc.rollout().latest();
                dc.rollout().status();
            }
        }
    }


    stage('Switch over to new Version') {

        openshift.withCluster(ovhURL, ovhToken) {
            openshift.withProject(openShiftProdEnv) {

                def newTarget = getNewTarget()
                def currentTarget = getCurrentTarget()

                input "Switch Production from coolstore-${currentTarget} to coolstore-${newTarget} ?"

                if (newTarget == "blue"){
                    openshift.set("route-backends", "coolstore", "coolstore-blue=100%", "coolstore-green=0%")
                } else {
                    openshift.set("route-backends", "coolstore", "coolstore-blue=0%", "coolstore-green=100%")
                }
            }
        }
    }
}

def getCurrentTarget() {
    def weight = readFile 'weight'
    weight = weight.trim()
    if (weight == "0") {
      currentTarget = "green"
    } else {
      currentTarget = "blue"
    }
    return currentTarget
}

def getNewTarget() {
    def currentTarget = getCurrentTarget()
    def newTarget = ""
    if (currentTarget == "blue") {
        newTarget = "green"
    } else if (currentTarget == "green") {
        newTarget = "blue"
    } else {
        echo "OOPS, wrong target"
    }
    return newTarget
}

def promoteImage(srcImageTag,destImageTag) {
    def from = "docker://${QUAY_REGISTRY}:${srcImageTag}"
    def to = "docker://${QUAY_REGISTRY}:${destImageTag}"

    echo "Now Promoting ${from} -> ${to}"
    sh """
        set +x
        skopeo copy --remove-signatures \
        --src-creds ${QUAY_USER}:${QUAY_PASSWORD} \
        --dest-creds ${QUAY_USER}:${QUAY_PASSWORD}  --dest-tls-verify=false ${from} ${to}
    """
}