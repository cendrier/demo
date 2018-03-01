
pipeline {
    agent {
      label "jenkins-maven"
    }

    environment {
      ORG 		        = 'jenkinsx'
      APP_NAME          = 'demo'
      GIT_CREDS         = credentials('jenkins-x-git')
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')

      GIT_USERNAME      = "$GIT_CREDS_USR"
      GIT_API_TOKEN     = "$GIT_CREDS_PSW"
      JOB_NAME          = "$JOB_NAME"
      BUILD_NUMBER      = "$BUILD_NUMBER"
    }

    stages {
      stage('CI Build and push snapshpt') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container('maven') {
            sh "mvn versions:set -DnewVersion=$PREVIEW_VERSION"
            sh "mvn install"
            sh "docker build -f Dockerfile.release -t $JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:$PREVIEW_VERSION ."
            sh "docker push $JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:$PREVIEW_VERSION"
          }

		  // comment out until draft pack includes preview environment charts
          //dir ('./charts/preview') {
          //  container('maven') {
          //    sh "make preview"
          //  }
          //}
        }
      }

      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          container('maven') {
            // ensure we're not on a detached head
            sh "git checkout master"

            // until we switch to the new kubernetes / jenkins credential implementation use git credentials store
            sh "git config --global credential.helper store"

            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            sh "mvn versions:set -DnewVersion=\$(cat VERSION)"
          }

          dir ('./charts/demo') {
            container('maven') {
              sh "make tag"
            }
          }

          container('maven') {
            sh 'mvn clean deploy'
            sh "docker build -f Dockerfile.release -t $JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:\$(cat VERSION) ."
            sh "docker push $JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:\$(cat VERSION)"
            sh 'jx step changelog --version \$(cat VERSION)'
          }
        }
      }

      stage('Promote to Environments') {
        environment {
          GIT_USERNAME = "$GIT_CREDS_USR"
          GIT_API_TOKEN = "$GIT_CREDS_PSW"
        }
        when {
          branch 'master'
        }
        steps {
          dir ('./charts/demo') {
            container('maven') {

              // release the helm chart
              sh 'make release'

              // promote through all 'Auto' promotion Environments
              sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)'
            }
          }
        }
      }

    }
  }
