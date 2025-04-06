pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Deploy') {
            steps {
                // Utilizar las variables de entorno definidas globalmente para transferir los archivos estáticos
                script {
                    // Instalar sshpass si es necesario
                    sh 'which sshpass || sudo apt-get install -y sshpass'
                    
                    // Usar SCP con sshpass para evitar la interacción manual con la contraseña
                    sh '''
                        export SSHPASS=${REMOTE_PASSWORD}
                        
                        # Crear directorio de destino si no existe
                        sshpass -e ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "mkdir -p ${REMOTE_PATH}"
                        
                        # Transferir archivos HTML y CSS utilizando SCP
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