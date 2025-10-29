pipeline {
  agent any

  // Run daily at midnight
  triggers {
    cron('H 0 * * *')
  }

  environment {
    IMAGE_NAME = "generic-app"
    IMAGE_TAG  = "pruned-${BUILD_NUMBER}"
    DOCKERHUB_CREDENTIALS = credentials('docker-cred')
    DOCKER_USER = "madhavan2454"
  }

  stages {

    /* ───────────────────────────────
       1️⃣ Checkout - Clone and list branches
    ─────────────────────────────── */
    stage('Checkout') {
      steps {
        echo "Cloning repository..."
        git branch: 'main', url: 'https://github.com/madhavan2454/star-agile-insurance-project.git'

        sh '''
          git fetch --all --prune
          echo "Branches sorted by latest commit:"
          git branch -a --sort=-committerdate || true
        '''
      }
    }

    stage('Run Test Cases'){
      steps {
        echo "Running test cases"
        sh 'mvn clean package'
      }
    }

    stage('Git Prune Ops') {
      steps {
        script {
          echo "Cleaning up local branches older than 7 days..."
          try {
            timeout(time: 2, unit: 'MINUTES') {
              sh '''
                set -e
                CUTOFF=$(date -d "10 minutes ago" +%s)
                git for-each-ref --format='%(refname:short) %(creatordate:unix)' refs/heads/ 
                | awk -v c=$CUTOFF '$2 < c {print $1}' \
                | grep -Ev '^ (main|master|develop|release/)' \
                | xargs -r git branch -D
              '''
            }
          } catch (err) {
            echo "Branch prune timed out or failed: ${err}"
            currentBuild.result = 'ABORTED'
            error("Stopping pipeline due to prune issue.")
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
        sh '''
          docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} .
          docker images ${IMAGE_NAME}
        '''
      }
    }

    /* ───────────────────────────────
       4️⃣ Prune & Push - Cleanup + Push to Docker Hub
    ─────────────────────────────── */
    stage('Prune and Push') {
      steps {
        script {
          // Clean up old images
          

          // Push to Docker Hub
            try {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}'
            } catch (err) {
              echo "Docker push failed: ${err}"
              currentBuild.result = 'ABORTED'
              error("Stopping pipeline due to push failure.")
            }

            try {
            timeout(time: 2, unit: 'MINUTES') {
              sh 'docker image prune -f || true'
            }
          } catch (err) {
            echo "Docker prune timed out or failed."
            currentBuild.result = 'ABORTED'
            error("Stopping pipeline due to prune issue.")
          }
        }
      }
    }
  }

  post {
    always {
      sh 'docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" || true'
    }
    success {
      echo "✅ Pipeline completed successfully."
    }
    aborted {
      echo "⚠️  Pipeline aborted (timeout or push error)."
    }
    failure {
      echo "❌ Pipeline failed."
    }
  }
}
