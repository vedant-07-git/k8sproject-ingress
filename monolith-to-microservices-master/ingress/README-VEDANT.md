# K8s Project – Ingress | My Deployment Documentation

This README documents the steps I personally followed to deploy the **Monolith-to-Microservices application** on **Kubernetes using AWS EKS**, expose the application through an **NGINX Ingress Controller**, and deploy the React frontend using my own Docker image.

## Architecture

The project contains the following application components:

- **Frontend microservice** – backend service
- **Products microservice** – product-related APIs
- **Orders microservice** – order-related APIs
- **React frontend** – user-facing web application
- **NGINX Ingress Controller** – routes external traffic to Kubernetes services
- **AWS EKS** – managed Kubernetes cluster
- **Docker Hub** – stores the React frontend image

### Traffic Flow

```text
User / Browser
      |
      v
AWS Load Balancer
      |
      v
NGINX Ingress Controller
      |
      +--------------------+
      |                    |
      v                    v
React Frontend       Backend Services
                           |
                    +------+------+
                    |             |
                    v             v
                 Products       Orders
```

## Environment Used

- **Cloud:** AWS
- **Kubernetes:** Amazon EKS
- **Region:** `ap-south-1`
- **Cluster Name:** `eks-cdec-b71`
- **Kubernetes Version:** `1.35`
- **Node Group:** `linux-nodes`
- **Node Type:** `c7i-flex.large`
- **Initial Nodes:** `1`
- **Ingress:** NGINX Ingress Controller
- **Container Registry:** Docker Hub
- **Docker Image:** `vedantsatpute07/reactfe:v1`

> **Note:** The cluster configuration above reflects the environment I used for this deployment.

## Step 1 – Install eksctl

I installed `eksctl`, which I used to create the EKS cluster.

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

## Step 2 – Install kubectl

I installed the Kubernetes command-line tool `kubectl`.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
kubectl version --client
```

## Step 3 – Install AWS CLI

I installed the AWS CLI so the EC2/Linux environment could interact with AWS services.

```bash
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Before running the EKS commands, AWS credentials/configuration must be available to the environment.

## Step 4 – Create the EKS Cluster

I created the EKS cluster using `eksctl`.

```bash
eksctl create cluster \
  --name eks-cdec-b71 \
  --region ap-south-1 \
  --version 1.35 \
  --nodegroup-name linux-nodes \
  --node-type c7i-flex.large \
  --nodes 1
```

This command created the EKS control plane and a node group with one worker node.

## Step 5 – Configure kubectl for EKS

After the cluster was created, I updated the local kubeconfig so `kubectl` could communicate with the EKS cluster.

```bash
aws eks update-kubeconfig --name eks-cdec-b71 --region ap-south-1
```

## Step 6 – Clone the Project Repository

I cloned the project repository and moved to the Kubernetes manifest directory.

```bash
git clone https://github.com/vedant-07-git/k8sproject-ingress.git
cd k8sproject-ingress/
cd monolith-to-microservices-master/
cd ingress/
ls
```

## Step 7 – Deploy Backend Microservices

I deployed the three backend Kubernetes manifests.

```bash
kubectl apply -f frontend.yml
kubectl apply -f orders.yml
kubectl apply -f products.yml
```

These manifests create the Kubernetes resources required for the backend services.

## Step 8 – Install Docker

I installed Docker so I could build the React frontend image.

```bash
# Add Docker's official GPG key
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Step 9 – Install Helm

I used Helm to install the NGINX Ingress Controller.

During the setup I tried multiple Helm installation methods, including Snap and the Helm installation script.

```bash
sudo snap install helm --classic
```

I also used the official Helm installation script while troubleshooting the installation:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

Then I verified the Kubernetes workloads and ingress namespace.

```bash
kubectl get pod
kubectl get pod --namespace ingress-nginx
```

## Step 10 – Install NGINX Ingress Controller

I added the NGINX Ingress Helm repository and updated it.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Then I installed the NGINX Ingress Controller in the `ingress-nginx` namespace.

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

## Step 11 – Get the Ingress Load Balancer Address

After installing the controller, I checked its Kubernetes service.

```bash
kubectl get svc -n ingress-nginx
```

The NGINX Ingress Controller service exposes an AWS Load Balancer. I used the assigned address for the ingress configuration and frontend environment configuration.

## Step 12 – Configure the Ingress Rule

I pulled the latest repository changes and updated the ingress configuration.

```bash
git pull
```

Then I edited the ingress rule:

```bash
vim ingressrule.yml
```

I updated the `host:` value in `ingressrule.yml` with the Load Balancer address/domain generated for the ingress controller.

After updating the file, I applied it:

```bash
kubectl apply -f ingressrule.yml
```

## Step 13 – Configure and Build the React Frontend

I moved to the React application directory.

```bash
cd ..
cd react-app/
ls
```

I configured the React application's environment so that it could communicate through the ingress endpoint.

Then I built the Docker image using my Docker Hub username.

```bash
docker build -t vedantsatpute07/reactfe:v1 .
```

I verified the local image:

```bash
docker images
```

## Step 14 – Login to Docker Hub and Push Image

I logged in to Docker Hub:

```bash
docker login
```

Then I pushed the React image:

```bash
docker push vedantsatpute07/reactfe:v1
```

The image was then available in Docker Hub for Kubernetes to pull.

## Step 15 – Deploy the React Frontend to Kubernetes

I returned to the ingress directory and applied the React frontend manifest.

```bash
cd ..
cd ingress/
ls
kubectl apply -f reactfe.yml
```

The `reactfe.yml` manifest was updated to use my Docker Hub image:

```text
vedantsatpute07/reactfe:v1
```

## Step 16 – Verify the Deployment

I checked the Kubernetes services and ingress resources.

```bash
kubectl get svc -n ingress-nginx
kubectl get ingress -n ingress-nginx
```

I also checked the pods during deployment:

```bash
kubectl get pod
kubectl get pod -n ingress-nginx
```

## Kubernetes Resources Used

The main project manifests are:

```text
ingress/
├── frontend.yml       # Frontend backend microservice
├── orders.yml         # Orders microservice
├── products.yml       # Products microservice
├── reactfe.yml        # React frontend deployment/service
└── ingressrule.yml    # Ingress routing rules
```

## Namespace

The project resources are deployed in the `ingress-nginx` namespace as described by the project setup.

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingress -n ingress-nginx
```

## Complete Deployment Flow

The overall process I followed was:

```text
AWS CLI
   |
   v
Create EKS Cluster with eksctl
   |
   v
Configure kubeconfig
   |
   v
Clone Git Repository
   |
   v
Deploy Backend Microservices
   |
   v
Install Docker + Helm
   |
   v
Install NGINX Ingress Controller
   |
   v
Get AWS Load Balancer Address
   |
   v
Update ingressrule.yml
   |
   v
Configure React application
   |
   v
Build Docker Image
   |
   v
Push Image to Docker Hub
   |
   v
Deploy reactfe.yml
   |
   v
Verify Pods / Services / Ingress
   |
   v
Access Application using Load Balancer endpoint
```

## Commands I Used for Verification

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingress -n ingress-nginx
```

These commands helped me verify that the workloads, services, ingress controller, and ingress rules were created successfully.

## Troubleshooting / Important Notes

### Helm command not found

If `helm` is not available, install it before running the NGINX Ingress installation commands.

```bash
helm version
```

### Check the Ingress Controller

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Check Ingress Configuration

```bash
kubectl get ingress -n ingress-nginx
```

### Check all Pods

```bash
kubectl get pods -A
```

### Docker Image

The React image used in my deployment was:

```text
vedantsatpute07/reactfe:v1
```

If the image name or tag changes, update `reactfe.yml` accordingly before applying it.

## My Final Deployment Stack

| Layer | Technology |
|---|---|
| Cloud | AWS |
| Kubernetes Platform | Amazon EKS |
| Cluster Provisioning | eksctl |
| Kubernetes CLI | kubectl |
| Package Manager | Helm |
| Ingress | NGINX Ingress Controller |
| Containerization | Docker |
| Image Registry | Docker Hub |
| Frontend | React |
| Backend | Node.js microservices |
| Routing | Kubernetes Ingress |

## Result

The application was deployed on **AWS EKS** with multiple backend microservices and a React frontend. The **NGINX Ingress Controller** provides a single entry point through the AWS Load Balancer and routes incoming requests to the appropriate Kubernetes service.

This project gave me practical experience with:

- Creating an EKS cluster using `eksctl`
- Connecting `kubectl` to AWS EKS
- Deploying Kubernetes manifests
- Installing and configuring Helm
- Installing NGINX Ingress Controller
- Working with AWS Load Balancers through Kubernetes services
- Building Docker images
- Pushing images to Docker Hub
- Deploying a custom container image to Kubernetes
- Verifying Kubernetes Pods, Services, and Ingress resources
