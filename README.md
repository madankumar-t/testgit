
 # Helm Chart Installation and Deployment for Facade Service

This guide walks you through the steps to install Helm, create a Helm chart for the facade service, package and publish it to an Amazon ECR repository, and deploy it to an EKS cluster.

## Prerequisites
- Amazon Linux instance
- AWS CLI configured with the required IAM permissions
- kubectl installed and configured to interact with an EKS cluster
- Helm installed on your local machine

## 1. Install Helm on Amazon Linux

To install Helm, follow these steps:

```bash
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh

Verify Helm Installation:
$ helm version

##** 2. Create Helm Chart for Facade Service**
Navigate to the Helm charts directory:
$ cd core-architecture/cloud-lib/helm-charts

Create a new Helm chart for the facade service:
$ helm create sandbox_node_facade
$ cd sandbox_node_facade


This will generate the following file structure:
$ tree sandbox_node_facade
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

**3. Update the Files as Required**
Modify the following files as required for your service:
values.yaml
Chart.yaml
deployment.yaml
service.yaml

**4. Package the Helm Chart**
Once your modifications are complete, package the Helm chart:
$ cd ..
$ helm package sandbox_node_facade
This command should create a file: sandbox_node_facade-0.1.0.tgz.

**5. Authenticate to ECR (for Helm OCI)**
Run the following command to authenticate to Amazon ECR:
$ aws ecr get-login-password | helm registry login --username AWS --password-stdin 977099017816.dkr.ecr.us-east-1.amazonaws.com

**6. Publish Chart to ECR Helm Repo**
Publish the Helm chart to your ECR repository:
$ helm push sandbox_node_facade-0.1.0.tgz oci://977099017816.dkr.ecr.us-east-1.amazonaws.com/core-helm

**7. Install Helm Chart from ECR into EKS**
Deploy the service into your EKS cluster using the following command:
$ helm install sandbox_node_facade oci://977099017816.dkr.ecr.us-east-1.amazonaws.com/core-helm/facade --version 0.1.0 --namespace default

**8. Verify Deployment**
Check the status of the deployment:
$ kubectl get pods -n default
$ kubectl get svc -n default

**9. Upgrade or Uninstall**
To upgrade or uninstall the Helm release:
**Upgrade:**
$ helm upgrade sandbox_node_facade oci://977099017816.dkr.ecr.us-east-1.amazonaws.com/core-helm/sandbox_node_facade --version 0.1.0 --namespace default

**Uninstall:**
$ helm uninstall sandbox_node_facade --namespace default


**10. Post Installation**
After installation, verify the pod status, pod logs, and the service details in the EKS cluster.

**11. Option 1: Installing Helm Charts with Environment Variables Defined**
You can pass values using the --set flag:

$ helm install sandbox_java_service ./sandbox_java_service \
  --namespace your-namespace \
  --set env.SPRING_PROFILE=prod

This can also be used with helm upgrade:
$ helm upgrade sandbox_java_service ./sandbox_java_service \
  --set env.SPRING_PROFILE=prod

**12. Option 2: Use a Separate Custom values-prod.yaml File**
Create a values-prod.yaml file like:
env:
  SPRING_PROFILE: prod

Then deploy using:
$ helm install sandbox_java_service ./sandbox_java_service -f values-prod.yaml --namespace your-namespace

Or upgrade:
$ helm upgrade sandbox_java_service ./sandbox_java_service -f values-prod.yaml --namespace your-namespace

Note: Ensure to replace placeholders like your-namespace with actual values specific to your environment.
      Above details are only example, please update your actual values while working. 




