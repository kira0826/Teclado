pipeline {
    agent {
        docker {
            image 'alpine/sshpass'
            reuseNode true
        }
    }
    
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        
        stage('Deploy') {
            steps {
                sh 'apk add --no-cache openssh-client'
                script {
                    sh '''
                        export SSHPASS=${REMOTE_PASSWORD}
                        # Crear directorio remoto si no existe
                        sshpass -e ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "mkdir -p ${REMOTE_PATH}"
                        # Copiar archivos
                        sshpass -e scp -o StrictHostKeyChecking=no -r *.html *.css assets/ images/ js/ ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}/
                    '''
                }
            }
        }
    }
}