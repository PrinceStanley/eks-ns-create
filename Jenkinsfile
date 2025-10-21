pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp # Jenkins JNLP Agent
      image: jenkins/inbound-agent:latest # Or a specific version like '4.13.2-1'
      env:
        # Replace with your Jenkins service URL that agents can reach
        - name: JENKINS_URL
          value: "http://jenkins.jenkins.svc.cluster.local:8080"
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent/workspace
    - name: aws
      image: 643208527630.dkr.ecr.us-east-1.amazonaws.com/jenkins-agent:latest
      command:
        - cat
      tty: true
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent/workspace
  serviceAccountName: jenkins-agent
  volumes:
    - name: workspace-volume
      emptyDir: {}
"""
        }
    }

    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS Region for ECR and EKS')
        string(name: 'CLUSTER_NAME', defaultValue: 'uc-devops-eks-cluster', description: 'EKS Cluster Name')
        string(name: 'NAMESPACE_NAME', defaultValue: 'app', description: 'Kubernetes Namespace for Deployment')
    }

    environment {
        AWS_REGION = "${params.AWS_REGION}"
        NAMESPACE_NAME = "${params.NAMESPACE_NAME}"
        CLUSTER_NAME = "${params.CLUSTER_NAME}"
        KUBECONFIG_FILE = "/home/jenkins/agent/workspace/kubeconfig-${params.NAMESPACE_NAME}.yaml"
    }

    stages {
        stage('Checkout') {
            steps {
                container('jnlp') {
                    script {
                        echo "Checking out code from SCM..."
                        checkout scm
                        echo "Code checked out successfully."
                    }
                }
            }
        }

        stage('Create Namespace in the EKS cluster') {
            steps {
                container('aws') {
                    script {
                        echo "Creating Kubernetes namespace: \${NAMESPACE_NAME}"
                        sh """
                            KUBECONFIG=/home/jenkins/agent/workspace/.kube/config aws eks update-kubeconfig --name \${CLUSTER_NAME} --region \${AWS_REGION}
                            kubectl create namespace \${NAMESPACE_NAME} --dry-run=client -o yaml | kubectl apply -f -
                            echo "Namespace \${NAMESPACE_NAME} created successfully."
                        """
                    }
                }
            }
        }

        stage('Create Service Account, Role, and RoleBinding') {
            steps {
                container('aws') {
                    script {
                         sh """
                            KUBECONFIG=/home/jenkins/agent/workspace/.kube/config aws eks update-kubeconfig --name \${CLUSTER_NAME} --region \${AWS_REGION}
                            sed -i 's/NAMESPACE_NAME/\${NAMESPACE_NAME}/g' 1-ns-sa.yaml
                            kubectl apply -f 1-ns-sa.yaml
                            sed -i 's/NAMESPACE_NAME/\${NAMESPACE_NAME}/g' 2-ns-role.yaml
                            kubectl apply -f 2-ns-role.yaml
                            sed -i 's/NAMESPACE_NAME/\${NAMESPACE_NAME}/g' 3-ns-rolebinding.yaml
                            kubectl apply -f 3-ns-rolebinding.yaml
                            echo "ServiceAccount, Role, and RoleBinding created successfully."
                        """
                    }
                }
            }
        }

        stage('Generate Token and Kubeconfig') {
            steps {
                container('aws') {
                    script {
                        sh """
                            KUBECONFIG=/home/jenkins/agent/workspace/.kube/config aws eks update-kubeconfig --name \${CLUSTER_NAME} --region \${AWS_REGION}
                            echo "Generating token for ServiceAccount..."
                            TOKEN="\$(kubectl create token \${NAMESPACE_NAME}-user -n \${NAMESPACE_NAME} --duration=99999h)"

                            echo "Fetching EKS cluster details..."
                            ENDPOINT="\$(aws eks describe-cluster --name \${CLUSTER_NAME} --region \${AWS_REGION} --query "cluster.endpoint" --output text)"
                            CERT="\$(aws eks describe-cluster --name \${CLUSTER_NAME} --region \${AWS_REGION} --query "cluster.certificateAuthority.data" --output text)"

                            echo "Building kubeconfig file..."
                    cat > \${KUBECONFIG_FILE} <<EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: \${CERT}
    server: \${ENDPOINT}
  name: eks-cluster
contexts:
- context:
    cluster: eks-cluster
    namespace: \${NAMESPACE_NAME}
    user: \${NAMESPACE_NAME}-user
  name: \${NAMESPACE_NAME}-context
current-context: \${NAMESPACE_NAME}-context
users:
- name: \${NAMESPACE_NAME}-user
  user:
    token: \${TOKEN}
EOF
                    echo "✅ Kubeconfig generated: \${KUBECONFIG_FILE}"
                    """
                    }
                }
            }
        }
    }


    post {
        failure {
            script {
                currentBuild.result = 'FAILURE'
                echo "Build failed. Please check the logs for more details."
            }
        }
        success {
            script {
                currentBuild.result = 'SUCCESS'
                echo "Build and deployment succeeded."
            }
        }
    }
}