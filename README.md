### Infrastructure Management

The infrastructure is managed using **Terraform** with the **Azurerm** provider. It is organized into three reusable and modular components:

1. **Networking**
2. **Virtual Machines**
3. **GitHub Integration**

This modular approach allows for greater flexibility and reusability of infrastructure components.

The GitHub module is specifically used to dynamically create a webhook between the frontend repository and the Jenkins server. Since the server's public IP changes every time the infrastructure is redeployed, a DNS-based approach was avoided due to cost considerations.

📁 [GitHub – kira0826/terraform_for_each_vm](https://github.com/kira0826/terraform_for_each_vm)

---

### Frontend, Jenkinsfile, and GitHub Actions Workflow

The [Teclado repository](https://github.com/kira0826/Teclado/) contains the entire frontend, which is deployed to a virtual machine using **Nginx**. The deployment is triggered by a **Jenkinsfile** executed by a Jenkins server hosted in a VM. This deployment process is automatically triggered after a **push to the `main` branch**, which can only occur **after a pull request is approved**.

Additionally, the repository includes a **GitHub Actions workflow** that runs whenever a pull request is created. This workflow performs a **code analysis using a SonarQube server**, which is deployed in the same VM as Jenkins. The results of the analysis are posted as comments on the pull request. Approval is **blocked until the analysis comments are available**, enforcing code quality gates.

A **Terraform-created webhook** notifies the Jenkins server whenever a push occurs in the frontend repository.

📁 [GitHub – kira0826/Teclado](https://github.com/kira0826/Teclado/)

---

### Provisioning with Ansible

Two virtual machines are provisioned using **Ansible**:

1. One dedicated to the **frontend deployment** using **Nginx**.
2. One dedicated to **DevOps services**, running **Jenkins** and **SonarQube** in Docker containers.

All services are fully automated for out-of-the-box functionality.

- **Jenkins** is configured using **Configuration as Code (JCasC)** to automatically create plugins and pipeline jobs.
- **SonarQube** setup includes automatic token generation through Ansible. This token is then securely added as a **GitHub secret** in the frontend repository, allowing GitHub Actions to authenticate and run code analysis pipelines.

### Key Ansible Roles:

1. **`devops_services`**
    
    Installs Docker Compose and starts containers for Jenkins, SonarQube, and PostgreSQL (used by SonarQube).
    
2. **`jenkins`**
    
    Automatically installs required Jenkins plugins via `plugins.txt` and sets up jobs using **Groovy scripts and JCasC**.
    
3. **`enable_nginx`**
    
    Prepares static frontend content, configures `nginx.conf` using templates, and exposes the application on port 80.
    
4. **`sonarqube`**
    
    Generates a SonarQube token and stores it as a **GitHub secret** in the frontend repository to enable code analysis during pull requests.
    


📁 [GitHub – kira0826/ansible-pipeline](https://github.com/kira0826/ansible-pipeline)

---
# Frontend Deployment and Jenkins Integration

The frontend repository contains all the **static files** that are deployed to the **frontend virtual machine** using **Nginx**. It also includes the `Jenkinsfile`, which defines the Jenkins job executed by the Jenkins server. The pipeline looks as follows:

```groovy
pipeline {
    agent any

    stages {
        stage('Prepare Artifacts') {
            steps {
                echo "Listing files to be copied:"
                sh 'ls -la'
            }
        }

        stage('Copy Files to Remote Server') {
            steps {
                script {
                    echo "Host: ${env.REMOTE_HOST}"
                    echo "User: ${env.REMOTE_USER}"
                    echo "Target Path: ${env.REMOTE_PATH}"

                    // Create a deployment script with logs
                    writeFile file: 'deploy.sh', text: """#!/bin/bash
                    echo "Attempting SSH connection for authentication check..."
                    sshpass -p '${REMOTE_PASSWORD}' ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "echo '✅ SSH connection successful with ${REMOTE_USER}'"

                    echo "Executing SCP..."
                    sshpass -p '${REMOTE_PASSWORD}' scp -v -o StrictHostKeyChecking=no -r css index.html script.js ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}
                    echo "✔️ SCP completed"
                    """
                    sh 'chmod +x deploy.sh'

                    // Show the generated script content
                    echo "Contents of deploy.sh:"
                    sh 'cat deploy.sh'

                    // Execute the script
                    sh './deploy.sh'
                }
            }
        }
    }

    post {
        success {
            echo '✔️ Files copied successfully.'
        }
        failure {
            echo '❌ Failed to copy files.'
        }
    }
}

```

This pipeline uses **environment variables** defined and provisioned by **Ansible**. The `sshpass` utility, required for password-based SSH and SCP, was installed by customizing the default Jenkins Docker image.

Jenkins is configured using **Jenkins Configuration as Code (JCasC)**. The job is defined as follows:

```yaml
jobs:
  - script: >
      pipelineJob('teclado-pipeline') {
        definition {
          cpsScm {
            scm {
              git {
                remote {
                  url('https://github.com/kira0826/Teclado.git')
                }
                branches('*/main')
              }
            }
            scriptPath('Jenkinsfile')
          }
        }
        triggers {
          githubPush()
        }
      }

```

Note: The `Jenkinsfile` is retrieved via **SCM (Git)**, pointing to the GitHub repository.

---

## SonarQube Integration via GitHub Actions

This repository also includes a **GitHub Actions workflow** for **SonarQube analysis**. Secrets such as the **SonarQube token** and **host URL** are provisioned via **Ansible** and used within the workflow:

```yaml
name: SonarQube Analysis
on:
  pull_request:
    branches:
      - main

jobs:
  sonar:
    name: Run SonarQube
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '11'

      - name: Setup SonarQube Scanner
        uses: SonarSource/sonarqube-scan-action@v5.1.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST }}
        with:
          args: >
            -Dsonar.projectKey=TecladoAnal
            -Dsonar.sources=.

      - name: Install jq (for parsing JSON)
        run: sudo apt-get update && sudo apt-get install -y jq

      - name: Get SonarQube analysis report
        id: sonar-report
        run: |
          PROJECT_KEY="TecladoAnal"
          SONAR_URL="${{ secrets.SONAR_HOST }}"
          SONAR_TOKEN="${{ secrets.SONAR_TOKEN }}"

          RESPONSE=$(curl -s -u $SONAR_TOKEN: "$SONAR_URL/api/measures/component?component=$PROJECT_KEY&metricKeys=code_smells,bugs,vulnerabilities,coverage,duplicated_lines_density")
          echo "$RESPONSE" > sonar-report.json

          # Parse JSON with jq
          BUGS=$(jq -r '.component.measures[] | select(.metric=="bugs") | .value' sonar-report.json)
          CODE_SMELLS=$(jq -r '.component.measures[] | select(.metric=="code_smells") | .value' sonar-report.json)
          VULNERABILITIES=$(jq -r '.component.measures[] | select(.metric=="vulnerabilities") | .value' sonar-report.json)
          COVERAGE=$(jq -r '.component.measures[] | select(.metric=="coverage") | .value' sonar-report.json)
          DUPLICATION=$(jq -r '.component.measures[] | select(.metric=="duplicated_lines_density") | .value' sonar-report.json)

          REPORT="🚨 **SonarQube Report**
          - 🐞 Bugs: $BUGS
          - 💨 Code Smells: $CODE_SMELLS
          - 🔐 Vulnerabilities: $VULNERABILITIES
          - 🧪 Coverage: $COVERAGE%
          - 📦 Duplication: $DUPLICATION%"

          echo "$REPORT"
          echo "report<<EOF" >> $GITHUB_OUTPUT
          echo "$REPORT" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          header: sonar-report
          message: ${{ steps.sonar-report.outputs.report }}

```

> ⚠️ Note: Since we’re using the Community Edition of SonarQube, automatic pull request decoration (native comments) is not supported.
> 
> 
> As a workaround, we manually extract key metrics from the SonarQube API response and format them into a Markdown comment, which is then posted using the `marocchino/sticky-pull-request-comment` GitHub Action.
>