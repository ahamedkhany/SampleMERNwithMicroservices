# MERN Microservices — AWS EKS Container Orchestration

## 1. Project Overview

This project demonstrates the containerization, CI/CD automation, deployment, monitoring, logging, secret management, ingress routing, and autoscaling of a MERN-based microservices application on **Amazon Web Services (AWS)** using:

* Docker
* Amazon ECR
* Jenkins
* Amazon EKS
* Kubernetes
* AWS Application Load Balancer (ALB)
* AWS Secrets Manager
* EKS Pod Identity
* Secrets Store CSI Driver
* Amazon CloudWatch / Container Insights
* Kubernetes Metrics Server
* Horizontal Pod Autoscaler (HPA)
* MongoDB Atlas

The application consists of three services:

1. **Frontend** — React application served through Nginx
2. **Hello Service** — Node.js/Express microservice
3. **Profile Service** — Node.js/Express + MongoDB microservice

---

# 2. Architecture

```text
                              Internet
                                  |
                                  v
                       AWS Application Load Balancer
                                  |
                         Kubernetes Ingress
                                  |
              +-------------------+-------------------+
              |                   |                   |
              v                   v                   v
            /hello         /profile/fetchUser         /
              |                fetchUser              |
              v                   |                   v
       Hello Service              |                Frontend
          :3001                   v                  :80
                           Profile Service
                                :3002
                                  |
                                  v
                             MongoDB Atlas
```

### AWS / Kubernetes Architecture

```text
                         AWS
                          |
                +---------+---------+
                |       EKS        |
                |                  |
                |  3/4 Worker Nodes|
                |                  |
                |  +------------+  |
                |  | Frontend   |  |
                |  | 2-4 Pods   |  |
                |  +------------+  |
                |                  |
                |  +------------+  |
                |  | Hello      |  |
                |  | 2-4 Pods   |  |
                |  +------------+  |
                |                  |
                |  +------------+  |
                |  | Profile    |  |
                |  | 2-4 Pods   |  |
                |  +------------+  |
                |                  |
                +------------------+
                          |
                     ALB Ingress
                          |
                       Internet
```

---

# 3. Application Components

## Frontend

Technology:

* React
* Axios
* Nginx

The frontend calls:

```text
/hello/
```

and:

```text
/profile/fetchUser
```

through the ALB Ingress.

---

## Hello Service

Technology:

* Node.js
* Express

Port:

```text
3001
```

Endpoint:

```text
GET /hello
```

Example response:

```json
{
  "msg": "Hello World"
}
```

---

## Profile Service

Technology:

* Node.js
* Express
* Mongoose
* MongoDB Atlas

Port:

```text
3002
```

Endpoint:

```text
GET /profile/fetchUser
```

The service retrieves users from MongoDB Atlas.

---

# 4. Repository Structure

The project follows a structure similar to:

```text
SampleMERNwithMicroservices/
│
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       └── Home.js
│
├── helloService/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js
│
├── profileService/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js
│
├── infrastructure/
│   └── eks/
│       │ 
│       ├── cluster.yaml
│       ├── cloudwatch-trust-policy.json
│       ├── iam_policy.json
│       ├── pod-identity-trust-policy.json
│       ├── secrets-manager-policy.json
│       └── manifests/
│           ├── frontend-deployment.yaml
│           ├── hello-deployment.yaml
│           ├── profile-deployment.yaml
│           ├── frontend-service.yaml
│           ├── hello-service.yaml
│           ├── profile-service.yaml
│           ├── ingress.yaml
│           ├── frontend-hpa.yaml
│           ├── hello-service-hpa.yaml
│           ├── profile-service-hpa.yaml
│           └── profile-secrets.yaml (SecretProviderClass)
│       
├── .gitignore  
├── README.md    
└── Jenkinsfile
```

---

# 5. Prerequisites

Install/configure the following:

* AWS CLI
* kubectl
* eksctl
* Docker
* Jenkins
* Git
* Node.js
* npm
* MongoDB Atlas account

Verify:

```powershell
aws --version
kubectl version --client
eksctl version
docker --version
git --version
node --version
npm --version
```

---

# 6. AWS Configuration

AWS Region used:

```text
us-east-1
```

Verify AWS credentials:

```powershell
aws sts get-caller-identity
```

Example account:

```text
533612070969
```

---

# 7. Docker Containerization

Each application component was containerized separately.

## Frontend Docker Image

The frontend was built into a Docker image and served through Nginx.

## Hello Service Docker Image

The Hello Service was containerized and exposed on:

```text
3001
```

## Profile Service Docker Image

The Profile Service was containerized and exposed on:

```text
3002
```

---

# 8. Local Docker Testing

Before pushing images to ECR, each service was tested locally.

Example:

```powershell
docker build -t hello-service .
docker build -t profile-service .
docker build -t frontend .
```

Containers were started and their endpoints were tested.

This ensured that application-level problems were separated from Kubernetes/AWS problems.

---

# 9. Amazon ECR

Three ECR repositories were created:

```text
streaming-frontend
streaming-hello-service
streaming-profile-service
```

Region:

```text
us-east-1
```

Example ECR registry:

```text
533612070969.dkr.ecr.us-east-1.amazonaws.com
```

Authenticate Docker with ECR:

```powershell
aws ecr get-login-password --region us-east-1 |
docker login --username AWS --password-stdin 533612070969.dkr.ecr.us-east-1.amazonaws.com
```

Images were tagged and pushed to ECR.

Example:

```powershell
docker tag hello-service:1 `
  533612070969.dkr.ecr.us-east-1.amazonaws.com/streaming-hello-service:1
```

Push:

```powershell
docker push `
  533612070969.dkr.ecr.us-east-1.amazonaws.com/streaming-hello-service:1
```

The same approach was followed for the frontend and profile service.

---

# 10. Jenkins CI/CD

Jenkins was deployed using Docker.

The Jenkins container was configured with:

* Jenkins home volume
* Docker socket
* Docker CLI
* AWS CLI
* Git

Jenkins was able to execute:

```powershell
docker version
aws --version
git --version
aws sts get-caller-identity
```

The Jenkins pipeline was configured to:

1. Checkout source code
2. Build Docker images
3. Authenticate with AWS
4. Login to ECR
5. Build service images
6. Tag images
7. Push images to ECR

---

# 11. Jenkins Build Versioning

Each successful Jenkins build produced a new image version.

Example:

```text
streaming-frontend:<BUILD_NUMBER>
streaming-hello-service:<BUILD_NUMBER>
streaming-profile-service:<BUILD_NUMBER>
```

The corresponding image version was updated in the Kubernetes Deployment manifests before deploying the new version.

This allowed the deployment to use the exact image produced by Jenkins.

---

# 12. Amazon EKS Cluster

EKS cluster:

```text
container-orch-cluster
```

Region:

```text
us-east-1
```

Node group:

```text
container-orch-nodes
```

The node group was configured with:

```text
Minimum: 2
Desired: 4
Maximum: 4
```

The cluster was eventually expanded to four nodes to provide sufficient Pod capacity for the application and monitoring components.

Check cluster:

```powershell
kubectl get nodes
```

Example:

```text
NAME                            STATUS   ROLES    VERSION
ip-192-168-32-95.ec2.internal   Ready    <none>   v1.34.x
ip-192-168-63-54.ec2.internal   Ready    <none>   v1.34.x
ip-192-168-7-170.ec2.internal   Ready    <none>   v1.34.x
...
```

---

# 13. EKS Add-ons

The following EKS add-ons were configured:

```text
coredns
eks-pod-identity-agent
kube-proxy
metrics-server
vpc-cni
amazon-cloudwatch-observability
```

Check add-ons:

```powershell
eksctl get addon `
  --cluster container-orch-cluster `
  --region us-east-1
```

Expected status:

```text
ACTIVE
```

for the required add-ons.

---

# 14. Kubernetes Deployments

Each application was deployed using a Kubernetes Deployment.

## Frontend

```text
Deployment: frontend
Port: 80
Replicas: 2
```

## Hello Service

```text
Deployment: hello-service
Port: 3001
Replicas: 2
```

## Profile Service

```text
Deployment: profile-service
Port: 3002
Replicas: 2
```

Check:

```powershell
kubectl get deployments
kubectl get pods
```

---

# 15. Kubernetes Services

Three ClusterIP services were created:

```text
frontend
hello-service
profile-service
```

Check:

```powershell
kubectl get services
```

The services provide stable internal networking between Kubernetes workloads.

---

# 16. MongoDB Atlas

The Profile Service uses MongoDB Atlas.

The MongoDB connection string is stored as an AWS Secrets Manager secret instead of being hard-coded into the application or Kubernetes manifest.

MongoDB Atlas Network Access was configured to allow the IP address used by the EKS workload to reach the cluster.

The application was successfully tested with:

```text
MongoDB connected successfully
```

---

# 17. AWS Secrets Manager

Secret created:

```text
streaming/profile-service/mongo-url
```

The MongoDB connection URL is stored securely in AWS Secrets Manager.

The application does not store the actual MongoDB connection string in GitHub.

---

# 18. EKS Pod Identity

EKS Pod Identity was used to allow the Profile Service Pod to access the required AWS secret.

Service Account:

```text
profile-service
```

Namespace:

```text
default
```

IAM Role:

```text
StreamingProfileServiceSecretsRole
```

Verify the association:

```powershell
aws eks list-pod-identity-associations `
  --cluster-name container-orch-cluster `
  --region us-east-1
```

The association maps:

```text
default/profile-service
        |
        v
StreamingProfileServiceSecretsRole
```

---

# 19. IAM Trust Policy

The IAM role uses the EKS Pod Identity principal:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "pods.eks.amazonaws.com"
  },
  "Action": [
    "sts:AssumeRole",
    "sts:TagSession"
  ]
}
```

This allows EKS Pods to assume the IAM role through EKS Pod Identity.

---

# 20. IAM Permissions

The IAM policy attached to the role allows:

```text
secretsmanager:GetSecretValue
secretsmanager:DescribeSecret
```

for the specific MongoDB secret.

The permission is scoped to:

```text
streaming/profile-service/mongo-url
```

rather than granting unrestricted Secrets Manager access.

---

# 21. Secrets Store CSI Driver

The following components were installed:

```text
Secrets Store CSI Driver
AWS Secrets Manager / Parameter Store Provider
```

Verify:

```powershell
kubectl get pods -n kube-system
```

The following components should be running:

```text
csi-secrets-store
secrets-store-csi-driver-provider-aws
```

---

# 22. SecretProviderClass

A Kubernetes `SecretProviderClass` was configured to retrieve:

```text
streaming/profile-service/mongo-url
```

from AWS Secrets Manager.

The secret is mounted into the Profile Service Pod and synchronized into:

```text
profile-service-secret
```

The application consumes:

```text
MONGO_URL
```

through the Kubernetes Secret.

---

# 23. Profile Service Secret Flow

The complete flow is:

```text
AWS Secrets Manager
        |
        | GetSecretValue
        v
EKS Pod Identity
        |
        v
IAM Role
StreamingProfileServiceSecretsRole
        |
        v
Secrets Store CSI Driver
        |
        v
SecretProviderClass
        |
        v
Kubernetes Secret
profile-service-secret
        |
        v
Profile Service
MONGO_URL
        |
        v
MongoDB Atlas
```

Verify:

```powershell
kubectl exec deployment/profile-service -- printenv MONGO_URL
```

The application was successfully able to connect to MongoDB Atlas.

---

# 24. AWS Load Balancer Controller

AWS Load Balancer Controller was installed in the EKS cluster.

Verify:

```powershell
kubectl get pods -n kube-system | findstr aws-load-balancer
```

Two controller Pods were running successfully.

The controller manages AWS Application Load Balancers for Kubernetes Ingress resources.

---

# 25. ALB Ingress

An AWS ALB Ingress was configured to expose the application.

Ingress:

```text
streaming-ingress
```

Ingress class:

```text
alb
```

Scheme:

```text
internet-facing
```

Target type:

```text
ip
```

Routing:

```text
/hello
    |
    v
hello-service:3001

/profile
    |
    v
profile-service:3002

/
    |
    v
frontend:80
```

Check:

```powershell
kubectl get ingress
```

The ALB DNS name was generated automatically by AWS.

---

# 26. Final Application Routes

The application was successfully tested using the ALB DNS name.

### Frontend

```text
http://<ALB-DNS>/
```

### Hello Service

```text
http://<ALB-DNS>/hello
```

### Profile Service

```text
http://<ALB-DNS>/profile/fetchUser
```

All three URLs were successfully tested.

---

# 27. Important Application Routing Fix

Initially, the application routes did not match the ALB paths.

The final application endpoints were aligned with the Ingress:

### Hello Service

```javascript
app.get('/hello', ...)
```

### Profile Service

```javascript
app.get('/profile/fetchUser', ...)
```

Frontend:

```javascript
axios.get("hello/")
```

and:

```javascript
axios.get("profile/fetchUser")
```

This allowed the browser requests to match the ALB Ingress paths correctly.

---

# 28. CloudWatch Monitoring

Amazon CloudWatch Container Insights was enabled using the:

```text
amazon-cloudwatch-observability
```

EKS add-on.

Check:

```powershell
eksctl get addon `
  --cluster container-orch-cluster `
  --region us-east-1
```

CloudWatch components:

```text
cloudwatch-agent
fluent-bit
amazon-cloudwatch-observability-controller-manager
```

Verify:

```powershell
kubectl get pods -n amazon-cloudwatch
```

---

# 29. CloudWatch Application Logs

CloudWatch Container Insights created the following log groups:

```text
/aws/containerinsights/container-orch-cluster/application
/aws/containerinsights/container-orch-cluster/dataplane
/aws/containerinsights/container-orch-cluster/host
/aws/containerinsights/container-orch-cluster/performance
```

Application logs are centralized in:

```text
/aws/containerinsights/container-orch-cluster/application
```

Verify:

```powershell
aws logs describe-log-groups `
  --region us-east-1 `
  --query "logGroups[].logGroupName" `
  --output table
```

---

# 30. Application Logging

Application-level logging was added to the services.

For example:

```javascript
console.log("Hello endpoint called");
```

Kubernetes captures the container's stdout/stderr.

The logging flow is:

```text
Application
    |
    v
Container stdout/stderr
    |
    v
Kubernetes node
    |
    v
Fluent Bit
    |
    v
CloudWatch Logs
```

Logs can be viewed using AWS CLI or the CloudWatch console.

Example:

```powershell
aws logs describe-log-streams `
  --log-group-name "/aws/containerinsights/container-orch-cluster/application" `
  --region us-east-1 `
  --query "logStreams[?contains(logStreamName, 'hello-service')].logStreamName" `
  --output table
```

---

# 31. Metrics Server

Metrics Server was already available in the EKS cluster.

Verify:

```powershell
kubectl get pods -n kube-system | findstr metrics-server
```

Test:

```powershell
kubectl top pods
```

Example:

```text
NAME                         CPU(cores)   MEMORY(bytes)
frontend-...                 1m           7Mi
hello-service-...            8m           82Mi
profile-service-...          11m          97Mi
```

This confirms that Kubernetes can retrieve resource metrics.

---

# 32. Horizontal Pod Autoscaler

HPA was configured for all three application services.

Configuration:

```text
Minimum replicas: 2
Maximum replicas: 4
CPU target: 70%
```

HPAs:

```text
frontend-hpa
hello-service-hpa
profile-service-hpa
```

Check:

```powershell
kubectl get hpa
```

Expected structure:

```text
NAME                  MINPODS   MAXPODS   REPLICAS
frontend-hpa          2         4         2
hello-service-hpa     2         4         2
profile-service-hpa   2         4         2
```

---

# 33. CPU Resource Requests

HPA CPU utilization requires CPU requests to be defined in the Deployment.

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "250m"
    memory: "128Mi"
```

Profile Service was given a slightly higher memory allocation because it communicates with MongoDB.

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Without CPU requests, HPA reports:

```text
cpu: <unknown>/70%
```

and cannot calculate CPU utilization.

---

# 34. HPA Scaling Model

The final scaling model is:

```text
                   HPA
                    |
             CPU utilization
                    |
             Target = 70%
                    |
       +------------+------------+
       |            |            |
   Frontend       Hello        Profile
    2-4 Pods      2-4 Pods      2-4 Pods
```

HPA can increase the number of application Pods when CPU utilization exceeds the configured target.

---

# 35. Node Capacity

The EKS nodes were configured with sufficient capacity for the application and supporting workloads.

Pod capacity was checked using:

```powershell
kubectl get nodes -o custom-columns="NODE:.metadata.name,PODS:.status.allocatable.pods"
```

During the initial configuration, the cluster had:

```text
2 nodes
```

with approximately:

```text
11 Pods per node
```

available for scheduling.

The cluster was expanded to:

```text
4 nodes
```

to provide additional capacity.

---

# 36. Final Kubernetes Validation

Check all application Pods:

```powershell
kubectl get pods
```

Check Deployments:

```powershell
kubectl get deployments
```

Check Services:

```powershell
kubectl get services
```

Check Ingress:

```powershell
kubectl get ingress
```

Check HPA:

```powershell
kubectl get hpa
```

Check node status:

```powershell
kubectl get nodes
```

Check metrics:

```powershell
kubectl top pods
```

---

# 37. Final Application Validation

The following URLs were tested successfully:

```text
http://<ALB-DNS>/
```

```text
http://<ALB-DNS>/hello
```

```text
http://<ALB-DNS>/profile/fetchUser
```

Validation results:

```text
Frontend                ✅ Working
Hello Service            ✅ Working
Profile Service          ✅ Working
MongoDB Atlas            ✅ Connected
ALB Ingress              ✅ Working
AWS Secrets Manager      ✅ Working
EKS Pod Identity         ✅ Working
CloudWatch Logs          ✅ Working
Metrics Server           ✅ Working
HPA                      ✅ Working
```

---

# 38. Major Issues Faced and Resolutions

## Issue 1 — EKS Pod Identity Association Not Working

### Problem

Profile Service received:

```text
An IAM role must be associated with service account profile-service
```

### Cause

The Kubernetes ServiceAccount and EKS Pod Identity association were not correctly aligned.

### Resolution

Verified:

```text
namespace: default
serviceAccount: profile-service
```

and created/validated the association with:

```text
StreamingProfileServiceSecretsRole
```

Verification:

```powershell
aws eks list-pod-identity-associations `
  --cluster-name container-orch-cluster `
  --region us-east-1
```

---

## Issue 2 — Incorrect IAM Trust Policy

### Problem

The Pod needed to assume the IAM role.

### Resolution

The IAM role trust relationship was configured with:

```json
"Principal": {
  "Service": "pods.eks.amazonaws.com"
}
```

and:

```json
"Action": [
  "sts:AssumeRole",
  "sts:TagSession"
]
```

---

## Issue 3 — Secrets Store Object Alias Error

### Problem

The CSI driver reported:

```text
file matching objectName streaming/profile-service/mongo-url not found in the pod
```

### Cause

The `secretObjects.data.objectName` value did not match the mounted object name.

### Resolution

The SecretProviderClass was corrected so that the mounted secret object and the Kubernetes Secret mapping used the same expected object name.

This allowed:

```text
profile-service-secret
```

to be created successfully.

---

## Issue 4 — MongoDB Atlas Connection Failed

### Problem

Profile Service reported:

```text
MongoDB connection failed:
Could not connect to any servers in your MongoDB Atlas cluster.
```

### Cause

The network source IP used by the workload was not allowed by MongoDB Atlas Network Access.

### Resolution

The appropriate IP address was added to the MongoDB Atlas IP access list.

After correction:

```text
MongoDB connected successfully
```

---

## Issue 5 — ALB `/hello` Returned 502

### Problem

The ALB returned:

```text
502 Bad Gateway
```

for:

```text
/hello
```

### Cause

The application was listening on the wrong/undefined port because the expected environment configuration was missing.

Logs initially showed:

```text
Server is running on port undefined
```

### Resolution

The Hello Service was corrected to listen on:

```text
3001
```

and the Deployment was updated accordingly.

Final log:

```text
Server is running on port 3001
```

---

## Issue 6 — `/hello` Returned `Cannot GET /hello`

### Problem

The application was reachable, but:

```text
/hello
```

returned:

```text
Cannot GET /hello
```

### Cause

The Express route did not match the Ingress route.

### Resolution

The Express endpoint was changed to:

```javascript
app.get('/hello', ...)
```

The application was rebuilt and redeployed.

---

## Issue 7 — `/profile/fetchUser` Returned `Cannot GET`

### Problem

The ALB path was:

```text
/profile/fetchUser
```

but the Express application exposed:

```text
/fetchUser
```

### Resolution

The Profile Service route was changed to:

```javascript
app.get('/profile/fetchUser', ...)
```

The frontend was also configured to call:

```text
profile/fetchUser
```

The new image was built and deployed.

---

## Issue 8 — Frontend Service Not Found

### Problem

The Ingress initially showed:

```text
<error: services "frontend" not found>
```

### Cause

The Ingress referenced a Service that had not been created or had an incorrect name.

### Resolution

The frontend Kubernetes Service was created/corrected with the expected name:

```text
frontend
```

The ALB was then successfully reconciled.

---

## Issue 9 — Test Curl Pod Could Not Be Scheduled

### Problem

A temporary test Pod using:

```text
curlimages/curl
```

could not be scheduled and eventually timed out.

### Cause

The worker nodes had reached their Pod scheduling capacity.

### Resolution

Node capacity was inspected:

```powershell
kubectl get nodes -o custom-columns="NODE:.metadata.name,PODS:.status.allocatable.pods"
```

The cluster was expanded from two nodes to four nodes.

---

## Issue 10 — Pods Stuck in Pending State

### Problem

Pods showed:

```text
Pending
```

with:

```text
0/3 nodes are available:
3 Too many pods
```

### Cause

The nodes had reached their maximum Pod capacity.

### Resolution

The EKS node group's desired/max capacity was increased.

After adding additional capacity:

```text
All application Pods → Running
```

---

## Issue 11 — CloudWatch Observability Add-on Degraded

### Problem

The CloudWatch add-on showed:

```text
DEGRADED
```

and the controller Pod was:

```text
Pending
```

### Cause

There was insufficient Pod capacity on the two-node cluster.

### Resolution

The node group was scaled up.

After adding the additional node capacity:

```text
amazon-cloudwatch-observability-controller-manager
1/1 Running
```

and:

```text
cloudwatch-agent
3/3 Running
```

---

## Issue 12 — HPA Showed `<unknown>/70%`

### Problem

HPA showed:

```text
cpu: <unknown>/70%
```

### Cause

Metrics Server was working, but the application containers did not have CPU requests.

HPA reported:

```text
missing request for cpu in container frontend
```

### Resolution

CPU requests were added to the Deployments.

Example:

```yaml
resources:
  requests:
    cpu: "100m"
```

After redeployment, HPA was able to calculate CPU utilization correctly.

---

# 39. Useful Troubleshooting Commands

## Pods

```powershell
kubectl get pods
kubectl get pods -A
kubectl get pods -o wide
```

## Pod logs

```powershell
kubectl logs deployment/frontend
kubectl logs deployment/hello-service
kubectl logs deployment/profile-service
```

For a specific Pod:

```powershell
kubectl logs <pod-name>
```

## Deployment status

```powershell
kubectl rollout status deployment/frontend
kubectl rollout status deployment/hello-service
kubectl rollout status deployment/profile-service
```

## Services

```powershell
kubectl get services
kubectl get endpoints
kubectl get endpointslices
```

## Ingress

```powershell
kubectl get ingress
kubectl describe ingress streaming-ingress
```

## HPA

```powershell
kubectl get hpa
kubectl describe hpa frontend-hpa
```

## Metrics

```powershell
kubectl top nodes
kubectl top pods
```

## Nodes

```powershell
kubectl get nodes
kubectl describe node <node-name>
```

## CloudWatch

```powershell
aws logs describe-log-groups `
  --region us-east-1 `
  --output table
```

## EKS Pod Identity

```powershell
aws eks list-pod-identity-associations `
  --cluster-name container-orch-cluster `
  --region us-east-1
```

---

# 40. Final Deployment Flow

The complete deployment process is:

```text
Developer
    |
    v
GitHub Repository
    |
    v
Jenkins Pipeline
    |
    +---- Build Frontend Image
    |
    +---- Build Hello Image
    |
    +---- Build Profile Image
    |
    v
Amazon ECR
    |
    v
Kubernetes Deployment
    |
    v
Amazon EKS
    |
    +------------------+
    |                  |
    v                  v
Kubernetes Services   HPA
    |                  |
    v                  v
AWS ALB Ingress    2 → 4 Pods
    |
    v
Internet
```

For secrets:

```text
AWS Secrets Manager
        |
        v
EKS Pod Identity
        |
        v
IAM Role
        |
        v
CSI Driver
        |
        v
Profile Service
        |
        v
MongoDB Atlas
```

For monitoring:

```text
Application
    |
    v
Container stdout/stderr
    |
    v
Fluent Bit
    |
    v
CloudWatch Logs

Metrics Server
    |
    v
HPA
    |
    v
Application Scaling
```

---

# 41. Final Validation Checklist

* [x] Source code available in GitHub
* [x] Dockerfiles created
* [x] Images built successfully
* [x] Images pushed to Amazon ECR
* [x] Jenkins CI pipeline configured
* [x] Jenkins successfully builds application images
* [x] EKS cluster created
* [x] EKS worker nodes configured
* [x] Kubernetes Deployments created
* [x] Kubernetes Services created
* [x] AWS Load Balancer Controller configured
* [x] ALB Ingress configured
* [x] Frontend accessible
* [x] Hello Service accessible
* [x] Profile Service accessible
* [x] MongoDB Atlas connected successfully
* [x] AWS Secrets Manager configured
* [x] EKS Pod Identity configured
* [x] Secrets Store CSI Driver configured
* [x] CloudWatch Container Insights configured
* [x] Application logs visible in CloudWatch
* [x] Metrics Server working
* [x] CPU requests configured
* [x] HPA configured
* [x] HPA CPU metrics working
* [x] Node capacity increased to support workloads
* [x] Application tested through ALB
* [x] Final validation completed

---

# 42. Future Improvements

The following can be implemented as future enhancements:

* Cluster Autoscaler or Karpenter for automatic node scaling
* HTTPS using AWS Certificate Manager
* Route 53 custom domain
* Network Policies
* Kubernetes Secrets encryption with AWS KMS
* GitHub webhook integration with Jenkins
* Blue/Green or Canary deployments
* Argo CD / GitOps
* Prometheus and Grafana
* Automated integration testing
* Security scanning using Trivy
* Docker image vulnerability scanning
* SNS + Slack/Microsoft Teams ChatOps integration

---

# 43. Bonus — ChatOps Integration

The assignment's optional ChatOps requirement can be implemented using:

```text
Jenkins
   |
   v
Amazon SNS
   |
   +---- Deployment Success
   |
   +---- Deployment Failure
   |
   v
Slack / Microsoft Teams / Telegram
```

SNS topics can be created for:

```text
deployment-success
deployment-failure
```

Notifications can then be forwarded to the desired messaging platform.

---

# 44. Conclusion

This project successfully demonstrates an end-to-end cloud-native deployment of a MERN microservices application on AWS.

The final solution provides:

```text
Containerization
      +
CI/CD
      +
Amazon ECR
      +
Amazon EKS
      +
Kubernetes
      +
ALB Ingress
      +
AWS Secrets Manager
      +
EKS Pod Identity
      +
MongoDB Atlas
      +
CloudWatch
      +
Metrics Server
      +
HPA
```

The application was successfully deployed, exposed through an AWS Application Load Balancer, connected securely to MongoDB Atlas, monitored through CloudWatch, and configured for Kubernetes-based horizontal scaling.

**Final application status: ALL THREE SERVICES WORKING SUCCESSFULLY.**

```text
Frontend                  ✅
Hello Service             ✅
Profile Service           ✅
MongoDB Atlas             ✅
ECR                       ✅
Jenkins CI/CD             ✅
EKS                       ✅
ALB Ingress               ✅
Secrets Manager           ✅
EKS Pod Identity          ✅
CSI Secret Mount          ✅
CloudWatch Logging        ✅
Metrics Server            ✅
HPA                       ✅
Final Validation          ✅
```
