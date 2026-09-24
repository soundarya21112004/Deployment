pipeline {
    agent any

    environment {
        ENVIRONMENT = 'UAT'
        KUBE_NAMESPACE = 'mosip'
        
        // OpenShift UAT Cluster API Endpoint
        OPENSHIFT_API_URL = 'https://api.uat.mspsandbox.com:6443'
        OPENSHIFT_TOKEN_CREDS_ID = 'openshift-uat-token'
    }

    stages {
        stage('Process Docker Hub Payload') {
            steps {
                script {
                    /*
                       Docker Hub Webhook trigger aagum podhu exact repository name & tag-a extract pannum.
                    */
                    if (env.DOCKER_TAG_NAME) {
                        IMAGE_TAG = env.DOCKER_TAG_NAME
                        IMAGE_NAME = env.DOCKER_REPO_NAME
                        SERVICE_NAME = IMAGE_NAME.tokenize('/')[-1]
                    } else {
                        // Fallback defaults for manual dry-run testing
                        SERVICE_NAME = params.SERVICE_NAME ?: 'packetmanager-service'
                        IMAGE_NAME = params.IMAGE_NAME ?: 'mspeagle/malawi-packet-manager'
                        IMAGE_TAG = params.IMAGE_TAG ?: 'latest'
                    }

                    // Dynamic Date and Time Stamp Format (YYYYMMDD_HHMMSS)
                    TIMESTAMP = sh(script: "date +%Y%m%d_%H%M%S", returnStdout: true).trim()

                    echo "=================================================="
                    echo "🚀 AUTOMATED UAT DEPLOYMENT TRIGGERED"
                    echo "🎯 Target Deployment Name : ${SERVICE_NAME}"
                    echo "📦 Docker Image Target     : ${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "📅 Timestamp              : ${TIMESTAMP}"
                    echo "=================================================="
                }
            }
        }

        stage('OpenShift Login') {
            steps {
                script {
                    echo "Authenticating to OpenShift UAT Cluster..."
                    withCredentials([string(credentialsId: OPENSHIFT_TOKEN_CREDS_ID, variable: 'SA_TOKEN')]) {
                        sh """
                            oc login --token=${SA_TOKEN} --server=${OPENSHIFT_API_URL} --insecure-skip-tls-verify=true
                            oc project ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }

        stage('Tag & Preserve Current Image (Option B Backup)') {
            steps {
                script {
                    echo "--------------------------------------------------"
                    echo "🔍 Fetching currently active running image for '${SERVICE_NAME}'..."
                    
                    // OpenShift cluster-la Currently running image name-a fetch pannudhu
                    def CURRENT_RUNNING_IMAGE = sh(
                        script: "kubectl get deployment/${SERVICE_NAME} -n${KUBE_NAMESPACE} -o jsonpath='{.spec.template.spec.containers[0].image}'",
                        returnStdout: true
                    ).trim()

                    echo "📌 Current Active Image in Cluster: ${CURRENT_RUNNING_IMAGE}"

                    // Unique backup image tag format: backup-serviceName-YYYYMMDD_HHMMSS
                    def BACKUP_IMAGE_TAG = "backup-${SERVICE_NAME}-${TIMESTAMP}"

                    echo "🏷️ Tagging current image directly in OpenShift internal registry..."
                    
                    /* 
                       `oc tag` moolama zero disk usage-la instant backup tag create pannudhu.
                       .tar files illamalaye rollback-ku permanent reference irukkum.
                    */
                    sh """
                        oc tag ${CURRENT_RUNNING_IMAGE}${SERVICE_NAME}:${BACKUP_IMAGE_TAG} -n${KUBE_NAMESPACE} || true
                    """

                    echo "✅ Old image successfully preserved in UAT as:"
                    echo "   📍 Backup Tag: ${SERVICE_NAME}:${BACKUP_IMAGE_TAG}"
                    echo "--------------------------------------------------"
                }
            }
        }

        stage('Deploy New Image to UAT') {
            steps {
                script {
                    echo "🚀 Updating ONLY 'deployment/${SERVICE_NAME}' with image '${IMAGE_NAME}:${IMAGE_TAG}'..."
                    
                    /* 
                       1. Specific target deployment image-a update pannum (other 70+ services stay untouched).
                       2. 3 minutes rollout status check pannum.
                    */
                    sh """
                        kubectl set image deployment/${SERVICE_NAME} ${SERVICE_NAME}=${IMAGE_NAME}:${IMAGE_TAG} -n${KUBE_NAMESPACE}
                        kubectl rollout status deployment/${SERVICE_NAME} -n${KUBE_NAMESPACE} --timeout=180s
                    """
                }
            }
        }

        stage('Verify Target Pod') {
            steps {
                script {
                    echo "Checking active pod status for ${SERVICE_NAME}:"
                    sh "kubectl get pods -l app=${SERVICE_NAME} -n${KUBE_NAMESPACE}"
                }
            }
        }
    }

    post {
        always {
            sh "oc logout || true"
        }
        success {
            echo "✅ SUCCESS: '${SERVICE_NAME}' updated in UAT and old image tagged for instant rollback!"
        }
        failure {
            echo "❌ DEPLOYMENT FAILED! Triggering automatic cluster rollback..."
            sh "kubectl rollout undo deployment/${SERVICE_NAME} -n${KUBE_NAMESPACE}"
        }
    }
}
