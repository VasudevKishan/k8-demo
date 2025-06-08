# Kubernetes Demo Setup with Minikube

This project demonstrates a simple Kubernetes setup using Minikube, deploying a MongoDB database and a Node.js web application that connects to it. The setup uses Kubernetes Secrets, ConfigMaps, Deployments, and Services.

---

## Table of Contents

- [Kubernetes Concepts](#kubernetes-concepts)
- [What is Minikube?](#what-is-minikube)
- [What is kubectl?](#what-is-kubectl)
- [YAML File Explanations](#yaml-file-explanations)
  - [mongo-secret.yaml](#1-mongo-secretyaml)
  - [mongo-config.yaml](#2-mongo-configyaml)
  - [mongo.yaml](#3-mongoyaml)
  - [web-app.yaml](#4-web-appyaml)
- [Step-by-Step Setup](#step-by-step-setup)
- [Accessing the Application](#accessing-the-application)
- [Notes](#notes)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Kubernetes Concepts

- **Pod**: The smallest deployable unit in Kubernetes, representing a single instance of a running process.
- **Deployment**: Manages a set of identical pods, ensuring the desired number are running and updating them as needed.
- **Service**: Exposes a set of pods as a network service. Types include ClusterIP (internal), NodePort (external via node IP and port), and LoadBalancer.
- **Secret**: Stores sensitive data (like passwords) in base64-encoded form.
- **ConfigMap**: Stores non-sensitive configuration data as key-value pairs.
- **ReplicaSet**: Ensures a specified number of pod replicas are running at any given time (managed by Deployments).
- **Label/Selector**: Used to organize and select groups of objects.

---

## What is Minikube?

- **Minikube** is a tool that runs a single-node Kubernetes cluster locally on your machine. It is ideal for learning, development, and testing Kubernetes applications without needing a full cloud-based cluster.

## What is kubectl?

- **kubectl** is the command-line tool for interacting with Kubernetes clusters. It allows you to deploy applications, inspect and manage cluster resources, and view logs and events.

---

## YAML File Explanations

### 1. mongo-secret.yaml

Stores MongoDB credentials as Kubernetes secrets.

```yaml
apiVersion: v1 # API version for Kubernetes objects
kind: Secret # Declares this as a Secret
metadata:
  name: mongo-secret # Name of the secret
type: Opaque # Arbitrary user-defined data
data: # Key-value pairs (base64 encoded)
  mongo-user: bW9uZ291c2Vy # "mongouser" (base64)
  mongo-password: bW9uZ29wYXNzd29yZA== # "mongopassword" (base64)
```

### 2. mongo-config.yaml

Stores the MongoDB connection URL as a ConfigMap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config
data:
  mongo-url: mongo-service # The hostname for MongoDB service (used by the app)
```

### 3. mongo.yaml

Defines the MongoDB Deployment and Service.

#### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-deployment
  labels:
    app: mongo
spec:
  replicas: 1 # Number of MongoDB pods
  selector:
    matchLabels:
      app: mongo # Selects pods with label app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:5.0 # MongoDB Docker image
          ports:
            - containerPort: 27017 # MongoDB default port
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-user # Injects username from secret
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-password # Injects password from secret
```

#### Service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector:
    app: mongo
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

- Exposes MongoDB internally in the cluster as `mongo-service:27017`.

### 4. web-app.yaml

Defines the web application Deployment and Service.

#### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: nanajanashia/k8s-demo-app:v1.0
          ports:
            - containerPort: 3000
          env:
            - name: USER_NAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-user
            - name: USER_PWD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-password
            - name: DB_URL
              valueFrom:
                configMapKeyRef:
                  name: mongo-config
                  key: mongo-url
```

- The app gets MongoDB credentials from the secret and the DB URL from the config map.

#### Service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30100
```

- Exposes the web app on port 30100 of the Minikube node.

---

## Step-by-Step Setup

### 1. Start Minikube

```sh
minikube start --driver=docker
```

- Starts a local Kubernetes cluster using Docker.

### 2. Apply Kubernetes Resources

```sh
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-config.yaml
kubectl apply -f mongo.yaml
kubectl apply -f web-app.yaml
```

- Creates secrets, configmaps, deployments, and services.

### 3. Check Resource Status

```sh
kubectl get pods
kubectl get svc
```

- Ensure all pods are `Running` and services are created.

---

## Accessing the Application

### With Docker Driver

**NodePort is NOT directly accessible from your host. Use one of these:**

#### Option 1: `minikube service`

```sh
minikube service webapp-service
```

- Opens the app in your browser via a temporary tunnel.

#### Option 2: `kubectl port-forward`

```sh
kubectl port-forward service/webapp-service 30100:3000
```

- Access the app at [http://localhost:30100](http://localhost:30100).

---

## Notes

- **Secrets** are base64-encoded, not encrypted. Do not commit real secrets to version control.
- **ConfigMaps** are for non-sensitive configuration.
- **Deployments** manage pod lifecycle and scaling.
- **Services** expose pods to the cluster or externally.
- **Minikube Docker driver** does not expose NodePorts to your host by default. Always use `minikube service` or `kubectl port-forward`.

---

## Troubleshooting

- If pods are not running, check logs:
  ```sh
  kubectl logs <pod-name>
  ```
- If you can't access the app, ensure you are using the correct method for your Minikube driver.

---

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Minikube Handbook: Accessing Services](https://minikube.sigs.k8s.io/docs/handbook/accessing/)
