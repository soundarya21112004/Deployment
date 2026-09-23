pipeline {
    agent any

    environment {
        ENVIRONMENT = 'UAT'
        KUBE_NAMESPACE = 'mosip'
        
        // Exact OpenShift Cluster API Endpoint
        OPENSHIFT_API_URL = 'https://api.uat.mspsandbox.com:6443'
        
        // Jenkins Credentials Manager Secret Text ID
        OPENSHIFT_TOKEN_CREDS_ID = 'openshift-uat-token'
    }

    stages {
        stage('Process Docker Hub Payload') {
            steps {
                script {
                    // Docker Hub Webhook payload detect pannum logic
                    if (env.DOCKER_TAG_NAME) {
                        IMAGE_TAG = env.DOCKER_TAG_NAME
                        IMAGE_NAME = env.DOCKER_REPO_NAME
                        SERVICE_NAME = IMAGE_NAME.tokenize('/')[-1]
                    } else {
                        // Fallback manual defaults
                        SERVICE_NAME = params.SERVICE_NAME ?: 'packetmanager-service'
                        IMAGE_NAME = params.IMAGE_NAME ?: 'mspeagle/malawi-packet-manager'
                        IMAGE_TAG = params.IMAGE_TAG ?: 'prod-30-JAN-2026'
                    }

                    echo "=================================================="
                    echo "🚀 TARGETING OPENSHIFT UAT CLUSTER"
                    echo "🔗 Cluster URL: ${OPENSHIFT_API_URL}"
                    echo "🎯 Updating Service: ${SERVICE_NAME}"
                    echo "📦 Docker Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "=================================================="
                }
            }
        }

        stage('OpenShift Login') {
            steps {
                script {
                    echo "Authenticating to OpenShift UAT via Service Account Token..."
                    withCredentials([string(credentialsId: OPENSHIFT_TOKEN_CREDS_ID, variable: 'SA_TOKEN')]) {
                        sh """
                            oc login --token=${SA_TOKEN} --server=${OPENSHIFT_API_URL} --insecure-skip-tls-verify=true
                            oc project ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }

        stage('Update Target Service') {
            steps {
                script {
                    echo "Updating ONLY '${SERVICE_NAME}' deployment..."
                    sh """
                        kubectl set image deployment/${SERVICE_NAME} ${SERVICE_NAME}=${IMAGE_NAME}:${IMAGE_TAG} -n ${KUBE_NAMESPACE}
                        kubectl rollout status deployment/${SERVICE_NAME} -n ${KUBE_NAMESPACE} --timeout=180s
                    """
                }
            }
        }

        stage('Verify Active Pods') {
            steps {
                script {
                    sh "kubectl get pods -l app=${SERVICE_NAME} -n ${KUBE_NAMESPACE}"
                }
            }
        }
    }

    post {
        always {
            sh "oc logout || true"
        }
        success {
            echo "✅ Successfully updated ${SERVICE_NAME} on UAT (${OPENSHIFT_API_URL})!"
        }
        failure {
            echo "❌ Rollout failed for ${SERVICE_NAME}. Triggering automatic rollback..."
            sh "kubectl rollout undo deployment/${SERVICE_NAME} -n ${KUBE_NAMESPACE}"
        }
    }
}
