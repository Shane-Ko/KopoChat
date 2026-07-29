pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2-debug
    imagePullPolicy: IfNotPresent
    command:
    - sleep
    args:
    - "9999999"
    resources:
      requests:
        memory: "512Mi"
        cpu: "200m"
      limits:
        memory: "2Gi"
        cpu: "2000m"
    volumeMounts:
    - name: kaniko-secret
      mountPath: /kaniko/.docker
  - name: git
    image: alpine/git:latest
    command:
    - sleep
    args:
    - "9999999"
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "256Mi"
        cpu: "200m"
  - name: jnlp
    resources:
      requests:
        memory: "128Mi"
        cpu: "50m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  restartPolicy: Never
  volumes:
  - name: kaniko-secret
    secret:
      secretName: harbor-cred
      items:
      - key: .dockerconfigjson
        path: config.json
'''
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        HARBOR_URL = 'std-harbor.kopoctc.kr'
        HARBOR_PROJECT = 'kopo02'
        IMAGE_NAME = 'kopochat'
        IMAGE_TAG = "v${BUILD_NUMBER}"
        GITOPS_REPO = 'std-gitlab.kopoctc.kr/kopo021/gitops.git'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Source code checked out'
                sh 'ls -la'
            }
        }

        stage('Build & Push with Kaniko') {
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                          --context=\${WORKSPACE} \
                          --dockerfile=\${WORKSPACE}/Dockerfile \
                          --destination=${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG} \
                          --skip-tls-verify
                    """
                }
            }
        }

        stage('Update Manifest') {
            steps {
                container('git') {
                    withCredentials([usernamePassword(
                        credentialsId: 'gitlab-token',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh """
                            rm -rf gitops-repo
                            git clone https://\$GIT_USER:\$GIT_TOKEN@${GITOPS_REPO} gitops-repo
                            cd gitops-repo
                            sed -i 's|image: ${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:.*|image: ${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}|g' apps/kopochat/deployment.yaml
                            git config user.email "jenkins@kopoctc.kr"
                            git config user.name "Jenkins"
                            git add apps/kopochat/deployment.yaml
                            git diff --cached --quiet || git commit -m "Update kopochat image to ${IMAGE_TAG}"
                            git push origin main
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded! Image: ${IMAGE_TAG}"
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}
