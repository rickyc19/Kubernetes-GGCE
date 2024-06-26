# GGCE Kubernetes Deployment Guide

This guide provides step-by-step instructions to deploy the GGCE application in a Google Cloud Platform (GCP) Autopilot Kubernetes cluster, including the setup of a Cloud SQL Server instance and SSL configuration with Traefik.

## Prerequisites

1. **Google Cloud SDK** installed and authenticated.
2. **kubectl** CLI tool installed.
3. **openssl** installed for generating SSL certificates.

## Steps

### 1. Create Kubernetes Cluster

Create an Autopilot cluster in GCP:

```sh
gcloud container clusters create-auto ggce-cluster --region us-central1
```

### 2. Set up Cloud SQL Server

1. **Create a Cloud SQL Server instance:**
   - Go to the Cloud SQL section in the Google Cloud Console.
   - Click "Create Instance" and choose "SQL Server".
   - Set instance ID to `ggce-sql-instance`.
   - Choose region `us-central1`.
   - Set a strong password for the `sqlserver` user.

2. **Create the database and user:**
   - Connect to the SQL instance using a client (e.g., SQL Server Management Studio).
   - Create the `ggce` database and a new user `ggce_user` with a strong password.

```sql
   CREATE DATABASE ggce;
   CREATE LOGIN ggce_user WITH PASSWORD = 'YourStrong@Passw0rd';
   USE ggce;
   CREATE USER ggce_user FOR LOGIN ggce_user;
   ALTER ROLE db_owner ADD MEMBER ggce_user;
  ```

### 3. Create Kubernetes Secrets

1. **Create secret for SQL Server credentials:**

```sh
kubectl create secret generic sql-db-credentials \
  --from-literal=DB_USER=ggce_user \
  --from-literal=DB_PASSWORD=YourStrong@Passw0rd \
  -n ggce
```

2. **Create secret for Cloud SQL proxy credentials:**

```sh
kubectl create secret generic cloudsql-instance-credentials \
  --from-file=key.json=/path/to/your/cloud-sql-proxy-sa-key.json \
  -n ggce
```

3. **Create secret for SSL certificates:**

Generate the SSL certificate and key:

```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=<your-ip-address>.nip.io/O=ggce"
```

Create the Kubernetes secret:

```sh
kubectl create secret tls ggce-tls --key tls.key --cert tls.crt -n ggce
```

### 4. Apply Kubernetes Configuration Files

1. **Create namespace:**

```sh
kubectl create namespace ggce
```

2. **Apply Deployment and Service files:**

```sh
kubectl apply -f reverse-proxy-deployment.yaml
kubectl apply -f reverse-proxy-service.yaml
```

After deploying the services and creating the LoadBalancer, you need to update the configuration files with the external IP of the reverse-proxy service.

3. **Get the external IP of the reverse-proxy service:**

```sh
kubectl get svc reverse-proxy -n ggce
```

4. **Update the following files with the external IP:**

```markdown
ggce-api-deployment.yaml
ggce-ui-deployment.yaml
ggce-ingress.yaml
```

5. **Apply Deployment and Service files:**

```sh
kubectl apply -f ggce-api-deployment.yaml
kubectl apply -f ggce-api-service.yaml
kubectl apply -f ggce-ui-deployment.yaml
kubectl apply -f ggce-ui-service.yaml
```

6. **Apply Traefik configuration:**

```sh
kubectl apply -f traefik-config.yaml
kubectl apply -f traefik-cluster-role.yaml
kubectl apply -f traefik-service-account.yaml
kubectl apply -f traefik-binding.yaml
```

### 5. Verify Deployment

1. **Check the status of the pods:**

```sh
kubectl get pods -n ggce
```

2. **Check the services and endpoints:**

```sh
kubectl get svc -n ggce
kubectl get ingress -n ggce
```

3. **Verify that the application is running:**
   - Access `https://<your-ip-address>.nip.io/api` for the API.
   - Access `https://<your-ip-address>.nip.io/` for the UI.
