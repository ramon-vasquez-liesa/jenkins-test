pipeline {
  agent {
    docker {
      image 'docker/compose:1.29.2'
      args '''
        -v /var/run/docker.sock:/var/run/docker.sock
        -u root:root
        --entrypoint=""
      '''.stripIndent()
    }
  }

  tools { git 'Default' }

  environment {
    REPO_URL        = 'git@github.com:ramon-vasquez-liesa/doodba-v18.git'
    GIT_CREDENTIALS = 'doodba-v18-ssh'
    BRANCH          = 'main'

    NETWORK_NAME    = 'doodba-net'
    POSTGRES_IMAGE  = 'postgres:16'
    ODOO_IMAGE      = 'odoo:18.0'
    DB_USER         = 'odoo'
    DB_PASSWORD     = 'odoo'
    DB_NAME         = 'devel'
    DB_PORT         = '5432'

    DB_CONTAINER    = "odoo-db-${BUILD_ID}"
    ODOO_CONTAINER  = "odoo18-${BUILD_ID}"
    HOST_PORT       = '8069'
  }

  stages {
    stage('Prep Agent') {
      steps {
        // Install SSH client & ssh-agent
        sh '''
          if command -v apk >/dev/null; then
            apk add --no-cache openssh-client
          elif command -v apt-get >/dev/null; then
            apt-get update && apt-get install -y openssh-client
          fi
        '''
      }
    }

    stage('Cleanup Previous Containers') {
      steps {
        sh 'docker rm -f $DB_CONTAINER $ODOO_CONTAINER || true'
        sh 'docker ps -q --filter "publish=$HOST_PORT" | xargs -r docker rm -f || true'
        sh 'docker network rm $NETWORK_NAME || true'
      }
    }

    stage('Checkout') {
      steps {
        sshagent(credentials: [env.GIT_CREDENTIALS]) {
          checkout([
            $class: 'GitSCM',
            branches: [[name: env.BRANCH]],
            userRemoteConfigs: [[
              url:           env.REPO_URL,
              credentialsId: env.GIT_CREDENTIALS
            ]]
          ])
        }
      }
    }

    stage('Install Python Tools') {
      steps {
        script {
          docker.image('python:3.11-slim').inside(
            '--network host ' +
            '-v /var/run/docker.sock:/var/run/docker.sock ' +
            '-u root:root'
          ) {
            sh '''
              python3 -m venv .venv
              . .venv/bin/activate
              pip install --upgrade pip
              pip install copier invoke pre-commit
            '''
          }
        }
      }
    }

    stage('Start & Wait for PostgreSQL') {
      steps {
        sh "docker network inspect $NETWORK_NAME >/dev/null 2>&1 || docker network create $NETWORK_NAME"
        sh 'docker rm -f $DB_CONTAINER || true'
        sh """
          docker run -d --rm \
            --name $DB_CONTAINER \
            --network $NETWORK_NAME \
            --network-alias db \
            -e POSTGRES_USER=$DB_USER \
            -e POSTGRES_PASSWORD=$DB_PASSWORD \
            -e POSTGRES_DB=$DB_NAME \
            $POSTGRES_IMAGE
        """
      }
    }

    stage('Launch Odoo 18') {
      steps {
        sh """
      # Start the DB
      docker-compose up -d db

      # Wait for readiness…
      for i in \$(seq 1 30); do
        if docker-compose exec -T db pg_isready -h db -U ${DB_USER} >/dev/null; then
          echo "Postgres is up!"
          break
        fi
        echo "Waiting for Postgres (\$i/30)…"
        sleep 2
      done

      # Drop & recreate the DB
      echo \"DROP DATABASE IF EXISTS ${DB_NAME};\" | docker-compose exec -u postgres -T db psql -U odoo -d postgres\
      && echo \"CREATE DATABASE ${DB_NAME};\" | docker-compose exec -u postgres -T db psql -U odoo -d postgres
    """
      }
    }

    stage('Install Base Module') {
      steps {
        sh 'docker ps -q --filter "publish=$HOST_PORT" | xargs -r docker rm -f || true'
        sh """
          docker run -d --rm \
          --name $ODOO_CONTAINER \
          --network $NETWORK_NAME \
          -p 8069:8069 \
          -e HOST=db \
          -e USER=$DB_USER \
          -e PASSWORD=$DB_PASSWORD \
          $ODOO_IMAGE
        """
        sh """
          docker exec $ODOO_CONTAINER \
            odoo --stop-after-init \
                 --db_host=db \
                 --db_port=$DB_PORT \
                 --db_user=$DB_USER \
                 --db_password=$DB_PASSWORD \
                 -d $DB_NAME \
                 -i base --without-demo=all
        """
      }
    }
  }

  post {
    failure {
      sh 'docker rm -f $DB_CONTAINER $ODOO_CONTAINER || true'
    }
  }
}
