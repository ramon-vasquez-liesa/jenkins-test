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
  }

  stages {
    stage('Prep Agent') {
      steps {
        sh 'docker build -t ramonvasquezliesa/jenkins-test:test .'
      }
    }
  }
}
