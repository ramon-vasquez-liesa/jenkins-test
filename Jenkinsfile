pipeline {
  agent {
    docker {
      image 'ramonvasquezliesa/jenkins-test:test'
    }
  }

  tools { git 'Default' }

  environment {
    REPO_URL        = 'git@github.com:ramon-vasquez-liesa/test-image.git'
    GIT_CREDENTIALS = 'docker-hub-ssh'
    BRANCH          = 'main'
    DOCKERHUB_USER  = 'ramonvasquezliesa'
    DOCKERHUB_PASS  = 'liesatest'
  }

  stages {
    stage('Prep Agent') {
      steps {
        sh 'docker --username $DOCKERHUB_USER --password-stdin <<< $DOCKERHUB_PASS'
        sh 'docker build -t ramonvasquezliesa/jenkins-test:test .'
      }
    }
  }
}
