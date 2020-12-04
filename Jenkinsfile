// Set your project Prefix using your GUID
def prefix      = "7671"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -s ./nexus_openshift_settings.xml"
def mvnSonarCmd = "mvn -s ./sonar_settings.xml"
// Set Development and Production Project Names
def devProject  = "${prefix}-tasks-dev"
def prodProject = "${prefix}-tasks-prod"
// Set the tag for the development image: version + build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""

pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "maven-appdev"
  }
  stages {
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
      steps {
        // TBD: Get code from protected Git repository
        git credentialsId: '2783a10f-ff66-411e-8c70-814f2f82d32a', url: 'https://gogs-gogs-7671-gogs.apps.cluster-c78c.c78c.example.opentlc.com/CICDLabs/openshift-tasks-private.git'

       script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version

          // TBD: Set the tag for the development image: version + build number.
          // Example: def devTag  = "0.0-0"
          devTag = pom.version + "-" + currentBuild.number

          // TBD: Set the tag for the production image: version
          // Example: def prodTag = "0.0"
          prodTag = pom.version

          echo "Dev build tag is ${devTag}"           
          echo "Prod build tag is ${prodTag}"

        }
      }
    }

    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build War File') {
      steps {
        echo "Building version ${devTag}"
        sh "${mvnCmd} clean package -DskipTests"

      }
    }

    // Using Maven run the unit tests
    stage('Unit Tests') {
      steps {
        echo "Running Unit Tests"
        sh "${mvnCmd} test"
      }
    }

    //Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      steps {
        echo "Running Code Analysis"
        sh "${mvnSonarCmd} clean package sonar:sonar -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"

      }
    }

    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps {
        echo "Publish to Nexus"
        sh "${mvnCmd} deploy"
      }
    }

    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps {
        echo "Building OpenShift container image tasks:${devTag}"

        script {
          openshift.withCluster() {
            openshift.withProject("${devProject}") {
              def bc = openshift.selector("bc", "tasks")
              bc.describe()
              bc.startBuild("--from-file=http://nexus.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war", "--wait=true")
              openshift.selector("bc", "tasks").startBuild("--from-dir=.", "--wait=true")

              echo "Tagging the image"
              openshift.tag("tasks:latest", "tasks:${devTag}")
            }
          }
        }
      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      steps {
        echo "Deploy container image to Development Project"

        script {
          openshift.withCluster() {
            openshift.withProject("${devProject}") {

              echo "Update the image on the dev deployment config"
              openshift.set("image", "dc/tasks", "tasks=image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${devTag}")

              echo "Recreate the config maps"
              openshift.selector('configmap', 'tasks-config').delete()

              def cm = openshift.create('configmap', 'tasks-config', '--from-file=./configuration/application-users.properties', '--from-file=./configuration/application-roles.properties' )
              cm.describe()

              echo "Redeploy the dev deployment"
              def dc = openshift.selector("dc", "tasks")
              dc.rollout().latest()

              timeout (time: 10, unit: 'MINUTES') {
                echo "Waiting for ReplicationController tasks to be ready"
                dc.rollout().status()
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
          def status = "000"

          // Create a new task called "integration_test_1"
          echo "Creating task"
          // The next bit works - but only after the application
          // has been deployed successfully
          status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
          echo "Status: " + status
          if (status != "201") {
            error 'Integration Create Test Failed!'
          }

          echo "Retrieving tasks"
          status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Accept: application/json' -X GET http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
          echo "Status: " + status
          if (status != "200") {
            error 'Integration Retrieving Test Failed!'
          }

          echo "Deleting tasks"
          status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -X DELETE http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
          echo "Status: " + status
          if (status != "204") {
            error 'Integration Deleting Test Failed!'
          }

        }
      }
    }

    // Copy Image to Nexus Container Registry
    stage('Copy Image to Nexus Container Registry') {
      steps {
        echo "Copy image to Nexus Container Registry"

        sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:app_deploy docker://image-registry.openshift-image-registry.svc.cluster.local:5000/${devProject}/tasks:${devTag} docker://nexus-registry.${prefix}-nexus.svc.cluster.local:5000/tasks:${devTag}"

        script {
          openshift.withCluster() {
            openshift.withProject("${prodProject}") {
              echo "Tagging the image"
              openshift.tag("${devProject}/tasks:${devTag}", "${devProject}/tasks:${prodTag}")
            }
          }
        }

      }
    }

    // Blue/Green Deployment into Production
    // -------------------------------------
    // Do not activate the new version yet.
    stage('Blue/Green Production Deployment') {
      steps {
        echo "Blue/Green Deployment"
        script {
          openshift.withCluster() {
            openshift.withProject("${prodProject}") {

              echo "Trying to find currently running app"
              def runningApp = openshift.selector("route", "tasks").object().spec.to.name

              if (runningApp == "tasks-blue") {
                targetApp = "tasks-green"
              } else {
                targetApp = "tasks-blue"
              }

              echo "Currently active application is " + runningApp
              echo "We will change to application " + targetApp

              echo "Update application to " + targetApp
              def dc = openshift.selector("dc/${targetApp}").object()
              dc.spec.template.spec.containers[0].image="image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${prodTag}"
              openshift.apply(dc)

              echo "Recreate Configmap"
              openshift.selector("configmap", "${targetApp}-config").delete()
              def configmap = openshift.create("configmap", "${targetApp}-config", "--from-file=./configuration/application-users.properties", "--from-file=./configuration/application-roles.properties" )

              echo "Redeploy the prod deployment"
              def deploy = openshift.selector("dc", "${targetApp}").rollout().latest();
              deploy.rollout().latest()

              timeout (time: 10, unit: 'MINUTES') {
                echo "Waiting for ReplicationController tasks to be ready"
                deploy.rollout().status()
              }
            }
          }
        }
      }
    }

    stage('Switch over to new Version') {
      steps {
        // TBD: Stop for approval


        echo "Executing production switch"
        // TBD: After approval execute the switch

      }
    }
  }
}