// Run this pipeline on the custom Maven Slave ('maven-appdev')
// Maven Slaves have JDK and Maven already installed
// 'maven-appdev' has skopeo installed as well.

node('maven-appdev') {
  // Define Maven Command. Make sure it points to the correct
  // settings for our Nexus installation (use the service to
  // bypass the router). The file nexus_openshift_settings.xml
  // needs to be in the Source Code repository.
  def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

  // Checkout Source Code
  stage('Checkout Source') {
   git credentialsId: 'gogs', url: 'http://gogs-rn-gogs.apps.dfw2.example.opentlc.com/CICDLabs/openshift-tasks-private.git'
  }
  
  // The following variables need to be defined at the top level
  // and not inside the scope of a stage - otherwise they would not
  // be accessible from other stages.
  // Extract version and other properties from the pom.xml
  def groupId    = getGroupIdFromPom("pom.xml")
  def artifactId = getArtifactIdFromPom("pom.xml")
  def version    = getVersionFromPom("pom.xml")

  // Set the tag for the development image: version + build number
  def devTag  = "${version}-${BUILD_NUMBER}"
  // Set the tag for the production image: version
  def prodTag = "${version}"

  stage('Build war') {
    echo "Building version ${version}"
     echo "Building version ${devTag}"
     echo "____________________________________________________________"
	 echo "			BUILDING VERSION 	                               "
	 echo "____________________________________________________________"
	 
	sh "${mvnCmd} clean package -DskipTests"
  }
  
  // Using Maven run the unit tests
  stage('Unit Tests') {
     echo "____________________________________________________________"
	 echo "			MAVEN TEST                                         "
	 echo "____________________________________________________________"
    //sh "${mvnCmd} test"
  }

  // Using Maven call SonarQube for Code Analysis
  stage('Code Analysis') {
    echo "Running Code Analysis"
	 echo "____________________________________________________________"
	 echo "			SONAR QUBE TEST                                    "
	 echo "____________________________________________________________"
	//sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.rn-sonar.svc:9000 -Dsonar.projectName=${JOB_BASE_NAME}-${devTag}"
  }

  // Publish the built war file to Nexus
  stage('Publish to Nexus') {
    echo "Publish to Nexus"
	 echo "____________________________________________________________"
	 echo "			PUBLISH TO NEXUS                                   "
	 echo "____________________________________________________________"
    sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.nick-nexus.svc:8081/repository/releases"
  }

  
  stage('Build and Tag OpenShift Image') {
	  echo "Building OpenShift container image tasks:${devTag}"
	  echo "____________________________________________________________"
	  echo "			Build Openshift Image & Tag It                    "
	  echo "____________________________________________________________"
		  sh "oc start-build tasks --follow --from-file=./target/openshift-tasks.war -n rn-tasks-dev"
	//Tag the image using the devTag
		openshiftTag alias: 'false', destStream: 'tasks', destTag: devTag, destinationNamespace: 'rn-tasks-dev', namespace: 'rn-tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
  }

  stage('Deploy to Dev') {
    echo "Deploying container image to Development Project"
	  echo "____________________________________________________________"
	  echo "			Deploy Container Image to Dev Environment                        "
	  echo "____________________________________________________________"
    // Update the Image on the Development Deployment Config
	    sh "oc set image dc/tasks tasks=docker-registry.default.svc:5000/rn-tasks-dev/tasks:${devTag} -n rn-tasks-dev"

    // Update the Config Map which contains the users for the Tasks application
	    sh "oc delete configmap tasks-config -n rn-tasks-dev --ignore-not-found=true"
	    sh "oc create configmap tasks-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n rn-tasks-dev"

    // Deploy the development application.
    // Replace xyz-tasks-dev with the name of your production project
        openshiftDeploy depCfg: 'tasks', namespace: 'rn-tasks-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'
        openshiftVerifyDeployment depCfg: 'tasks', namespace: 'rn-tasks-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
        openshiftVerifyService namespace: 'rn-tasks-dev', svcName: 'tasks', verbose: 'false'
    }

  stage('Integration Tests') {
    echo "Running Integration Tests"
    echo "____________________________________________________________"
	  echo "		Run Integration Tests in Development Environment        "
	  echo "____________________________________________________________"
      echo "Running Integration Tests"
    sleep 15

    // Create a new task called "integration_test_1"
      echo "Creating task"
      sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.rn-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1"

    // Retrieve task with id "1"
      echo "Retrieving tasks"
      sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X GET http://tasks.rn-tasks-dev.svc.cluster.local:8080/ws/tasks/1"

    // Delete task with id "1"
      echo "Deleting tasks"
      sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X DELETE http://tasks.rn-tasks-dev.svc.cluster.local:8080/ws/tasks/1"
  }


  stage('Copy Image to Nexus Docker Registry') {
    echo "Copy image to Nexus Docker Registry"
    echo "____________________________________________________________"
	  echo "		Copy Image to Nexus Docker Registry                     "
	  echo "____________________________________________________________"
    sh "oc login -u bob.nicolais-perficient.com -p Riggst2007! https://master.dfw2.example.opentlc.com --insecure-skip-tls-verify=true"
    sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry.default.svc:5000/rn-tasks-dev/tasks:${devTag} docker://nexus-registry.xyz-nexus.svc.cluster.local:5000/tasks:${devTag}"

  // Tag the built image with the production tag.
  // Replace xyz-tasks-dev with the name of your dev project
    openshiftTag alias: 'false', destStream: 'tasks', destTag: prodTag, destinationNamespace: 'rn-tasks-dev', namespace: 'rn-tasks-dev', srcStream: 'tasks', srcTag: devTag, verbose: 'false'
  }

  // Blue/Green Deployment into Production
  // -------------------------------------
  // Do not activate the new version yet.
  def destApp   = "tasks-green"
  def activeApp = ""

  stage('Blue/Green Production Deployment') {
    echo "____________________________________________________________"
	  echo "		Blue Green Prodduction Deployment                       "
	  echo "____________________________________________________________"
      echo "Copy image to Nexus Docker Registry"

      // Replace xyz-tasks-dev and xyz-tasks-prod with your project names
      activeApp = sh(returnStdout: true, script: "oc get route tasks -n rn-tasks-prod -o jsonpath='{ .spec.to.name }'").trim()
      if (activeApp == "tasks-green") {
        destApp = "tasks-blue"
      }

      echo "Active Application:      " + activeApp
      echo "Destination Application: " + destApp

      // Update the Image on the Production Deployment Config
        sh "oc set image dc/${destApp} ${destApp}=docker-registry.default.svc:5000/rn-tasks-dev/tasks:${prodTag} -n rn-tasks-prod"

      // Update the Config Map which contains the users for the Tasks application
       sh "oc delete configmap ${destApp}-config -n rn-tasks-prod --ignore-not-found=true"
       sh "oc create configmap ${destApp}-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n rn-tasks-prod"

      // Deploy the inactive application.
      // Replace rn-tasks-prod with the name of your production project
       sh "oc policy add-role-to-user admin -z jenkins -n rn-tasks-prod"
       openshiftDeploy depCfg: destApp, namespace: 'rn-tasks-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
       openshiftVerifyDeployment depCfg: destApp, namespace: 'rn-tasks-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
       openshiftVerifyService namespace: 'rn-tasks-prod', svcName: destApp, verbose: 'false'
  }

  stage('Switch over to new Version') {
    // TBD
    echo "Switching Production application to ${destApp}."
    // TBD
  }
}

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}
