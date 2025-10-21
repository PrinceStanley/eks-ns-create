# eks-ns-create

Create a Kubernetes namespace in an EKS cluster and provision a ServiceAccount, Role, and RoleBinding for that namespace. The pipeline also generates a kubeconfig scoped to the created ServiceAccount and uploads it to AWS Secrets Manager.

Highlights
- Pipeline defined in [Jenkinsfile](Jenkinsfile)
- Namespace ServiceAccount template: [1-ns-sa.yaml](1-ns-sa.yaml)
- Role template: [2-ns-role.yaml](2-ns-role.yaml)
- RoleBinding template: [3-ns-rolebinding.yaml](3-ns-rolebinding.yaml)

Prerequisites
- Jenkins with the Kubernetes plugin and an agent that can run AWS CLI and kubectl.
- IAM permissions for the Jenkins agent to call EKS (describe/update-kubeconfig) and Secrets Manager (create/update secret).
- AWS CLI and kubectl available in the agent container used by the pipeline.

Pipeline parameters (see [Jenkinsfile](Jenkinsfile))
- [`params.AWS_REGION`](Jenkinsfile) — AWS Region (default: us-east-1)
- [`params.CLUSTER_NAME`](Jenkinsfile) — EKS cluster name
- [`params.NAMESPACE_NAME`](Jenkinsfile) — Kubernetes namespace to create
- Environment variable storing the output kubeconfig: [`KUBECONFIG_FILE`](Jenkinsfile)

What the pipeline does (stages)
1. Checkout — checks out the repository.
2. Create Namespace in the EKS cluster — updates kubeconfig for the cluster and creates the namespace.
3. Create Service Account, Role, and RoleBinding — substitutes `NAMESPACE_NAME` into the templates ([1-ns-sa.yaml](1-ns-sa.yaml), [2-ns-role.yaml](2-ns-role.yaml), [3-ns-rolebinding.yaml](3-ns-rolebinding.yaml)) and applies them.
4. Generate Token and Kubeconfig — creates a token for the ServiceAccount and builds a kubeconfig file at the path set by [`KUBECONFIG_FILE`](Jenkinsfile).
5. Upload Kubeconfig to AWS Secret Manager — uploads the generated kubeconfig to Secrets Manager under the name `eks/<CLUSTER_NAME>/<NAMESPACE_NAME>/kubeconfig`.

Important security note
- The Role in [2-ns-role.yaml](2-ns-role.yaml) currently grants wildcards for apiGroups/resources/verbs, which effectively allows full access within the namespace. Review and tighten RBAC rules before using in production.

Usage
- Trigger the pipeline from Jenkins and supply/accept the parameters (`AWS_REGION`, `CLUSTER_NAME`, `NAMESPACE_NAME`).
- After success, the kubeconfig is stored in AWS Secrets Manager at `eks/<CLUSTER_NAME>/<NAMESPACE_NAME>/kubeconfig`.

Troubleshooting
- Ensure Jenkins agent has AWS credentials and network access to the EKS cluster API.
- If `kubectl create token` fails, verify kubectl version and cluster RBAC configuration.
- Inspect the Jenkins logs for the `aws` container output in the pipeline run (agent logs).

Files
- [Jenkinsfile](Jenkinsfile)
- [1-ns-sa.yaml](1-ns-sa.yaml)
- [2-ns-role.yaml](2-ns-role.yaml)
- [3-ns-rolebinding.yaml](3-ns-rolebinding.yaml)
- [README.md](README.md)