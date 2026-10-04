# Cloud Services - Weekly Assignment 5

Three-tier web application extended with a public weather API and deployed on CSC Rahti using OpenShift/Kubernetes.

The application was created for the Week 5 assignment of the Cloud Services course at Oulu University of Applied Sciences. It extends the Week 4 application with an Open-Meteo public API integration.

## Application

The application consists of three containers:

- **Frontend:** Nginx
- **Backend:** Flask + Gunicorn
- **Database:** MySQL 8.4

The frontend communicates with the backend through a Kubernetes Service. The backend communicates with MySQL through another internal Service.

The backend reads the current MySQL server time and updates a persistent visit counter every time the frontend requests data.

Only the frontend is exposed to the Internet through an OpenShift Route. The backend and database remain internal to the cluster.

## Week 5 - Public API Integration

For Week 5, the Week 4 application was extended with an external public API.

The Flask backend now provides a `/api/weather` endpoint. This endpoint requests current weather information for Oulu, Finland, from the Open-Meteo public API and returns the relevant data to the frontend as JSON.

The displayed weather information includes temperature, apparent temperature, wind speed, weather condition and observation time.

The new data flow is:

Frontend -> Flask Backend -> Open-Meteo Public API

The original database functionality is still available:

Frontend -> Flask Backend -> MySQL

This allows the application to use both persistent data from MySQL and live information from an external REST API. Open-Meteo does not require an API key for this use case.

## Architecture

```text
Internet
   |
   v
OpenShift Route
   |
   v
Frontend Service
   |
   v
Frontend Pod (Nginx)
   |
   v
Backend Service
   |
   v
Backend Pod(s) (Flask + Gunicorn)
   |
   v
MySQL Service
   |
   v
MySQL Pod
   |
   v
PersistentVolumeClaim
```

## Repository Structure

```text
cloud-services-week5/
├── backend/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── db/
│   └── init/
│       └── init.sql
├── frontend/
│   ├── Dockerfile
│   ├── index.html
│   └── nginx.conf
├── k8s/
│   ├── 01-secret.example.yaml
│   ├── 02-configmap.yaml
│   ├── 03-mysql-pvc.yaml
│   ├── 04-mysql.yaml
│   ├── 05-mysql-init.yaml
│   ├── 06-backend.yaml
│   └── 07-frontend.yaml
├── docker-compose.yml
└── .gitignore
```

## Configuration and Secrets

Application configuration is separated from the container images.

A **ConfigMap** is used for non-sensitive configuration such as:

- database host
- database name
- database user

A **Secret** is used for sensitive values such as the MySQL passwords.

The real Secret file is excluded from Git with `.gitignore`. Only `01-secret.example.yaml`, containing placeholder values, is included in this repository.

Before deploying the application, create the real Secret file:

```bash
cp k8s/01-secret.example.yaml k8s/01-secret.yaml
```

Then replace the placeholder values with your own passwords.

Do not commit `k8s/01-secret.yaml` to Git.

## Container Images

The application images are available on Docker Hub:

```text
iagoprolg/cloud-services-week5-frontend
iagoprolg/cloud-services-week5-backend
```

The images used by Rahti were built for `linux/amd64`.

Example:

```bash
docker buildx build --platform linux/amd64 \
  -f frontend/Dockerfile \
  -t iagoprolg/cloud-services-week5-frontend:1.0.1 \
  --push .
```

## Deploying to Rahti

Log in to Rahti with the OpenShift CLI and select or create a project.

Apply the resources in this order:

```bash
oc apply -f k8s/01-secret.yaml
oc apply -f k8s/02-configmap.yaml
oc apply -f k8s/03-mysql-pvc.yaml
oc apply -f k8s/05-mysql-init.yaml
oc apply -f k8s/04-mysql.yaml
oc apply -f k8s/06-backend.yaml
oc apply -f k8s/07-frontend.yaml
```

Check the resources:

```bash
oc get pods
oc get services
oc get routes
oc get pvc
```

## Persistent Storage

MySQL uses a PersistentVolumeClaim called `mysql-data`.

The database data therefore exists independently of the MySQL Pod. This was tested by deleting the MySQL Pod and allowing the Deployment to create a replacement. The visit counter remained stored after the new Pod started.

In the previous Docker Compose deployment, persistence was provided by a Docker named volume. In Rahti, persistence is provided through a Kubernetes PersistentVolumeClaim, which is independent of an individual Pod.

## Self-Healing and Scaling

The backend runs as a Kubernetes Deployment.

Deleting the backend Pod caused the Deployment and ReplicaSet to automatically create a replacement Pod to restore the desired state.

The backend was also scaled from one replica to two replicas:

```bash
oc scale deployment/backend --replicas=2
```

The frontend continued using the `backend` Service instead of communicating with individual Pod IP addresses.

Three replicas were initially tested, but the third Pod could not be created because the CSC project CPU quota had been reached. Two replicas were sufficient to verify the scaling behaviour.

After the experiment, the backend was returned to one replica.

## Rolling Update

The frontend image was updated from version `1.0.0` to `1.0.1`.

The Deployment was updated with:

```bash
oc set image deployment/frontend \
  frontend=iagoprolg/cloud-services-week5-frontend:1.0.1
```

During the rollout, a new Pod was created before the old Pod was removed.

The rollout status was checked with:

```bash
oc rollout status deployment/frontend
```

## Problems Encountered

### Container architecture

The first images were built on an Apple Silicon Mac and were therefore ARM64 images. Rahti worker nodes required AMD64 images, which caused an `ImagePullBackOff`.

The images were rebuilt and pushed using Docker Buildx with:

```bash
--platform linux/amd64
```

After this change, the containers started correctly on Rahti.

### Nginx permissions

The frontend initially had a permissions problem related to the Nginx PID file. This was solved by using the unprivileged Nginx image and configuring Nginx to run correctly on port 8080.

### CPU quota while scaling

Scaling the backend to three replicas caused a `FailedCreate` error because the CSC project CPU quota was reached. The Deployment was scaled to two replicas instead, which successfully demonstrated multiple backend Pods behind one Service.

## Docker Compose vs Rahti

Docker Compose runs the application as a group of containers on a single Docker host. Networking is simple because containers can communicate using Compose service names, and persistence can be provided with Docker volumes. However, the Docker host and the lifecycle of the containers have to be managed directly.

Rahti uses Kubernetes/OpenShift resources such as Deployments, Services, Routes, ConfigMaps, Secrets and PersistentVolumeClaims. The application configuration is more detailed, but Rahti provides orchestration features that Docker Compose does not provide by itself, including automatic Pod replacement, horizontal scaling and rolling updates.

Networking is also handled differently. In Docker Compose, ports can be mapped directly from the host to containers. In Rahti, internal components communicate through Services, while the frontend is exposed externally using an OpenShift Route.

For database persistence, Docker Compose used a named Docker volume. In Rahti, the MySQL Pod uses a PersistentVolumeClaim, allowing the database data to survive Pod replacement.

## Running Application

https://frontend-cloud-services-week4-iago.2.rahtiapp.fi

## Author

Iago Prol González
