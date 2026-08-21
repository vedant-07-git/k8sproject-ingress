# K8s Project – Ingress

A simple project where a Node.js app is split into microservices (Frontend, Products, Orders) and a React app, all deployed on Kubernetes (AWS EKS) and connected together using a single Ingress.

## What this project does

- Runs multiple small services (frontend, products, orders) instead of one big app
- All services are deployed on Kubernetes
- One Ingress controller handles all incoming traffic and routes it to the correct service
- Auto-scaling is set up so pods increase/decrease based on load

## Tech Used

- Node.js (backend microservices)
- React (frontend)
- Docker (to build images)
- Kubernetes / EKS (to run everything)
- NGINX Ingress Controller (to route traffic)

## Project Files

```
ingress/
├── frontend.yml     -> Frontend microservice
├── orders.yml       -> Orders microservice
├── products.yml     -> Products microservice
├── reactfe.yml       -> React app
└── ingressrule.yml   -> Routing rules (Ingress)
```

## How I Deployed It

1. Created the EKS cluster
2. Cloned this repo
   ```bash
   git clone https://github.com/dineshgirde/k8sproject-ingress.git
   cd k8sproject-ingress/monolith-to-microservices-master/ingress
   ```
3. Deployed the backend services
   ```bash
   kubectl apply -f frontend.yml
   kubectl apply -f orders.yml
   kubectl apply -f products.yml
   ```
4. Installed Ingress Controller using Helm
   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo update
   helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
   ```
5. Got the Load Balancer address
   ```bash
   kubectl get svc -n ingress-nginx
   ```
6. Put that address inside `ingressrule.yml` (in the `host:` field), then applied it
   ```bash
   kubectl apply -f ingressrule.yml
   ```
7. Installed Docker, added the Load Balancer URL to React app's `.env` file
8. Did `git pull` to get latest code
9. Updated `reactfe.yml` with my own Docker image name
10. Built and pushed the Docker image, then deployed it
    ```bash
    docker build -t <your-dockerhub-username>/reactfe:v1 .
    docker push <your-dockerhub-username>/reactfe:v1
    kubectl apply -f reactfe.yml
    ```

## Note

All resources (frontend, orders, products, react app, ingress controller, ingress rule) are in the **same namespace: `ingress-nginx`**.

## How to Check if it's Working

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingress -n ingress-nginx
```

Then open the Load Balancer URL in your browser to see the app running.
