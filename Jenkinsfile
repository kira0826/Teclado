pipeline {
    agent any

    stages {
        stage('Preparar artefactos') {
            steps {
                echo "Listando archivos que se copiarán:"
                sh 'ls -la'
            }
        }

        stage('Copiar archivos a servidor remoto') {
            steps {
                script {
                    echo "Copiando archivos a ${env.REMOTE_USER}@${env.REMOTE_HOST}:${env.REMOTE_PATH}"

                    writeFile file: 'deploy.sh', text: """#!/bin/bash
                    sshpass -p "${REMOTE_PASSWORD}" scp -o StrictHostKeyChecking=no -r css index.html script.js ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}
                    """
                    sh 'chmod +x deploy.sh'
                    sh './deploy.sh'
                }
            }
        }
    }

    post {
        success {
            echo '✔️ Archivos copiados con éxito.'
        }
        failure {
            echo '❌ Falló la copia de archivos.'
        }
    }
}
