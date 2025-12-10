# CST-8915-Final-Project

## link to youtube

https://youtu.be/lY_Ns3OH52w

## links to repos

https://github.com/ObaidaKandakji/store-front-L8

https://github.com/ObaidaKandakji/store-admin-L8

https://github.com/ObaidaKandakji/order-service-L8

https://github.com/ObaidaKandakji/product-service-L8

https://github.com/ObaidaKandakji/makeline-service-L8

---

### Architecture Walkthrough

The system is a microservices-based e-commerce platform with separate customer-facing and employee-facing UIs, backed by asynchronous processing and a shared database.

- **Store-Front (Vue.js, customer-facing)**  
  - Public web app where customers browse products and place orders.  
  - Calls **product-service** to fetch product data.  
  - Calls **order-service** to submit new orders.

- **Order-Service (Node.js)**  
  - Receives orders from store-front.  
  - Publishes each order to **RabbitMQ** (order queue) for asynchronous processing by makeline-service.

- **Product-Service (Rust)**  
  - Exposes CRUD APIs for products (create, update, delete, list).  
  - Serves product data to **store-front** and **store-admin**.

- **Store-Admin (Vue.js, employee-facing)**  
  - Internal UI for staff to manage the store.  
  - Uses **product-service** for product management.  
  - Uses **makeline-service** and **ai-service** for viewing/handling orders and AI-powered helper features.

- **Makeline-Service (Go)**  
  - Consumes orders from **RabbitMQ**.  
  - Processes and marks orders as completed.  
  - Persists order history in **MongoDB**.

- **Infrastructure**  
  - **MongoDB (StatefulSet)** stores order data.  
  - **RabbitMQ (StatefulSet)** handles the order queue.  
  - All services run as Kubernetes workloads on **AKS**, exposed via internal ClusterIP services and external LoadBalancers for the UIs.


## Deployment Instructions

### 1. Prerequisites

- Azure subscription (e.g., Azure for Students)
- Docker Hub account
- GitHub account
- Tools installed locally:
  - [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/)
  - [Docker Desktop](https://www.docker.com/products/docker-desktop)
  - `git`

The project uses:
- Azure Kubernetes Service (AKS)
- Docker Hub container registry
- GitHub Actions for CI/CD
- A single Kubernetes manifest (`aps-all-in-one.yaml`) that deploys:
  - MongoDB (StatefulSet)
  - RabbitMQ (StatefulSet)
  - order-service
  - makeline-service
  - product-service
  - store-front
  - store-admin

---

### 2. Clone the Repositories

Clone the **infrastructure** repo (contains `aps-all-in-one.yaml`):

```bash
git clone https://github.com/<your-github-username>/CST-8915-Final-Project.git
cd CST-8915-Final-Project
```

Clone the microservice repos (each has its own CI/CD workflow):

### 3. Build and Push Docker Images (Initial Setup)

From each service folder, build and push an image to Docker Hub.

Example for store-front:

```bash
docker build -t <dockerhub-username>/store-front:latest .

docker push <dockerhub-username>/store-front:latest
```

Repeat for the other services, changing the image name appropriately

Update aps-all-in-one.yaml so each image: line points to the correct image

```bash
    image: docker.io/<dockerhub-username>/store-front:latest
```

### 4. Create Resource Group and AKS Cluster

Log in and create a resource group:

```bash
    az login
az account set --subscription "<Your Subscription Name>"

az group create \
  --name cst8915-final-rg \
  --location canadacentral

```
Create an AKS cluster with 2 nodes

### 5. Connect kubectl to the AKS Cluster

Get credentials and verify

```bash
az aks get-credentials \
--resource-group cst8915-final-rg \
--name cst8915-final-aks \
--overwrite-existing
kubectl get nodes
```

### 6. Deploy All Services Using aps-all-in-one.yaml

From the infrastructure repo folder (CST-8915-Final-Project):

```bash 
kubectl apply -f ./aps-all-in-one.yaml
```

### 7. Configure GitHub Actions (CI/CD)

For each service repo (store-front, store-admin, product-service, order-service, makeline-service, ai-service):

7.1. Add Repository Secrets

In GitHub:

Settings → Secrets and variables → Actions → New repository secret

Add:

DOCKER_USERNAME = dockerhub-username

DOCKER_PASSWORD = Docker Hub access token

KUBE_CONFIG_DATA = base64-encoded kubeconfig

### 7.2. Add Repository Variables

Settings → Secrets and variables → Actions → Variables

Example

```bash
store-front repo:

DOCKER_IMAGE_NAME = store-front

DEPLOYMENT_NAME = store-front

CONTAINER_NAME = store-front
```

Do this for all repos

Make sure each repo has a workflow file at:
```bash
.github/workflows/ci_cd.yaml
```

Get external IP from azure AKS Services and ingresses