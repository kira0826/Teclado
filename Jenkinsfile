pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Preparar SSH') {
            steps {
                script {
                    // Instalar sshpass sin usar sudo (en contenedores)
                    sh '''
                        if ! command -v sshpass &> /dev/null; then
                            apt-get update && apt-get install -y sshpass
                        fi
                    '''
                }
            }
        }
        
        stage('Deploy') {
            steps {
                // Utilizar las variables de entorno definidas globalmente para transferir los archivos estáticos
                script {
                    
                    
                    sh '''
                        export SSHPASS=${REMOTE_PASSWORD}
                        
                        # Crear directorio de destino si no existe
                        sshpass -e ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "mkdir -p ${REMOTE_PATH}"
                        
                        # Transferir archivos HTML fy CSS utilizando SCP
                        sshpass -e scp -o StrictHostKeyChecking=no -r *.html *.css assets/ images/ js/ ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}/
                        
                        echo "Archivos estáticos transferidos exitosamente a ${REMOTE_HOST}:${REMOTE_PATH}"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo 'Despliegue completado con éxito! Los archivos estáticos ya están disponibles en el servidor frontend.'
        }
        failure {
            echo 'El proceso de despliegue ha fallado. Revisa los logs para más detalles.'
        }
    }
}