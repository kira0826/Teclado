pipeline {
    agent {
        docker {
            image 'alpine'       // Imagen base (puedes usar una que ya tenga sshpass si prefieres)
            args '-u root'       // Ejecutar como root para instalar paquetes
        }
    }

    stages {
        stage('Instalar sshpass y openssh') {
            steps {
                sh '''
                    apk update
                    apk add sshpass openssh
                '''
            }
        }

        stage('Desplegar por SCP') {
            steps {
                withCredentials([string(credentialsId: 'credencial-ssh', variable: 'SSH_PASSWORD')]) {
                    sh '''
                        sshpass -p "$SSH_PASSWORD" scp -o StrictHostKeyChecking=no -r \
                            *.html *.css assets/ js/ images/ \
                            ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}/

                        sshpass -p "$SSH_PASSWORD" ssh -o StrictHostKeyChecking=no \
                            ${REMOTE_USER}@${REMOTE_HOST} "ls -la ${REMOTE_PATH}"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Despliegue exitoso a ${REMOTE_HOST}:${REMOTE_PATH}"
        }
        failure {
            echo "❌ Falló el despliegue"
        }
    }
}
