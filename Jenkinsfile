pipeline {
  agent any
  options {
    timestamps()
  }

  environment {
    AWS_REGION   = 'ap-southeast-1'
    EKS_CLUSTER  = 'eksdevopssd6116'
    NAMESPACE    = 'dev'

    BACKEND_IMAGE_NAME  = 'backend'
    FRONTEND_IMAGE_NAME = 'frontend'

    TAG = ''
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
          // TAG = <branch>-<buildNumber>-<shortsha>
          def shortSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          def branch   = env.BRANCH_NAME ?: 'main'
          env.TAG = "${branch}-${env.BUILD_NUMBER}-${shortSha}"

          def hasDots = sh(script: "grep -qx '\\.\\.\\.' src/frontend/Dockerfile && echo yes || echo no", returnStdout: true).trim()
          env.BuildFrontend = (hasDots == 'yes') ? 'false' : 'true'

          echo "TAG=${env.TAG}"
          echo "BuildFrontend=${env.BuildFrontend}"
        }
      }
    }

    stage('AWS Login & ECR Login') {
      steps {
        script {
          sh """
            aws --version
            aws sts get-caller-identity
            """
          // Nếu bạn dùng AccessKey/Secret (không chạy Jenkins trên EC2/role):
          // withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds-id']]) { ... }
          sh """
            ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)
            echo "ACCOUNT_ID=\$ACCOUNT_ID"
            echo "AWS_REGION=${AWS_REGION}"
            ECR_REGISTRY="\$ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com"
            echo "ECR_REGISTRY=\$ECR_REGISTRY" > ecr.env

            aws ecr get-login-password --region ${AWS_REGION} \
              | docker login --username AWS --password-stdin "\$ECR_REGISTRY"

            # Tạo repo nếu chưa có (idempotent)
            aws ecr describe-repositories --repository-name ${BACKEND_IMAGE_NAME} --region ${AWS_REGION} >/dev/null 2>&1 || \
              aws ecr create-repository --repository-name ${BACKEND_IMAGE_NAME} --region ${AWS_REGION}
            if [ "${BuildFrontend}" = "true" ]; then
              aws ecr describe-repositories --repository-name ${FRONTEND_IMAGE_NAME} --region ${AWS_REGION} >/dev/null 2>&1 || \
                aws ecr create-repository --repository-name ${FRONTEND_IMAGE_NAME} --region ${AWS_REGION}
            fi
          """
        }
      }
    }

    stage('Build & Push Docker Images') {
      steps {
        script {
          def e = readProperties file: 'ecr.env'
          def REG = e['ECR_REGISTRY']

          // Backend
          sh """
            docker build -f src/backend/Dockerfile -t ${BACKEND_IMAGE_NAME}:${TAG} src/backend
            docker tag ${BACKEND_IMAGE_NAME}:${TAG} ${REG}/${BACKEND_IMAGE_NAME}:${TAG}
            docker push ${REG}/${BACKEND_IMAGE_NAME}:${TAG}
          """

          // Frontend
          if (env.BuildFrontend == 'true') {
            sh """
              docker build -f src/frontend/Dockerfile -t ${FRONTEND_IMAGE_NAME}:${TAG} src/frontend
              docker tag ${FRONTEND_IMAGE_NAME}:${TAG} ${REG}/${FRONTEND_IMAGE_NAME}:${TAG}
              docker push ${REG}/${FRONTEND_IMAGE_NAME}:${TAG}
            """
          } else {
            echo 'Skip building frontend because Dockerfile contains "..." placeholder.'
          }
        }
      }
    }

    stage('Render manifests (Kustomize)') {
      steps {
        script {
          def e = readProperties file: 'ecr.env'
          def REG = e['ECR_REGISTRY']

          sh """
            set -euo pipefail
            MANIFEST_DIR="aws"
            [ -d "\$MANIFEST_DIR" ] || MANIFEST_DIR="azure"

            mkdir -p out
            cp -r "\$MANIFEST_DIR"/* out/

            {
              echo "resources:"
              [ -f out/backend.yaml ] && echo "- backend.yaml"
              if [ "${BuildFrontend:-false}" = "true" ] && [ -f out/frontend.yaml ]; then echo "- frontend.yaml"; fi
              [ -f out/mongodb.yaml ] && echo "- mongodb.yaml"
              [ -f out/ingress.yml ]  && echo "- ingress.yml"
              echo "images:"
              echo "- name: ${REG}/${BACKEND_IMAGE_NAME}"
              echo "  newName: ${REG}/${BACKEND_IMAGE_NAME}"
              echo "  newTag: ${TAG}"
              if [ "${BuildFrontend:-false}" = "true" ] && [ -f out/frontend.yaml ]; then
                echo "- name: ${REG}/${FRONTEND_IMAGE_NAME}"
                echo "  newName: ${REG}/${FRONTEND_IMAGE_NAME}"
                echo "  newTag: ${TAG}"
              fi
            } > out/kustomization.yaml

            echo "----- kustomization.yaml -----"
            cat out/kustomization.yaml

            # Render
            kubectl kustomize out > out/rendered.yaml
          """
        }
        archiveArtifacts artifacts: 'out/rendered.yaml', fingerprint: true
      }
    }

    stage('Deploy to EKS') {
      steps {
        script {
          def e = readProperties file: 'ecr.env'
          sh """
            aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}

            cat <<EOF | kubectl apply -f -
            apiVersion: v1
            kind: Namespace
            metadata:
              name: ${NAMESPACE}
            EOF

            kubectl apply -n ${NAMESPACE} -f out/rendered.yaml

            # Rollout status backend
            kubectl rollout status deploy/backend -n ${NAMESPACE} --timeout=180s || true
          """
          if (env.BuildFrontend == 'true') {
            sh """
              kubectl rollout status deploy/frontend -n ${NAMESPACE} --timeout=180s || true
            """
          }
          sh "kubectl get pods,svc -n ${NAMESPACE} -o wide || true"
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
