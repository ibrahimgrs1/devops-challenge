pipeline {
    agent any
    
    environment {
        // İmaj ismini ve Registry adresini tanımlıyoruz
        REGISTRY_URL = "host.docker.internal:5000" // Jenkins içinden Host'a erişim için
        IMAGE_NAME   = "todo-app"
        // Commit SHA'yı versiyon olarak kullanıyoruz (Challenge gereksinimi)
        IMAGE_TAG    = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    stages {
        stage('Kodu Çek (Checkout)') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    // Docker paketini oluşturuyoruz
                    sh "docker build -t ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG} ."
                    sh "docker tag ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_URL}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Registry Push') {
            steps {
                script {
                    // Özel depomuza (Registry) giriş yapıp paketi gönderiyoruz
                    // Not: Şifreyi Jenkins içinden güvenli bir şekilde vereceğiz
                    withCredentials([usernamePassword(credentialsId: 'registry-credentials', passwordVariable: 'REG_PASS', usernameVariable: 'REG_USER')]) {
                        sh "echo \$REG_PASS | docker login ${REGISTRY_URL} -u \$REG_USER --password-stdin"
                        sh "docker push ${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${REGISTRY_URL}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deployment') {
            steps {
                // Docker Compose ile uygulamayı ayağa kaldırıyoruz
                sh "docker-compose up -d --force-recreate"
            }
        }
    }
}