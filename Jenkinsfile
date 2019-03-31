pipeline {
  agent {
    label "jenkins-gradle"
  }
  environment {
    ORG = 'emcgowan'
    APP_NAME = 'jm15'
    CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
  }
  stages {
    stage('CI Build and push snapshot') {
      when {
        branch 'PR-*'
      }
      environment {
        PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
        PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
        HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
      }
      steps {
        container('gradle') {
          sh "gradle clean build"
          sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml"
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          dir('./charts/preview') {
            sh "make preview"
            sh "jx preview --app $APP_NAME --dir ../.."
            sh 'echo FROM $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION > dockerfile'
            sh 'echo ADD https://get.aquasec.com/microscanner / >> dockerfile'
            sh 'echo RUN chmod +x /microscanner >> dockerfile' 
            sh 'echo ARG token  >> dockerfile'
            sh 'echo RUN /microscanner "\$"{token} --no-verify --full-output >> dockerfile'
            sh 'echo RUN echo No vulnerabilities!  >> dockerfile'
            sh 'docker build --build-arg=token=NjUxMTUxZDA2MGFi --no-cache --network=host . '
          }
        }
      }
    }
    stage('Build Release') {
      when {
        branch 'master'
      }
      steps {
        container('gradle') {

          // ensure we're not on a detached head
          sh "git checkout master"
          sh "git config --global credential.helper store"
          sh "jx step git credentials"

          // so we can retrieve the version in later steps
          sh "echo \$(jx-release-version) > VERSION"
          sh "jx step tag --version \$(cat VERSION)"
          sh "gradle clean build"
          sh "export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml"
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
        }
      }
    }
    stage('Promote to Environments') {
      when {
        branch 'master'
      }
      steps {
        container('gradle') {
          dir('./charts/jm15') {
            sh "jx step changelog --version v\$(cat ../../VERSION)"

            // release the helm chart
            sh "jx step helm release"

            // promote through all 'Auto' promotion Environments
            sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
          }
        }
      }
    }
  }
  post {
        always {
          cleanWs()
        }
  }
}
