@Library('gstjenkinslib') _

pipeline {

 agent {
    kubernetes {
      inheritFrom 'jenkins-pipeline'
      label 'jenkins-pipeline'
      defaultContainer 'mvn-jdk25'
    }
  }

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: env.BRANCH_NAME == 'master' ? '5' : '1'))
  }

  environment {
    RELEASE_BRANCH = 'master'
    BUILD_TYPE = 'RELEASE'
  }

  stages {
    stage('Analyze Dependencies') {
      steps {
        setEnvProperties()
        mavenDependencyVersionAnalysis()
      }
    }

    stage('Compile') {
      steps {
        mavenCompile()
      }
    }

    stage('Test') {
      steps {
        withCredentials([file(credentialsId: 'maven-credentials', variable: 'MVN_SETTINGS')]) {
          sh 'mvn --settings $MVN_SETTINGS test'
        }
      }
      post {
        always {
          testReportsHTML()
        }
      }
    }

    stage('Analyze Coverage') {
        steps {
            discoverGitReferenceBuild()
            recordCoverage(tools: [[parser: 'JACOCO']],
                checksAnnotationScope: 'MODIFIED_LINES',
                sourceCodeRetention: 'MODIFIED',
                sourceDirectories: [
                        [path: 'apiphany-core/src/main/java'],
                        [path: 'apiphany-httpclient5/src/main/java'],
                        [path: 'apiphany-okhttp/src/main/java'],
                        [path: 'apiphany-spring/src/main/java'],
                        [path: 'apiphany-core/target/generated-sources/annotations'],
                        [path: 'apiphany-httpclient5/target/generated-sources/annotations'],
                        [path: 'apiphany-okhttp/target/generated-sources/annotations'],
                        [path: 'apiphany-spring/target/generated-sources/annotations']],
                qualityGates: [
                        [threshold: 28.0, metric: 'LINE', baseline: 'PROJECT', criticality: 'UNSTABLE'],
                        [threshold: 13.0, metric: 'BRANCH', baseline: 'PROJECT', criticality: 'UNSTABLE']])
        }
    }

//     stage('Code analysis') {
//       steps {
//         mavenCodeAnalysis()
//       }
//     }

    stage('Determine version') {
      when { expression { env.BRANCH_NAME == env.RELEASE_BRANCH } }

      steps {
        withCredentials([file(credentialsId: 'maven-credentials', variable: 'MVN_SETTINGS')]) {
          script {
            // read the project version from the Maven pom file
            // note that using readMavenPom step is not recommended
            // see https://jenkins.io/doc/pipeline/steps/pipeline-utility-steps/#readmavenpom-read-a-maven-project-file)
            def projectVersion = sh script: 'mvn --settings $MVN_SETTINGS help:evaluate -Dexpression=project.version -q -DforceStdout', returnStdout: true
            // Build version includes the date time formatter
            BUILD_DATE_FORMATTED = java.time.LocalDateTime.now().format(java.time.format.DateTimeFormatter.ofPattern("yyyyMMddHHmmSS"))
            env.RELEASE_VERSION = "${projectVersion}-${BUILD_DATE_FORMATTED}-${env.BUILD_TYPE}"
            echo "Build version ${RELEASE_VERSION}"
          }
        }
      }
    }

    stage('Install library') {
      when { expression { env.BRANCH_NAME == env.RELEASE_BRANCH } }

      steps {
        withCredentials([file(credentialsId: 'maven-credentials', variable: 'MVN_SETTINGS')]) {
            script {
               MAVEN_DEPLOY_REPO = 'azure-maven-repo-release'
               sh 'mvn --settings $MVN_SETTINGS versions:set -DnewVersion=${RELEASE_VERSION} -DprocessAllModules=true -DgenerateBackupPoms=false'
               sh "mvn --settings $MVN_SETTINGS deploy -DskipTests=true -DrepositoryId=$MAVEN_DEPLOY_REPO -pl .,apiphany-test,apiphany-core,apiphany-httpclient5,apiphany-okhttp,apiphany-spring -am"
               appendToBuildDescription("${RELEASE_VERSION}")
            }
        }
      }
    }
  }
}
