def appName = "${params.APPLICATION_NAME}"
def GIT_URL = "${params.GIT_URL}"

/*
  Persistence Volume, the purpose of this object is to cache your Jenkins workspace,
   so there is no unnecessary dependency pulling (faster builds).
*/

def PVC_MAVEN_CACHE = 'maven-cache'

/*
  Config Map configuration for your Jenkins Worker.
  Folder where to include your settings.xml, then you can do (mvn -s /cfg/settings.xml package).
*/

def CONFIG_MAP = 'my-configmap'
def CONFIG_MAP_MOUNT = '/cfg'

/*
  'jnlp'

  By naming this container 'jnlp' we tell Jenkins that this container contains also the Jenkins agent, provided by 
  this Red Hat image [ registry.redhat.io/openshift3/jenkins-agent-maven-35-rhel7:v3.11 ] 
*/

def POD_LABEL  = 'jnlp'
def JENKINS_CONTAINER_IMAGE = "registry.redhat.io/openshift3/jenkins-agent-maven-35-rhel7:v3.11"

// This is a small-hack because Jenkins seems to not include the image built-in PATH variable.
def PATH = "/opt/rh/rh-maven35/root/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

podTemplate(
  cloud: 'openshift',
  label: BUILD_TAG,
  serviceAccount: 'jenkins',

  volumes: [
    // Cache this folder with a PVC
    persistentVolumeClaim(claimName: PVC_MAVEN_CACHE, mountPath: "/home/jenkins", readOnly: false)
  ],

  containers: [
    containerTemplate(
      name: POD_LABEL,
      image: JENKINS_CONTAINER_IMAGE,
      envVars: [envVar(key: 'PATH', value: PATH)] )
  ]) {

    node (BUILD_TAG) 
    {

        container(POD_LABEL) {

          stage("Configuring Openshift Components") {
            def BUILD_SCRIPT = "https://raw.githubusercontent.com/cesarvr/Spring-Boot/master/jenkins/build.sh"
            git "${GIT_URL}"

            // This script creates Openshift objects to run your service. 
            sh "curl ${BUILD_SCRIPT} -o build.sh && chmod +x ./build.sh && ./build.sh ${appName} "
          }

          stage('Testing and Packaging') {
            try {
              sh "mvn ${JVM_OPTIONS} package"
            }finally {
              junit 'target/surefire-reports/*.xml'
              archiveArtifacts artifacts: 'target/**.jar', fingerprint: true
            }
          }

          stage('Creating Container'){
            sh "oc start-build bc/${appName} --from-file=\$(ls target/*.jar) --follow"
          }

          stage('Deploy') {
            script {
              sh "oc rollout latest dc/${appName} || true"
              sh "oc wait dc/${appName} --for condition=available --timeout=-1s"
            }
          }
        }
    } // node
} // podTemplate
