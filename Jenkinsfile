pipeline {
  agent any

  // One Jenkinsfile for both CI (DEPLOY=false) and CD (DEPLOY=true)
  parameters {
    booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Run deployment (CD)?')
    string(name: 'CI_JOB', defaultValue: 'node-ci', description: 'CI job name to fetch artifact from (used when DEPLOY=true)')
  }

  tools {
    nodejs 'NodeJS 20'   // Jenkins > Manage Jenkins > Global Tool Configuration
  }

  environment {
    // Azure targets (adjust only if your names differ)
    AZURE_RESOURCE_GROUP = 'Project-5'
    AZURE_WEBAPP_NAME    = 'Project-5'   // If deploy fails, confirm the exact (usually lowercase) web app name
    AZ_SUBSCRIPTION      = 'e5590ea2-6632-45ad-b793-2a856f1c41a8'

    // Credentials
    AZ_SP     = credentials('azure-sp')       // provides AZ_SP_USR (client id) and AZ_SP_PSW (secret value)
    AZ_TENANT = credentials('azure-tenant')   // provides AZ_TENANT (tenant id)
  }

  options { skipDefaultCheckout(true) }

  stages {
    stage('Checkout from Git') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD || true'
      }
    }

    stage('Install Dependencies (CI)') {
      when { expression { !params.DEPLOY } }   // CI only
      steps {
        sh 'npm ci || npm install'
      }
    }

    stage('Package & Archive (CI)') {
      when { expression { !params.DEPLOY } }   // CI only
      steps {
        sh '''
          rm -f build.zip || true
          zip -r build.zip . \
            -x "*.git/*" "*.github/*" "Jenkinsfile" ".vscode/*" \
               "*node_modules/*" "*.log" "coverage/*"
        '''
        archiveArtifacts artifacts: 'build.zip', fingerprint: true
      }
    }

    stage('Fetch Artifact from CI (CD)') {
      when { expression { params.DEPLOY } }    // CD only
      steps {
        copyArtifacts projectName: params.CI_JOB, selector: lastSuccessful(), filter: 'build.zip', fingerprintArtifacts: true
        sh 'ls -lh build.zip'
      }
    }

    stage('Login to Azure (CD)') {
      when { expression { params.DEPLOY } }    // CD only
      steps {
        sh '''
          set +x
          az login --service-principal -u "$AZ_SP_USR" -p "$AZ_SP_PSW" --tenant "$AZ_TENANT"
          az account set --subscription "$AZ_SUBSCRIPTION"
          set -x
        '''
      }
    }

    stage('Deploy to Azure Web App (CD)') {
      when { expression { params.DEPLOY } }    // CD only
      steps {
        sh """
          if az webapp deploy -h >/dev/null 2>&1; then
            az webapp deploy \
              --resource-group "$AZURE_RESOURCE_GROUP" \
              --name "$AZURE_WEBAPP_NAME" \
              --src-path build.zip \
              --type zip \
              --clean true
          else
            az webapp deployment source config-zip \
              --resource-group "$AZURE_RESOURCE_GROUP" \
              --name "$AZURE_WEBAPP_NAME" \
              --src build.zip
          fi
        """
      }
    }
  }

  post {
    always { echo 'Done.' }
  }
}