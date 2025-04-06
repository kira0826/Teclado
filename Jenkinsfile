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
                    echo "Host: ${env.REMOTE_HOST}"
                    echo "Usuario: ${env.REMOTE_USER}"
                    echo "Ruta destino: ${env.REMOTE_PATH}"

                    // Crear el script de despliegue con logs
                    writeFile file: 'deploy.sh', text: """#!/bin/bash
                    echo "Intentando conexión SSH para validar autenticación..."
                    sshpass -p '${REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "echo '✅ Conexión SSH exitosa con ${REMOTE_USER}'"

                    echo "Ejecutando SCP..."
                    sshpass -p '${REMOTE_PASSWORD}' scp -v -o StrictHostKeyChecking=no -r css index.html script.js ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}
                    echo "✔️ SCP completado"
                    sudo systemctl restart nginx.service
                    """
                    sh 'chmod +x deploy.sh'

                    // Mostrar contenido del script generado
                    echo "Contenido del script deploy.sh:"
                    sh 'cat deploy.sh'

                    // Ejecutar script
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
