import org.codehaus.groovy.runtime.GStringImpl;

// Set your project Prefix
def prefix      = "user9"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -s ./nexus_openshift_settings.xml"
// Set Development and Production Project Names
def projectUser = "user9"
def devProject  = "user9-tasks-dev"
def prodProject = "user9-tasks-prod"
// Set the tag for the development image: version + build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""
def gogsurl     = ""

pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "maven-appdev"
  }
  stages {
    stage ('get gogs url') {
      steps {
        script {
          gogsurl = sh(
            script: "oc get route gogs -n ${projectUser}-gogs --template='{{ .spec.host }}'",
            returnStdout: true
          ).trim()
        }
      }
    }
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
      steps {
      checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'git', url: "http://${gogsurl}/CICDLabs/openshift-tasks-private.git"]]])
       script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version

          // TBD: Set the tag for the development image: version + build number.
          // Example: def devTag  = "0.0-0"
          devTag = "${version}-${currentBuild.number}"

          // TBD: Set the tag for the production image: version
          // Example: def prodTag = "0.0"
          prodTag = "${version}"

        }
      }
    }

    // Using Maven run the unit tests
    stage('Unit Tests') {
      steps {
        echo "Running Unit Tests"

        sh("mvn test -s ./nexus_settings.xml")

      }
    }

    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build War File') {
      steps {
        echo "Building version ${devTag}"

        sh("mvn clean install -DskipTests=true -s ./nexus_settings.xml")

      }
    }

    //Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      steps {
        echo "Running Code Analysis"

        sh("mvn sonar:sonar -DskipTests=true -s ./nexus_settings.xml -Dsonar.host.url=http://\$(oc get route sonarqube -n ${projectUser}-sonarqube --template='{{ .spec.host }}')")

      }
    }

    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps {
        echo "Publish to Nexus"

        sh("mvn -s ./nexus_settings.xml deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://\$(oc get route nexus3 -n ${projectUser}-nexus --template='{{ .spec.host }}')/repository/releases")

      }
    }

    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps {
        echo "Building OpenShift container image tasks:${devTag}"

        // TBD: Start binary build in OpenShift using the file we just published.
        // Either use the file from your
        // workspace (filename to pass into the binary build
        // is openshift-tasks.war in the 'target' directory of
        // your current Jenkins workspace).
        // OR use the file you just published into Nexus:
        // "--from-file=http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war"
        dir('target') {
          sh("oc start-build tasks --from-dir=. --follow -n ${devProject}")
        }
        script {
          openshift.withCluster() {
            openshift.withProject("${devProject}") {
              openshift.tag("tasks:latest", "tasks:${devTag}")
            }
          }
        }

        // sh("oc tag tasks:latest tasks:${devTag} -n ${devProject}")

      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      steps {
        echo "Deploy container image to Development Project"

        // TBD: Deploy the image
        // 1. Update the image on the dev deployment config
        sh("oc set image dc/tasks tasks=${devProject}/tasks:${devTag} --source=imagestreamtag -n ${devProject}")
        // 2. Update the config maps with the potentially changed properties files
       // 3. Reeploy the dev deployment
        sh("oc rollout latest dc/tasks -n ${devProject}")
       // 4. Wait until the deployment is running
       //    The following code will accomplish that by
       //    comparing the requested replicas
       //    (rc.spec.replicas) with the running replicas
       //    (rc.status.readyReplicas)
       //
       script {
         openshift.withCluster() {
           openshift.withProject("${devProject}") {
             def dc = openshift.selector("dc", "tasks").object()
             def dc_version = dc.status.latestVersion
             def rc = openshift.selector("rc", "tasks-${dc_version}").object()
             echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
             while (rc.spec.replicas != rc.status.readyReplicas) {
               echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
               sleep 5
               rc = openshift.selector("rc", "tasks-${dc_version}").object()
             }
           }
         }
       }

      }
    }

    // Run Integration Tests in the Development Environment.
    stage('Integration Tests') {
      steps {
        echo "Running Integration Tests"
        script {
          def create_task_status = "000"

          // Create a new task called "integration_test_1"
          echo "Creating task"
          // The next bit works - but only after the application
          // has been deployed successfully
          create_task_status = sh(
            returnStdout: true,
            script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${projectUser}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1"
          ).trim()

          echo "Status: " + create_task_status
          if (create_task_status != "201") {
            error 'Integration Create Test Failed!'
          }

          echo "Retrieving tasks"
          retrieve_tasks = sh(
            returnStdout: true,
            script: "curl -u 'tasks:redhat1' -H 'Content-Length: 0' -X GET http://tasks.${projectUser}-tasks-dev.svc.cluster.local:8080/ws/tasks"
          ).trim()
          echo "${retrieve_tasks}"

          echo "Deleting tasks"
          delete_task = sh(
            returnStdout: true,
            script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X DELETE http://tasks.${projectUser}-tasks-dev.svc.cluster.local:8080/ws/tasks/1"
          ).trim()

        }
      }
    }

    // Copy Image to Nexus Docker Registry
    stage('Copy Image to Nexus Docker Registry') {
      steps {
        echo "Copy image to Nexus Docker Registry"

        sh("skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds=openshift:\$(oc whoami -t) --dest-creds=admin:admin123 docker://docker-registry-default.apps.7a47.openshift.opentlc.com/${projectUser}-tasks-dev/tasks:${devTag} docker://\$(oc get route nexus-registry -n ${projectUser}-nexus --template='{{ .spec.host }}')/${projectUser}-jenkins/tasks:${prodTag}")
        // TBD. Use skopeo to copy

      }
    }

    // Blue/Green Deployment into Production
    // -------------------------------------
    // Do not activate the new version yet.
    stage('Blue/Green Production Deployment') {
      steps {
        echo "Blue/Green Deployment"
        script {
          echo "Get active service"
          def get_service = sh(
            returnStdout: true,
            script: "oc get route tasks -n ${projectUser}-tasks-prod -o jsonpath='{ .spec.to.name }'"
            ).trim()

            if ( get_service == 'tasks-green' ) {
              destApp = 'tasks-blue'
            }

            sh("oc set image dc/${destApp} ${destApp}=\$(oc get route nexus-registry -n ${projectUser}-nexus --template='{{ .spec.host }}')/${projectUser}-jenkins/tasks:${prodTag} --source=docker -n ${prodProject}")

            openshift.withCluster() {
                openshift.withProject("${prodProject}") {
                  def rm = openshift.selector("dc", "${destApp}").rollout()
                  timeout(10) {
                    def latestDeploymentVersion = openshift.selector("dc","${destApp}").object().status.latestVersion
                    def rc = openshift.selector("rc", "${destApp}-${latestDeploymentVersion}")
                    rc.untilEach(1){
                      def rcMap = it.object()
                      return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
                    }
                  }
                }
            }
            openshiftVerifyService namespace: prodProject, svcName: destApp, verbose: 'false'

            // openshiftDeploy depCfg: destApp, namespace: prodProject, verbose: 'false', waitTime: '', waitUnit: 'sec'
            // openshiftVerifyDeployment depCfg: destApp, namespace: prodProject, replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '10', waitUnit: 'min'
            // openshiftVerifyService namespace: prodProject, svcName: destApp, verbose: 'false'
        }
        // TBD: 1. Determine which application is active
        //      2. Update the image for the other application
        //      3. Deploy into the other application
        //      4. Update Config maps for other application
        //      5. Wait until application is running
        //         See above for example code

      }
    }

    stage('Switch over to new Version') {
      steps {

        input "Switch Production?"
        // TBD: Stop for approval

        echo "Executing production switch"
        sh('oc patch route tasks -p \'{"spec":{"to":{"name":"' + destApp + '"}}}\' -n ' + prodProject)
        // TBD: After approval execute the switch

      }
    }
  }
}
