pipeline {
    agent any
    
    // BONUS: Geri dönüş (Rollback) için parametre tanımlıyoruz
    parameters {
        string(name: 'DEPLOY_TAG', defaultValue: 'latest', description: 'Dağıtmak istediğiniz sürüm etiketi (Build No, Commit SHA veya latest)')
    }

    environment {
        REGISTRY_URL = "host.docker.internal:5000"
        IMAGE_NAME   = "todo-app"
        // Commit SHA'yı versiyon olarak alıyoruz
        IMAGE_TAG    = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        // Build numarasını rollback için benzersiz etiket olarak kullanıyoruz
        BUILD_TAG    = "${env.BUILD_NUMBER}"
    }

    stages {
        // Not: 'Kodu Çek' aşaması silindi çünkü Jenkins (Declarative Pipeline) kodu zaten otomatik çekiyor.

        stage('Docker Build (Optimized)') {
            steps {
                script {
                    // Önceki başarılı imajı cache olarak çekiyoruz
                    sh "docker pull ${REGISTRY_URL}/${IMAGE_NAME}:latest || true"
                    
                    // İmajı 3 farklı etiketle build ediyoruz: SHA, Build No ve Latest
                    sh """
                        docker build --cache-from ${REGISTRY_URL}/${IMAGE_NAME}:latest \
                        -t ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG} \
                        -t ${REGISTRY_URL}/${IMAGE_NAME}:${BUILD_TAG} \
                        -t ${REGISTRY_URL}/${IMAGE_NAME}:latest .
                    """
                }
            }
        }

        stage('Registry Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'registry-credentials', passwordVariable: 'REG_PASS', usernameVariable: 'REG_USER')]) {
                        sh "echo \$REG_PASS | docker login ${REGISTRY_URL} -u \$REG_USER --password-stdin"
                        
                        // Tüm etiketleri depoya (Registry) gönderiyoruz
                        sh "docker push ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${REGISTRY_URL}/${IMAGE_NAME}:${BUILD_TAG}"
                        sh "docker push ${REGISTRY_URL}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deployment & Health Check') {
            steps {
                script {
                    // BONUS: Parametreden gelen etiketi kullanarak deployment yapıyoruz
                    echo "Dağıtılan Sürüm Etiketi: ${params.DEPLOY_TAG}"
                    
                    // Docker Compose'a IMAGE_TAG değişkenini gönderiyoruz
                    sh "IMAGE_TAG=${params.DEPLOY_TAG} docker-compose up -d --force-recreate"
                    
                    // Health Check - Yeni mimariye uygun Nginx kontrolü
                    echo "Sistem sağlığı kontrol ediliyor..."
                    sleep 10
                    sh "docker ps | grep nginx-proxy || (echo 'HATA: Nginx Proxy ayağa kalkamadı!' && exit 1)"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline başarıyla tamamlandı! Dağıtılan sürüm: ${params.DEPLOY_TAG}"
            sh "docker image prune -f"
        }
        failure {
            echo "Hata oluştu! İbrahim, logları ve Jenkins Build #${env.BUILD_NUMBER} kayıtlarını kontrol et."
        }
    }
}