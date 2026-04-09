pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'v3', description: 'Тег нового образа (без latest!)')
    }

    environment {
        REGISTRY      = 'localhost:5000'
        IMAGE_NAME    = 'georgezhironkin/quarkus-native'
        DEPLOYMENT    = 'quarkus-app'
        CONTAINER     = 'quarkus-container'
        LOADTESTER_IMAGE = 'loadtester'
        SERVICE_URL      = 'http://quarkus-service/work'

        TARGET_RPS    = '300'
        TEST_DURATION = '2m'
    }

    stages {


        // ========== 2. Сохранить предыдущий стабильный тег ==========
        stage('Save Previous Tag') {
            steps {
                script {
                    env.PREVIOUS_IMAGE = sh(
                        script: """
                            kubectl get deployment ${DEPLOYMENT} \
                                -o jsonpath='{.spec.template.spec.containers[0].image}'
                        """,
                        returnStdout: true
                    ).trim()
                    echo "Previous image: ${env.PREVIOUS_IMAGE}"
                }
            }
        }

        // ========== 3. Обновить Deployment ==========
        stage('Deploy') {
            steps {
                sh """
                    echo "Deploying ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}..."
                    kubectl set image deployment/${DEPLOYMENT} \
                        ${CONTAINER}=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

                    echo "Waiting for rollout..."

                    kubectl wait pod \
                        --for=condition=Ready \
                        -l app=${DEPLOYMENT} \
                        --timeout=120s

                    echo "Current pods:"
                    kubectl get pods -l app=${DEPLOYMENT}
                """
            }
        }

        // ========== 4. Проверить доступность сервиса ==========
        stage('Verify Service') {
            steps {
                sh '''
                    echo "Checking service via cluster..."
                    for i in $(seq 1 10); do
                        STATUS=$(kubectl run curl-check-$i --image=curlimages/curl --rm -i --restart=Never \
                            -- curl -s -o /dev/null -w "%{http_code}" http://quarkus-service/work 2>/dev/null || true)
                        if echo "$STATUS" | grep -q "200"; then
                            echo "Service is UP (HTTP 200)"
                            exit 0
                        fi
                        echo "Attempt $i — retrying in 5s..."
                        kubectl delete pod curl-check-$i --ignore-not-found 2>/dev/null
                        sleep 5
                    done
                    echo "Service not available"
                    exit 1
                '''
            }
        }

        // ========== 5. Нагрузочный тест №1 — прогрев HPA ==========
        stage('Load Test 1 (Warmup)') {
            steps {
                sh '''
                    set -e

                    JOB_NAME="loadtest-warmup-job"

                    kubectl delete job ${JOB_NAME} --ignore-not-found=true

                    kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: ${JOB_NAME}
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: loadtester
          image: ${LOADTESTER_IMAGE}
          args:
            - "-url"
            - "${SERVICE_URL}"
            - "-rps"
            - "${TARGET_RPS}"
            - "-duration"
            - "${TEST_DURATION}"
EOF


                    if ! kubectl wait --for=condition=Complete --timeout=5m job/${JOB_NAME}; then
                        echo "Warmup job failed or timed out"
                        kubectl describe job ${JOB_NAME} || true
                        kubectl logs job/${JOB_NAME} || true
                        exit 1
                    fi

                    kubectl logs job/${JOB_NAME} | tee loadtest1.txt

                    echo '--- HPA status after warmup ---'
                    kubectl get hpa quarkus-hpa
                    kubectl get pods -l app=quarkus-app

                    echo 'Waiting 30s for HPA scale-up...'
                    sleep 30

                    kubectl get hpa quarkus-hpa
                    kubectl get pods -l app=quarkus-app
                '''
            }
        }

        // ========== 6. Нагрузочный тест №2 — решающий ==========
        stage('Load Test 2 (Deciding)') {
            steps {
            sh '''
                set -e

                JOB_NAME="loadtest-deciding-${BUILD_NUMBER}"

                kubectl delete job ${JOB_NAME} --ignore-not-found=true

                kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: ${JOB_NAME}
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: loadtester
          image: ${LOADTESTER_IMAGE}
          args:
            - "-url"
            - "${SERVICE_URL}"
            - "-rps"
            - "${TARGET_RPS}"
            - "-duration"
            - "${TEST_DURATION}"
EOF


                if ! kubectl wait --for=condition=Complete --timeout=5m job/${JOB_NAME}; then
                    echo "Deciding job failed or timed out"
                    kubectl describe job ${JOB_NAME} || true
                    kubectl logs job/${JOB_NAME} || true
                    exit 1
                fi

                kubectl logs job/${JOB_NAME} | tee loadtest2.txt
            '''
        }
        }
    

        // ========== 7. Анализ результатов ==========
        stage('Analyze Results') {
            steps {
                script {
                    def output = readFile('loadtest2.txt').trim()
                    echo "=== Load Test 2 Output ===\n${output}"

                    def lines = output.split('\n')
                    def successRate = 0.0
                    def successRPS = 0.0

                    for (line in lines) {
                        def successMatch = (line =~ /Процент успешных:\s+(\d+[\.,]?\d*)%/)
                        if (successMatch.find()) {
                            successRate = successMatch.group(1).replace(',', '.').toDouble()
                        }
                        def rpsMatch = (line =~ /Успешный RPS:\s+(\d+[\.,]?\d*)\s+req\/s/)
                        if (rpsMatch.find()) {
                            successRPS = rpsMatch.group(1).replace(',', '.').toDouble()
                        }
                    }

                    def targetRPS = env.TARGET_RPS.toDouble()

                    echo "Процент успешных: ${successRate}%"
                    echo "Успешный RPS: ${successRPS} req/s (target: ${targetRPS})"

                    if (successRate < 95 || successRPS < targetRPS * 0.9) {
                        echo ">>> RELEASE FAILED: success=${successRate}%, rps=${successRPS} <<<"
                        env.RELEASE_OK = 'false'
                    } else {
                        echo ">>> RELEASE PASSED: success=${successRate}%, rps=${successRPS} <<<"
                        env.RELEASE_OK = 'true'
                    }
                }
            }
        }

        // ========== 8. Откат при неуспехе ==========
        stage('Rollback') {
            when {
                expression { env.RELEASE_OK == 'false' }
            }
            steps {
                sh """
                    echo ">>> Rolling back to: ${PREVIOUS_IMAGE} <<<"

                    kubectl set image deployment/${DEPLOYMENT} \
                        ${CONTAINER}=${PREVIOUS_IMAGE}

                    kubectl rollout status deployment/${DEPLOYMENT} --timeout=300s

                    kubectl wait pod \
                        --for=condition=Ready \
                        -l app=${DEPLOYMENT} \
                        --timeout=120s

                    # Проверка: image откатился
                    CURRENT=\$(kubectl get deployment ${DEPLOYMENT} \
                        -o jsonpath='{.spec.template.spec.containers[0].image}')
                    echo "Image after rollback: \$CURRENT"

                    if [ "\$CURRENT" != "${PREVIOUS_IMAGE}" ]; then
                        echo "ERROR: Rollback verification failed!"
                        exit 1
                    fi

                    # Проверка: сервис отвечает
                    STATUS=\$(kubectl run curl-rollback --image=curlimages/curl --rm -i --restart=Never \
                        -- curl -s -o /dev/null -w "%{http_code}" http://quarkus-service/work 2>/dev/null || true)
                    if echo "\$STATUS" | grep -q "200"; then
                        echo "Service OK after rollback"
                    else
                        kubectl delete pod curl-rollback --ignore-not-found 2>/dev/null
                        echo "WARNING: Service check returned: \$STATUS"
                    fi
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'loadtest*.txt', allowEmptyArchive: true

            sh """
                echo '=== Final cluster state ==='
                kubectl get all -l app=quarkus-app
                kubectl get hpa quarkus-hpa
                kubectl get ingress quarkus-ingress
            """
        }
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline FAILED — check logs and artifacts.'
        }
    }
}
