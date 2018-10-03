#!groovy
/*
 * This Jenkins Pipeline depends on the following plugins :
 *  - Pipeline Utility Steps (https://plugins.jenkins.io/pipeline-utility-steps)
 *  - Credentials Binding (https://plugins.jenkins.io/credentials-binding)
 *
 * This pipeline accepts the following parameters :
 *  - OPENSHIFT_IMAGE_STREAM: The ImageStream name to use to tag the built images
 *  - OPENSHIFT_BUILD_CONFIG: The BuildConfig name to use
 *  - OPENSHIFT_SERVICE: The Service object to update (either green or blue)
 *  - OPENSHIFT_DEPLOYMENT_CONFIG: The DeploymentConfig name to use
 *  - OPENSHIFT_BUILD_PROJECT: The OpenShift project in which builds are run
 *  - OPENSHIFT_TEST_ENVIRONMENT: The OpenShift project in which we will deploy the test version
 *  - OPENSHIFT_PROD_ENVIRONMENT: The OpenShift project in which we will deploy the prod version
 *  - OPENSHIFT_TEST_URL: The App URL in the test environment (to run the integration tests)
 *  - NEXUS_REPO_URL: The URL of your Nexus repository. Something like http://<nexushostname>/repository/maven-snapshots/
 *  - NEXUS_MIRROR_URL: The URL of your Nexus public mirror. Something like http://<nexushostname>/repository/maven-all-public/
 *  - NEXUS_USER: A nexus user allowed to push your software. Usually 'admin'.
 *  - NEXUS_PASSWORD: The password of the nexus user. Usually 'admin123'.
 */

node('maven') {

    slackSend channel: 'monolith', color: 'green', message: "started ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"

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
    def newVersion     = "${version}-${BUILD_NUMBER}"
    def artifactId     = pom.artifactId
    def groupId        = pom.groupId

    // Using Mav build the war file
    stage('Build war file') {
        sh "mvn -s mvn-settings.xml clean install -DskipTests=true"
    }         
    
    // Using Maven run the unit tests
    stage('Unit Tests') {
      sh "${mvn} test"
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

        // Trigger an OpenShift build in the build environment
        openshiftBuild bldCfg: params.OPENSHIFT_BUILD_CONFIG, checkForTriggeredDeployments: 'false',
                  namespace: params.OPENSHIFT_BUILD_PROJECT, showBuildLogs: 'true',
                  verbose: 'false', waitTime: '', waitUnit: 'sec',
                  env: [ [ name: 'WAR_FILE_URL', value: "${WAR_FILE_URL}" ] ]

        // Tag the new build
        openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, destTag: "${newVersion}",
                destinationNamespace: params.OPENSHIFT_BUILD_PROJECT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                srcStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: 'latest', verbose: 'false'
    }

    stage('Deploy to TEST') {
        openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: "${newVersion}",
                destinationNamespace: params.OPENSHIFT_BUILD_PROJECT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                srcStream: params.OPENSHIFT_IMAGE_STREAM, destTag: 'promoteToTest', verbose: 'false'

        // Trigger a new deployment
        openshiftDeploy deploymentConfig: 'coolstore', namespace: params.OPENSHIFT_BUILD_PROJECT
    }

    stage('Integration Test') {
        sh "sleep 7"

        // Tag the new build as "ready-for-prod"
        openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: "${newVersion}",
                destinationNamespace: params.OPENSHIFT_PROD_ENVIRONMENT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                srcStream: params.OPENSHIFT_IMAGE_STREAM, destTag: 'promoteToProd', verbose: 'false'
    }

    stage('Deploy to PROD') {
        // Yes, this is mandatory for the next command to succeed. Don't know why...
        sh "oc project ${params.OPENSHIFT_PROD_ENVIRONMENT}"

        // Extract the route target (xxx-green or xxx-blue)
        // This will be used by getCurrentTarget and getNewTarget methods
        sh "oc get route coolstore -n ${params.OPENSHIFT_PROD_ENVIRONMENT} -o template --template='{{ .spec.to.weight }}' > weight"

        // Flip/flop target (green goes blue and vice versa)
        def newTarget = getNewTarget()

        // Trigger a new deployment
        openshiftDeploy deploymentConfig: "${params.OPENSHIFT_DEPLOYMENT_CONFIG}-${newTarget}", namespace: params.OPENSHIFT_PROD_ENVIRONMENT
        openshiftVerifyDeployment deploymentConfig: "${params.OPENSHIFT_DEPLOYMENT_CONFIG}-${newTarget}", namespace: params.OPENSHIFT_PROD_ENVIRONMENT
        // Trigger a new deployment
    }


    stage('Switch over to new Version') {
        // Determine which is of green or blue is active
        def newTarget = getNewTarget()
        def currentTarget = getCurrentTarget()

        // Wait for administrator confirmation
        input "Switch Production from ${currentTarget} to ${newTarget} ?"

        if (${newTarget} == "blue"){
            // Switch blue/green
            sh """
              oc patch -n ${params.OPENSHIFT_PROD_ENVIRONMENT} route/coolstore --patch '
                {\"spec\":
                  {\"to\":
                    {\"kind\":\"Service\"},
                    {\"name\":\"coolstore-blue\"},
                    {\"weight\":\"100\"}
                  },
                  {\"alternateBackends\":
                    [{{\"kind\":\"Service\"},
                    {\"name\":\"coolstore-green\"},
                    {\"weight\":\"0\"}}]
                  }
                }'
            """
        } else {
            // Switch blue/green
            sh """
              oc patch -n ${params.OPENSHIFT_PROD_ENVIRONMENT} route/coolstore --patch '
                {\"spec\":
                  {\"to\":
                    {\"kind\":\"Service\"},
                    {\"name\":\"coolstore-blue\"},
                    {\"weight\":\"0\"}
                  },
                  {\"alternateBackends\":
                    [{{\"kind\":\"Service\"},
                    {\"name\":\"coolstore-green\"},
                    {\"weight\":\"100\"}}]
                  }
                }'
            """
        }
    }
}

def getCurrentTarget() {
    def currentTarget = getCurrentWeight()
    if (currentTarget == 0) {
      return "green"
    } else {
      return "blue"
    }
}

def getCurrentWeight() {
    def currentWeight = readFile 'weight'
    return currentWeight
}

// Flip/flop target (green goes blue and vice versa)
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

