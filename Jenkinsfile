pipeline {
  agent any

  // run nightly: H 0 * * * (configure on job / multibranch schedule)
  triggers { cron('H 0 * * *') }

  options {
    // keep console timestamps for easier debugging
    timestamps()
    // keep build logs for some time (optional)
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  environment {
    // image name - make sure to use your Docker Hub namespace if needed
    DOCKER_USER = "madhavan2454"
    IMAGE_NAME = "generic-app"
    IMAGE_TAG = "pruned-${env.BUILD_NUMBER}"
    FULL_IMAGE = "${env.DOCKER_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps {
        echo "Cloning repository..."
        // Standard checkout for a Jenkins multibranch or Pipeline job
        checkout scm

        echo "Listing branches (local + remote recent):"
        sh '''
          git fetch --all --prune
          echo "--- local branches sorted by committerdate ---"
          git branch -v --sort=-committerdate || true
          echo "--- remote branches sorted by committerdate ---"
          git branch -r --sort=-committerdate || true
          echo "--- all refs sorted by committerdate ---"
          git for-each-ref --format='%(refname:short) %(committerdate:iso8601)' --sort=-committerdate refs/heads/ || true
        '''
      }
    }

    stage('Git Prune Ops (cleanup local branches older than 7 days)') {
      steps {
        script {
          long start = System.currentTimeMillis()
          try {
            // Limit the prune operation to 2 minutes
            timeout(time: 2, unit: 'MINUTES') {
              sh '''
                set -euo pipefail
                git fetch --all --prune

                # compute cutoff epoch (7 days ago)
                CUTOFF=$(date -d "7 days ago" +%s)

                # list local branches with their commit creation date (unix)
                # NOTE: use creatordate:unix to get a stable epoch time
                git for-each-ref --format='%(refname:short) %(creatordate:unix)' refs/heads/ | \
                  awk -v cutoff="$CUTOFF" '$2 < cutoff { print $1 }' | \
                  tee /tmp/branches-to-delete || true

                if [ -s /tmp/branches-to-delete ]; then
                  echo "Branches to delete:"
                  cat /tmp/branches-to-delete
                  # delete one-by-one to avoid xargs failure if nothing
                  cat /tmp/branches-to-delete | xargs -r -n1 git branch -D || true
                else
                  echo "No local branches older than 7 days."
                fi
              '''
            } // timeout
            long elapsed = System.currentTimeMillis() - start
            echo "Git prune completed in ${elapsed/1000} seconds."
          } catch (org.jenkinsci.plugins.workflow.steps.TimeoutStepExecution.ExceededTimeout e) {
            long elapsed = System.currentTimeMillis() - start
            echo "Prune timed out after ${elapsed/1000} seconds."
            currentBuild.result = 'ABORTED'
            // abort the pipeline now
            error("Aborting build due to prune timeout.")
          } catch (err) {
            echo "Git prune error: ${err}"
            currentBuild.result = 'FAILURE'
            throw err
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        script {
          echo "Building Docker image: ${env.FULL_IMAGE}"
          // ensure Docker daemon available on agent
          sh """
            docker build -t ${env.FULL_IMAGE} .
            docker images --filter=reference='${env.IMAGE_NAME}:*' --format 'table {{.Repository}}\\t{{.Tag}}\\t{{.Size}}'
          """
        }
      }
    }

    stage('Prune and Push') {
      steps {
        script {
          // measure prune duration
          long pruneStart = System.currentTimeMillis()
          try {
            // protect prune with timeout too (prune can take long if diskfull)
            timeout(time: 2, unit: 'MINUTES') {
              sh 'docker image prune -a -f || true'
            }
            long pruneElapsed = System.currentTimeMillis() - pruneStart
            echo "Docker image prune finished in ${pruneElapsed/1000} seconds."
          } catch (org.jenkinsci.plugins.workflow.steps.TimeoutStepExecution.ExceededTimeout e) {
            long pruneElapsed = System.currentTimeMillis() - pruneStart
            echo "Docker prune timed out after ${pruneElapsed/1000} seconds."
            currentBuild.result = 'ABORTED'
            error("Aborting due to docker prune timeout.")
          } catch (err) {
            echo "Prune Error: ${err}"
            currentBuild.result = 'ABORTED'
            error("Aborting due to docker prune error.")
          }

          // Now push the newly built image to Docker Hub
          try {
            withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
              echo "Logging in to Docker Hub as ${env.DOCKER_USER}"
              sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'

              // optionally tag with your Docker Hub namespace (if IMAGE_NAME does not include <user>/)
              // If your IMAGE_NAME is "generic-app", ensure your Docker username is prefixing:

              echo "Pushing image ${remote}"
              sh """
                docker push ${FULL_IMAGE}
                docker logout || true
              """
            }
          } catch (err) {
            echo "Push Error: ${err}"
            currentBuild.result = 'ABORTED'
            error("Aborting due to docker push failure.")
          }
        }
      }
    } // end stage Prune and Push
  } // stages

  post {
    success {
      echo "Pipeline finished SUCCESS - image pushed: ${env.IMAGE_TAG}"
    }
    aborted {
      echo "Pipeline ABORTED - check logs for timeout or errors."
    }
    failure {
      echo "Pipeline FAILED - check logs."
    }
    always {
      // small inventory of local images (helpful when debugging disk usage)
      sh 'echo "Local images snapshot:"; docker images --format "table {{.Repository}}\\t{{.Tag}}\\t{{.Size}}" || true'
    }
  }
}
