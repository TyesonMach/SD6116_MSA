pipeline {
  agent any
  options {
    timestamps()
  }

  environment {
    AWS_REGION   = 'ap-southeast-1'
    EKS_CLUSTER  = 'devops-lab-eks'
    NAMESPACE    = 'dev'

    BACKEND_IMAGE_NAME  = 'backend'
    FRONTEND_IMAGE_NAME = 'frontend'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'git --no-pager log -1 --oneline'
      }
    }

    stage('Compute TAG & Decide FE Build') {
      steps {
        script {
          def shortSha = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
          def branch   = (env.BRANCH_NAME ?: 'main').replaceAll('[^A-Za-z0-9._-]', '-')
          env.TAG = "${branch}-${env.BUILD_NUMBER}-${shortSha}"

          env.BuildFrontend = sh(
            script: "grep -qx '\\.\\.\\.' src/frontend/Dockerfile && echo false || echo true",
            returnStdout: true
          ).trim()

          echo "TAG=${env.TAG}"
          echo "BuildFrontend=${env.BuildFrontend}"
        }
      }
    }

  stage('AWS Login & ECR Login') {
    steps {
      withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds-id']]) {
        script {
          sh '''
            set -euo pipefail

            aws --version
            aws sts get-caller-identity >/dev/null

            ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
            REG="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

            aws ecr describe-repositories --repository-name "$BACKEND_IMAGE_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || \
              aws ecr create-repository --repository-name "$BACKEND_IMAGE_NAME" --region "$AWS_REGION"

            if [ "$BuildFrontend" = "true" ]; then
              aws ecr describe-repositories --repository-name "$FRONTEND_IMAGE_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || \
                aws ecr create-repository --repository-name "$FRONTEND_IMAGE_NAME" --region "$AWS_REGION"
            fi

            aws ecr get-login-password --region "$AWS_REGION" \
              | docker login --username AWS --password-stdin "$REG"

            echo "ECR_REGISTRY=$REG" > ecr.env
            echo "REG=$REG"
          '''

          env.ECR_REGISTRY = sh(
            script: 'aws sts get-caller-identity --query Account --output text',
            returnStdout: true
          ).trim() + ".dkr.ecr.${env.AWS_REGION}.amazonaws.com"
          echo "ECR_REGISTRY=${env.ECR_REGISTRY}"
        }
      }
    }
  }


    stage('Build & Push Docker Images') {
      steps {
        script {
          def REG = (env.ECR_REGISTRY ?: '').trim()
          if (!env.TAG?.trim()) {
            error "TAG is empty. Check stage 'Compute TAG & Decide FE Build'."
          }
          if (!REG) {
            error "ECR_REGISTRY is empty. Ensure stage 'AWS Login & ECR Login' set env.ECR_REGISTRY."
          }

          echo "Using REG=${REG}, TAG=${env.TAG}, BuildFrontend=${env.BuildFrontend}"

          withEnv(["DOCKER_BUILDKIT=1"]) {
            // Backend
            sh """
              set -e
              docker build -f src/backend/Dockerfile -t ${BACKEND_IMAGE_NAME}:${env.TAG} src/backend
              docker tag   ${BACKEND_IMAGE_NAME}:${env.TAG} ${REG}/${BACKEND_IMAGE_NAME}:${env.TAG}
              docker push  ${REG}/${BACKEND_IMAGE_NAME}:${env.TAG}
            """

            // Frontend
            if (env.BuildFrontend == 'true') {
              sh """
                set -e
                docker build -f src/frontend/Dockerfile -t ${FRONTEND_IMAGE_NAME}:${env.TAG} src/frontend
                docker tag   ${FRONTEND_IMAGE_NAME}:${env.TAG} ${REG}/${FRONTEND_IMAGE_NAME}:${env.TAG}
                docker push  ${REG}/${FRONTEND_IMAGE_NAME}:${env.TAG}
              """
            } else {
              echo 'Skip building frontend because Dockerfile contains "..." placeholder.'
            }
          }
        }
      }
    }

    stage('Render manifests (Kustomize)') {
      steps {
        script {
          def REG = env.ECR_REGISTRY
          sh """
            set -euo pipefail

            MANIFEST_DIR="aws"
            [ -d "\$MANIFEST_DIR" ] || MANIFEST_DIR="azure"

            BUILD_FRONTEND='${env.BuildFrontend}'
            ECR_REG='${REG}'
            BE='${BACKEND_IMAGE_NAME}'
            FE='${FRONTEND_IMAGE_NAME}'
            TAG='${env.TAG}'

            mkdir -p out
            cp -r "\$MANIFEST_DIR"/* out/ || true

            {
              echo "resources:"
              [ -f out/backend.yaml ]  && echo "- backend.yaml"
              if [ "\$BUILD_FRONTEND" = "true" ] && [ -f out/frontend.yaml ]; then echo "- frontend.yaml"; fi
              [ -f out/mongodb.yaml ]  && echo "- mongodb.yaml"
              [ -f out/ingress.yml ]   && echo "- ingress.yml"

              echo "images:"
              echo "- name: \$ECR_REG/\$BE"
              echo "  newName: \$ECR_REG/\$BE"
              echo "  newTag: \$TAG"

              if [ "\$BUILD_FRONTEND" = "true" ] && [ -f out/frontend.yaml ]; then
                echo "- name: \$ECR_REG/\$FE"
                echo "  newName: \$ECR_REG/\$FE"
                echo "  newTag: \$TAG"
              fi
            } > out/kustomization.yaml

            echo '----- kustomization.yaml -----'
            cat out/kustomization.yaml

            kubectl kustomize out > out/rendered.yaml
          """
        }
        archiveArtifacts artifacts: 'out/rendered.yaml', fingerprint: true
      }
    }

    stage('Deploy to EKS') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds-id']]) {
          script {
            sh """
              set -e

              aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}

              kubectl get ns ${NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${NAMESPACE}

              kubectl apply -n ${NAMESPACE} -f out/rendered.yaml

              kubectl rollout status deploy/backend  -n ${NAMESPACE} --timeout=180s || true
            """
            if (env.BuildFrontend == 'true') {
              sh "kubectl rollout status deploy/frontend -n ${NAMESPACE} --timeout=180s || true"
            }
            sh "kubectl get pods,svc -n ${NAMESPACE} -o wide || true"
          }
        }
      }
    }
  }

  post {
    always {
      echo "Build finished with status: ${currentBuild.currentResult}"
    }
  }
}
