pipeline {
    agent any  // Usa cualquier agente con SSH/SCP instalado

    stages {
        stage('Verificar/Instalar SSH y SCP') {
            steps {
                sh '''
                    # Verifica si sshpass está instalado (para automatizar la contraseña)
                    if ! command -v sshpass &> /dev/null; then
                        echo "Instalando sshpass..."
                        # Detecta el gestor de paquetes
                        if [ -f /etc/debian_version ]; then
                            sudo apt-get update -qq && sudo apt-get install -y sshpass openssh-client
                        elif [ -f /etc/redhat-release ]; then
                            sudo yum install -y sshpass openssh-clients
                        else
                            echo "ERROR: Sistema no soportado. Instala sshpass manualmente."
                            exit 1
                        fi
                    fi
                '''
            }
        }

        stage('Despliegue vía SCP') {
            steps {
                withCredentials([string(credentialsId: 'credencial-ssh', variable: 'SSH_PASSWORD')]) {
                    sh """
                        # Usa sshpass para evitar prompts de contraseña
                        sshpass -p "$SSH_PASSWORD" scp -o StrictHostKeyChecking=no -r \
                            *.html *.css assets/ js/ images/ \
                            ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}/

                        # Verifica que los archivos se copiaron
                        sshpass -p "$SSH_PASSWORD" ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} \
                            "ls -la ${REMOTE_PATH}"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Archivos transferidos exitosamente a ${REMOTE_HOST}:${REMOTE_PATH}"
        }
        failure {
            echo "❌ Error en el despliegue. Revisa los logs."
        }
    }
}