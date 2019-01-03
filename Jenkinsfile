openshift.withCluster() {
  env.NAMESPACE = openshift.project()
  env.POM_FILE = env.BUILD_CONTEXT_DIR ? "${env.BUILD_CONTEXT_DIR}/pom.xml" : "pom.xml"
  env.APP_NAME = "${JOB_NAME}".replaceAll(/${NAMESPACE}-*/, '').replaceAll(/-?pipeline-?/,'')
  env.BUILD = "${env.NAMESPACE}"
  def PROJECT_BASE = "${env.NAMESPACE}".replaceAll(/-build/, '')
  env.CI = "${PROJECT_BASE}-ci"
  env.TEST = "${PROJECT_BASE}-test"
  env.STAGE = "${PROJECT_BASE}-stage"
  echo "Starting Pipeline for ${APP_NAME}..."
}


library identifier: "devops-pipeline-library@master",
retriever: modernSCM(
  [
    $class: "GitSCMSource",
    remote: "ssh://git@bitbucketdev.ca.bestbuy.com:7999/devops/openshift-pipeline-library.git",
    credentialsId: "${NAMESPACE}-devops-pipeline-library"
  ]
)



pipeline {
  // Use Jenkins Maven slave
  // Jenkins will dynamically provision this as OpenShift Pod
  // All the stages and steps of this Pipeline will be executed on this Pod
  // After Pipeline completes the Pod is killed so every run will have clean
  // workspace
  agent {
    label 'maven'
  }

  // Pipeline Stages start here
  // Requeres at least one stage
  stages {

    // Checkout source code
    // This is required as Pipeline code is originally checkedout to
    // Jenkins Master but this will also pull this same code to this slave
    stage('Git Checkout') {
      steps {
        // Turn off Git's SSL cert check, uncomment if needed
        // sh 'git config --global http.sslVerify false'
        checkout scm
      }
    }

    // Run Maven build, skipping tests
    stage('Build'){
      steps {
        sh "mvn clean install -DskipTests=true -f ${POM_FILE}"
      }
    }
    
    // Run Dependency Scan for packages
    stage('EIP Dependency Scan'){
      steps {
        dependencyPkgsAnalysis(pathToApp: env.POM_FILE)
      }
    }

    // Run Maven unit tests
    stage('Unit Test'){
      steps {
        sh "mvn test -f ${POM_FILE}"
      }
    }
    
    
    // Run Static Code Analysis using sonar
    stage('Sonar Code Analysis'){
      steps {
        sonarStaticCodeAnalysis(pomFile: "${POM_FILE}")
      }
    }

    // Build Container Image using the artifacts produced in previous stages
    stage('Build Container Image'){
      steps {
        // Copy the resulting artifacts into common directory
        sh """
          ls target/*
          rm -rf oc-build && mkdir -p oc-build/deployments
          for t in \$(echo "jar;war;ear" | tr ";" "\\n"); do
            cp -rfv ./target/*.\$t oc-build/deployments/ 2> /dev/null || echo "No \$t files"
          done
        """

        // Build container image using local Openshift cluster
        // Giving all the artifacts to OpenShift Binary Build
        // This places your artifacts into right location inside your S2I image
        // if the S2I image supports it.
        binaryBuild(projectName: env.BUILD, buildConfigName: env.APP_NAME, artifactsDirectoryName: "oc-build")
      }
    }

    stage('Promote from Build to CI') {
      steps {
        tagImage(sourceImageName: env.APP_NAME, sourceImagePath: env.BUILD, toImagePath: env.CI)
      }
    }
    
    stage('Deploy to CI') {
      steps {
        rolloutDeployment(projectName: env.CI, targetApp: env.APP_NAME)
      }
    }

    stage ('Verify Deployment to CI') {
      steps {
        verifyDeployment(projectName: env.CI, targetApp: env.APP_NAME)
      }
    }

    stage('Promote from CI to Test') {
      steps {
        tagImage(sourceImageName: env.APP_NAME, sourceImagePath: env.CI, toImagePath: env.TEST)
      }
    }
    
    stage('Deploy to Test') {
      steps {
        rolloutDeployment(projectName: env.TEST, targetApp: env.APP_NAME)
      }
    }

    stage ('Verify Deployment to Test') {
      steps {
        verifyDeployment(projectName: env.TEST, targetApp: env.APP_NAME)
      }
    }

    stage('Promotion gate') {
      steps {
        script {
          input message: 'Promote application to Staging in Production?'
        }
      }
    }

    stage('Promote from Test to Stage') {
      steps {
        tagImage(sourceImageName: env.APP_NAME, sourceImagePath: env.TEST, toImagePath: env.STAGE)
      }
    }

    stage ('Verify Deployment to Stage') {
      steps {
        verifyDeployment(projectName: env.STAGE, targetApp: env.APP_NAME)
      }
    }
  }
}

