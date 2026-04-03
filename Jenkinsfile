pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', description: 'Тег нового образа (без latest!)')
    }

    environment {
        REGISTRY      = 'localhost:5000'
        IMAGE_NAME    = 'georgezhironkin/quarkus-native:v3'
        DEPLOYMENT    = 'quarkus-app'
        CONTAINER     = 'quarkus-container'
        SERVICE_URL   = 'http://work.local/work'
        TARGET_RPS    = '50'
        TEST_DURATION = '2m'
    }

    stages {

        // ========== 1. Проверить, что образ опубликован ==========
        stage('Verify Image') {
            steps {
                sh """
                    echo "Checking image ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}..."
                    RESULT=\$(curl -s -o /dev/null -w "%{http_code}" \
                        http://${REGISTRY}/v2/${IMAGE_NAME}/manifests/${IMAGE_TAG} \
                        -H "Accept: application/vnd.docker.distribution.manifest.v2+json")
                    if [ "\$RESULT" != "200" ]; then
                        echo "ERROR: Image not found in registry (HTTP \$RESULT)"
                        exit 1
                    fi
                    echo "Image verified OK"
                """
            }
        }

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
                    kubectl rollout status deployment/${DEPLOYMENT} --timeout=300s

                    echo "Waiting for pods Ready..."
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
                    echo "Checking service..."
                    for i in $(seq 1 10); do
                        STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://work.local/work || true)
                        if [ "$STATUS" = "200" ]; then
                            echo "Service is UP (HTTP 200)"
                            exit 0
                        fi
                        echo "Attempt $i: HTTP $STATUS — retrying in 5s..."
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
                sh """
                    echo '=== Load Test 1: HPA warmup ==='

                    docker run --rm --network host loadtester \
                        -url ${SERVICE_URL} \
                        -rps ${TARGET_RPS} \
                        -duration ${TEST_DURATION} \
                        | tee loadtest1.txt || true

                    echo '--- HPA status after warmup ---'
                    kubectl get hpa quarkus-hpa
                    kubectl get pods -l app=quarkus-app

                    echo 'Waiting 30s for HPA scale-up...'
                    sleep 30

                    kubectl get hpa quarkus-hpa
                    kubectl get pods -l app=quarkus-app
                """
            }
        }

        // ========== 6. Нагрузочный тест №2 — решающий ==========
        stage('Load Test 2 (Deciding)') {
            steps {
                sh """
                    echo '=== Load Test 2: Deciding run ==='

                    docker run --rm --network host loadtester \
                        -url ${SERVICE_URL} \
                        -rps ${TARGET_RPS} \
                        -duration ${TEST_DURATION} \
                        | tee loadtest2.txt
                """
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
                        // "Процент успешных: 100.0%"
                        def successMatch = (line =~ /Процент успешных:\s+(\d+[\.,]?\d*)%/)
                        if (successMatch.find()) {
                            successRate = successMatch.group(1).replace(',', '.').toDouble()
                        }
                        // "Успешный RPS: 50.00 req/s"
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
                    for i in \$(seq 1 10); do
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://work.local/work || true)
                        if [ "\$STATUS" = "200" ]; then
                            echo "Service OK after rollback"
                            exit 0
                        fi
                        sleep 5
                    done
                    echo "Service NOT available after rollback!"
                    exit 1
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