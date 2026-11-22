pipeline {
    agent any
    environment {
        SERVICE = 'exchange'
        NAME = "garciapp/${env.SERVICE}"
        TRIVY = 'aquasec/trivy:0.76.1'
    }
    stages {
        // Sem Maven: o Exchange é Python. O Jenkins não tem Python instalado, então os testes e o
        // pip-audit rodam dentro do estágio "test" do Dockerfile. Falhou um teste ou apareceu
        // vulnerabilidade conhecida numa dependência: o pipeline para aqui.
        stage('Test & Audit') {
            steps {
                sh "docker build --pull --target test -t ${env.SERVICE}:test ."
            }
        }
        // Bottleneck 2 (vulnerability scanning): o Trivy varre a imagem final (sistema da imagem
        // base + pacotes Python) ANTES de publicar. Vulnerabilidade HIGH ou CRITICAL que já tem
        // correção para o pipeline. O socket do Docker é o do host (o Jenkins já o monta).
        stage('Image Scan') {
            steps {
                sh "docker build -t ${env.SERVICE}:scan ."
                sh """
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v trivy-cache:/root/.cache/ \
                        ${env.TRIVY} image --scanners vuln \
                        --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
                        ${env.SERVICE}:scan
                """
            }
        }
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credential', usernameVariable: 'USERNAME', passwordVariable: 'TOKEN')]) {
                    sh 'echo $TOKEN | docker login -u $USERNAME --password-stdin'
                    sh "docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-${env.SERVICE} --name multi-platform-builder-${env.SERVICE}"
                    sh "docker buildx build --platform=linux/arm64,linux/amd64 --push --tag ${env.NAME}:latest --tag ${env.NAME}:${env.BUILD_ID} -f Dockerfile ."
                    sh "docker buildx rm --force multi-platform-builder-${env.SERVICE}"
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-credential',
                        usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        export AWS_DEFAULT_REGION=us-east-2
                        trap 'rm -f ./kubeconfig' EXIT
                        if ! out=$(aws eks describe-cluster --name eks-store 2>&1); then
                            if echo "$out" | grep -q ResourceNotFoundException; then
                                echo "Cluster eks-store nao esta de pe: pulando o deploy"; exit 0
                            fi
                            echo "$out"; exit 1
                        fi
                        aws eks update-kubeconfig --name eks-store --kubeconfig ./kubeconfig
                        kubectl --kubeconfig ./kubeconfig set image deployment/${SERVICE} ${SERVICE}=${NAME}:${BUILD_ID}
                        kubectl --kubeconfig ./kubeconfig rollout status deployment/${SERVICE} --timeout=300s
                    '''
                }
            }
        }
    }
    post {
        always {
            // limpa as imagens locais de teste/varredura (o que vai pro Docker Hub é o buildx)
            sh "docker rmi -f ${env.SERVICE}:test ${env.SERVICE}:scan || true"
        }
    }
}
