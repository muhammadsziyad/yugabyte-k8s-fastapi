
### Deploying YugabyteDB and FastAPI Application

## Overview

This project involves deploying YugabyteDB and a Python FastAPI application on Kubernetes. YugabyteDB will be deployed using StatefulSet for high availability and persistence, while the FastAPI application will be deployed using Deployment for scalability and load balancing. Kubernetes Services will expose the deployments, and Ingress will handle external access. Configuration and credentials will be managed using ConfigMaps and Secrets.

## Objectives

1.  **Deploy YugabyteDB** using StatefulSet to ensure high availability and data persistence.
2.  **Deploy a FastAPI application** with Deployment to ensure scaling and load distribution.
3.  **Expose services** using Kubernetes Services and Ingress for external access.
4.  **Manage configuration and credentials** using ConfigMaps and Secrets.

## 1. Installation

**Prerequisites:**

-   Kubernetes cluster (e.g., Minikube or any cloud provider)
-   `kubectl` CLI tool installed
-   Docker installed

### 1.1 Install Minikube (Optional, if you don’t have a Kubernetes cluster)

Follow the official Minikube installation guide to install Minikube on your system.

### 1.2 Start Minikube

```bash
minikube start
``` 

### 1.3 Install Helm (for managing Kubernetes applications, optional but recommended)

Follow the official Helm installation guide to install Helm.

## 2. Kubernetes Configuration

We will create Kubernetes YAML files for deploying YugabyteDB and a FastAPI application, along with Services, Ingress, ConfigMaps, and Secrets.

### 2.1 YugabyteDB StatefulSet and Service

**File: `yugabyte-db-statefulset.yaml`**



```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: yugabyte-db
spec:
  serviceName: "yugabyte-db"
  replicas: 2
  selector:
    matchLabels:
      app: yugabyte-db
  template:
    metadata:
      labels:
        app: yugabyte-db
    spec:
      containers:
      - name: yugabyte
        image: yugabytedb/yugabyte:latest
        ports:
        - containerPort: 5433
        volumeMounts:
        - name: data
          mountPath: /mnt/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
``` 

**File: `yugabyte-db-service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: yugabyte-db
spec:
  ports:
  - port: 5433
    targetPort: 5433
  clusterIP: None
  selector:
    app: yugabyte-db
``` 

### 2.2 FastAPI Deployment and Service

**File: `fastapi-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fastapi-app
  template:
    metadata:
      labels:
        app: fastapi-app
    spec:
      containers:
      - name: fastapi
        image: your-fastapi-app-image:latest
        ports:
        - containerPort: 8000
        env:
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db-url
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: password
``` 

**File: `fastapi-service.yaml`**


```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: fastapi-app
``` 

### 2.3 Ingress Configuration

**File: `fastapi-ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fastapi-ingress
spec:
  rules:
  - host: fastapi-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fastapi-service
            port:
              number: 80
``` 

### 2.4 ConfigMap and Secret

**File: `app-config.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  db-url: yugabyte-db.yugabyte-db.svc.cluster.local
``` 

**File: `db-secrets.yaml`**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
data:
  username: <base64-encoded-username>
  password: <base64-encoded-password>
``` 

_Note: Replace `<base64-encoded-username>` and `<base64-encoded-password>` with base64 encoded values._

## 3. Apply the Configuration

1.  **Deploy YugabyteDB:**

    
    ```bash
    kubectl apply -f yugabyte-db-statefulset.yaml
    kubectl apply -f yugabyte-db-service.yaml
    ``` 
    
2.  **Deploy FastAPI Application:**
    
    ```bash
    kubectl apply -f fastapi-deployment.yaml
    kubectl apply -f fastapi-service.yaml
    kubectl apply -f fastapi-ingress.yaml
    ``` 
    
3.  **Create ConfigMap and Secret:**
    
    ```bash
    kubectl apply -f app-config.yaml
    kubectl apply -f db-secrets.yaml
    ``` 
    
4.  **Add Hostname to `/etc/hosts` (for local testing):**
    
    ```bash
    echo "$(minikube ip) fastapi-app.local" | sudo tee -a /etc/hosts
    ``` 
    

## 4. Verify and Access the Application

1.  **Verify Pods and Services:**
    
    ```bash
    kubectl get pods
    kubectl get services
    kubectl get ingress
    ``` 
    
2.  **Access the FastAPI Application:**
    
    Open your browser and navigate to `http://fastapi-app.local`. You should see your FastAPI application running and connected to YugabyteDB.