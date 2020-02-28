Table of contents
=================

<!--ts-->
   * [Jenkins Worker Customization](#jenkins-worker-customization)
   * [Jenkins DSL](#jenkins-dsl)
   * [What Can We Do With This](#what-can-we-do-with-this)
   * [Building A Spring Boot Application](#using-this-with-the-spring-boot-project)
   * [Caching Maven Dependencies (Optimization)](#caching-maven-dependencies)
<!--te-->


## Jenkins Worker Customization

This plugin allows to use Kubernetes/Openshift [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) as Jenkins workers for job parallelization. We can use the fact that the pod loosely simulates a virtual machine to define via Jenkins DSL, the *resources* we need to build our project.

### Hello World

In this example we are going to provision a Jenkins worker/container with RHEL-7:

```java
podTemplate(cloud:'openshift', label: BUILD_TAG, containers: [
    containerTemplate(name: 'rhel', image: 'rhel7:latest', ttyEnabled: true, command: 'cat'),
  ]) {
    node(BUILD_TAG) {
        container('rhel') {
            stage('Hello') {
                echo "build: " + BUILD_TAG
                sh 'echo Hello World'
            }
            /* More stages */
        }
    }
}
```

If we run this we get:

```sh
#build: jenkins-pipeline-test-6
#[Pipeline] sh

#[pipeline-test] Running shell script

echo Hello World

#Hello World
#[Pipeline] }
#...
```

> To test this you just need to create new pipeline project in Jenkins and copy/paste this code.


### What Happened

This will start a new job in Jenkins, Then Jenkins will proceed to create a Pod with two containers:

![](https://github.com/cesarvr/cicd/raw/master/img/jnlp-1.PNG)

> Openshift then will take care of this booting up one pod with two containers, while Jenkins will wait for the JNLP (Jenkins agent).


![](https://github.com/cesarvr/cicd/raw/master/img/jnlp-2.PNG)

> Here we got two containers, one is our RHEL7 container, the other is running a container which contains the Jenkins agent client which will connect back to Jenkins Master to receive instructions.



![](https://github.com/cesarvr/cicd/raw/master/img/jnlp-3.PNG)

> Once connected Jenkins then will push the Job inside the ``container('rhel')`` to the container with that name.



### Jenkins DSL

Here is a quick explanation of the ``hello world`` we saw above:

```java
  podTemplate(cloud:'openshift', label: BUILD_TAG, containers: [/*..*/] ){}
```
Defines the [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) that will run our Jenkins, in this pod you can run one or more containers which be very important when we want to create more sophisticated builds, but for now think of this in just a way to externalize work from the Jenkins master.


```java
  containerTemplate(name: 'rhel', image: 'rhel7:latest', ttyEnabled: true, command: 'cat'),
```

Here we just saying that we want a ``rhel7`` container running inside the pod. Then we add the ``ttyEnabled`` and ``cat`` which basically will lock the main process until the Jenkins jobs finish.


### What Can We Do With This

We want to decouple from the Jenkins master because:
- If something goes wrong with the Jenkins master you just need to deploy a new one.
- Independence. Your code can run in any Jenkins instance in the cluster, making your code build/deploy/test elsewhere.
- We keep one source of knowledge for the whole pipeline project as in infrastructure as code.

#### Adding A Config Map

So let's say we want to include a configuration file in the build above:

First let's start by defining one:

```sh
echo "Hola Mundo" >> hello.txt
```

Define a new Config Map using ``oc-cli``:

```sh
oc create configmap hello-es --from-file=hello.txt
#configmap/hello-es created
```

Now that we got the Configuration Map, we need to mount it as a volume in the pod for that we can use ``configMapVolume`` to define it.  


```java
podTemplate(cloud:'openshift', label: BUILD_TAG,

volumes: [configMapVolume(configMapName: "hello-es", mountPath: "/my-config")],

containers: [
    containerTemplate(name: 'rhel', image: 'rhel7:latest', ttyEnabled: true, command: 'cat',  ),
  ]) {
    node(BUILD_TAG) {
        container('rhel') {
            stage('Hello') {
                echo "build: " + BUILD_TAG
                sh 'echo Hello World'
                echo "spanish hello: "
                sh "cat /my-config/hello.txt"
            }
            /* More stages */
        }
    }
}
```
> Here we just say that we want ``hello-es`` configuration map and we want to mount it inside the root folder of our container folder ``/my-config``.

When we run this we get:

```sh
#build: jenkins-pipeline-test-8
#[Pipeline] sh
#[pipeline-test] Running shell script

+ echo Hello World
Hello World
[Pipeline] echo
spanish hello:
#[Pipeline] sh
#[pipeline-test] Running shell script

+ cat /my-config/hello.txt
Hola Mundo
#[Pipeline] }
#...
#...
#[Pipeline] End of Pipeline
#Finished: SUCCESS

```

You see, we no longer need to touch the Jenkins master to setup our configuration.



### Using This With The Spring Boot Project

Here a quick example of a minimal pipeline to build and deploy the [Spring Boot](https://github.com/cesarvr/java-microservice) project, with a [Persistence Volume Claim (PVC)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) to cache our dependencies:


```groovy
def appName = "${params.APPLICATION_NAME}"
def PROXY   = "${params.PROXY}"
def GIT_URL = "${params.GIT_URL}"
def JVM_OPTIONS = "-DproxySet=true -DproxyHost=${PROXY} -DproxyPort=8080"

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
  Jenkins Specific Configuration
*/

def POD_LABEL  = 'jnlp'
def JENKINS_WORKING_DIR = "/home/jenkins"
def JENKINS_CONTAINER_IMAGE = "registry.redhat.io/openshift3/jenkins-agent-maven-35-rhel7:v3.11"
def JENKINS_PATH = "/opt/rh/rh-maven35/root/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

podTemplate(
  cloud: 'openshift',
  label: BUILD_TAG,
  serviceAccount: 'jenkins',

  volumes: [
    configMapVolume(configMapName: CONFIG_MAP, mountPath: CONFIG_MAP_MOUNT),
    persistentVolumeClaim(claimName: PVC_MAVEN_CACHE, mountPath: JENKINS_WORKING_DIR, readOnly: false)
  ],

  containers: [containerTemplate(
    name: POD_LABEL,
    image: JENKINS_CONTAINER_IMAGE,
    envVars: [envVar(key: 'PATH', value: JENKINS_PATH)],
    workingDir: JENKINS_WORKING_DIR ) ]
  ) {
    node (BUILD_TAG) {

        container(POD_LABEL) {

          stage("Configuring Openshift Components") {
            def BUILD_SCRIPT = "https://raw.githubusercontent.com/cesarvr/Spring-Boot/master/jenkins/build.sh"

            git "${GIT_URL}"
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
    }
}
```
> Aside from adding a configuration map we also add a persistent volume claim to cache the maven dependency this way we can increase the speed of our builds.

#### Testing

```sh
# Create the pipeline
oc new-build https://github.com/cesarvr/cicd.git --name=service-x --strategy=pipeline

# Configure the pipeline
oc set env bc/service-x APPLICATION_NAME=service-xyz /
  GIT_URL=https://github.com/cesarvr/Spring-Boot.git

# To start a new build
oc start-build service-x --follow
```

### Caching Maven Dependencies

If you run the script above you will see that it will try to mount the config map and the PVC, but if don't have any of those then you will see that it won't complain. To make sure next time our build is cached we are going to add a PVC, the easiest way is using the menu:

![](https://github.com/cesarvr/cicd/raw/master/img/cicd-simple-1.PNG)

> In the Openshift console click ``storage -> create storage``.

Just make sure you use the same name than here:

```java
  def PVC_MAVEN_CACHE = 'maven-cache'
```
