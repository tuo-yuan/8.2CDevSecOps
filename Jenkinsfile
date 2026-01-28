pipeline {
    agent any

    environment {
        PATH = '/opt/homebrew/bin:/usr/local/bin:/bin:/usr/bin:/sbin:/usr/sbin'
        IMAGE_NAME = 'goof-app'
    }

    stages {
        // Stage 1: Build
        stage('Build') {
      steps {
        script {
          env.GIT_COMMIT_SHORT = env.GIT_COMMIT.take(7)
        }
        sh '''
                    docker build -t ${IMAGE_NAME}:${GIT_COMMIT_SHORT} -t ${IMAGE_NAME}:latest .
                '''
      }
        }

        // Stage 2: Test
        stage('Test') {
      steps {
        sh '''
                    export GIT_COMMIT="${GIT_COMMIT_SHORT}"

                    # Start app with MongoDB for testing
                    docker compose -f docker-compose.staging.yml up -d

                    sleep 10

                    # Run integration test - verify app responds
                    if curl -sS -o /dev/null -w "%{http_code}" http://127.0.0.1:3001 | grep -q "200"; then
                      echo "Integration test PASSED - HTTP 200 OK"
                      docker compose -f docker-compose.staging.yml down || true
                      exit 0
                    fi

                    docker compose -f docker-compose.staging.yml logs
                    docker compose -f docker-compose.staging.yml down || true
                    exit 1
                '''
      }
        }

        // Stage 3: Code Quality
        stage('Code Quality') {
      steps {
        withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
          sh '''
                        rm -rf sonar-scanner-*
                        curl -sSLo sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-8.0.1.6346-macosx-aarch64.zip
                        unzip -q sonar-scanner.zip
                        ./sonar-scanner-*/bin/sonar-scanner -Dsonar.token=$SONAR_TOKEN
                    '''
        }
      }
        }

        // Stage 4: Security
        stage('Security') {
      steps {
        sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:0.50.4 \
                        image --scanners vuln \
                        --skip-dirs /usr/src/goof/node_modules/.cache \
                        --severity HIGH,CRITICAL \
                         ${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                '''
      }
        }

        // Stage 5: Deploy to Staging
        stage('Deploy to Staging') {
      steps {
        sh '''
                    export GIT_COMMIT="${GIT_COMMIT_SHORT}"
                    docker compose -f docker-compose.staging.yml down --remove-orphans || true
                    docker compose -f docker-compose.staging.yml up -d
                    sleep 10
                    curl -sS http://127.0.0.1:3001 || true
                '''
      }
        }

        // Stage 6: Release to Production
        stage('Release') {
      steps {
        sh """
                    git config user.name "jenkins"
                    git config user.email "jenkins@local"
                    git tag -a release-${GIT_COMMIT_SHORT} -m "Release ${GIT_COMMIT_SHORT}" || true
                    git push origin release-${GIT_COMMIT_SHORT} || true

                    export GIT_COMMIT="${GIT_COMMIT_SHORT}"
                    docker compose -f docker-compose.prod.yml down --remove-orphans || true
                    docker compose -f docker-compose.prod.yml up -d
                    sleep 10
                    curl -sS http://127.0.0.1:3002 || true
                """
      }
        }

        // Stage 7: Monitoring
        stage('Monitoring') {
      steps {
        sh '''
                    docker compose -f docker-compose.monitoring.yml up -d
                    sleep 5
                    curl -sS http://127.0.0.1:9090/-/healthy
                    echo "Prometheus Dashboard: http://localhost:9090"
                '''
      }
        }
    }

    post {
        always {
      echo "Pipeline finished: ${currentBuild.currentResult}"
        }
    }
}
