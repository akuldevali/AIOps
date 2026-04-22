# Full-Stack Application CI/CD Pipeline

## Project Overview

This project demonstrates a complete DevOps CI/CD pipeline for a full-stack web application. It includes automated testing, containerization, Kubernetes deployment, and monitoring.

## Architecture

```
┌─────────────┐
│   GitHub    │
│  (Source)   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ GitHub      │
│ Actions     │ ◄── CI/CD Pipeline
│ (CI/CD)     │
└──────┬──────┘
       │
       ├──► Build Docker Images
       ├──► Run Tests
       ├──► Security Scanning
       │
       ▼
┌─────────────┐
│ Docker Hub  │
│ (Registry)  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Kubernetes  │
│  Cluster    │ ◄── Terraform (IaC)
└──────┬──────┘
       │
       ├──► Frontend Pods
       ├──► Backend Pods
       ├──► Database
       │
       ▼
┌─────────────┐
│ Prometheus  │
│  Grafana    │ ◄── Monitoring
└─────────────┘
```

## Prerequisites

- Docker Desktop or Docker Engine
- Kubernetes cluster (minikube, kind, or cloud provider)
- Terraform >= 1.0
- kubectl
- GitHub account

## Project Structure

```
fullstack-cicd-pipeline/
├── frontend/              # React frontend application
├── backend/               # Node.js backend API
├── infrastructure/        # Terraform IaC
│   ├── kubernetes/
│   └── terraform.tf
├── .github/
│   └── workflows/
│       └── ci-cd.yml      # GitHub Actions workflow
├── docker-compose.yml     # Local development
├── k8s/                   # Kubernetes manifests
│   ├── frontend-deployment.yaml
│   ├── backend-deployment.yaml
│   └── ingress.yaml
└── monitoring/            # Prometheus & Grafana configs
```

## Getting Started

### 1. Clone the Repository

```bash
git clone <repository-url>
cd DevOps-Practice-Guide/projects/fullstack-cicd-pipeline
```

### 2. Local Development Setup

```bash
# Start services with Docker Compose
docker-compose up -d

# Access the application
# Frontend: http://localhost:3000
# Backend API: http://localhost:5000
```

### 3. CI/CD Pipeline Setup

1. Push code to GitHub
2. GitHub Actions will automatically:
   - Run tests
   - Build Docker images
   - Push to Docker Hub
   - Deploy to Kubernetes

### 4. Kubernetes Deployment

```bash
# Apply Kubernetes manifests
kubectl apply -f k8s/

# Check deployment status
kubectl get pods
kubectl get services
```

### 5. Infrastructure Provisioning

```bash
cd infrastructure
terraform init
terraform plan
terraform apply
```

## Features

- ✅ Automated testing (unit, integration)
- ✅ Docker containerization
- ✅ Multi-stage Docker builds
- ✅ CI/CD with GitHub Actions
- ✅ Kubernetes deployment
- ✅ Infrastructure as Code (Terraform)
- ✅ Monitoring with Prometheus & Grafana
- ✅ Logging aggregation
- ✅ Security scanning

## CI/CD Pipeline Stages

1. **Lint**: Code quality checks
2. **Test**: Run unit and integration tests
3. **Build**: Create Docker images
4. **Scan**: Security vulnerability scanning
5. **Push**: Push images to registry
6. **Deploy**: Deploy to Kubernetes
7. **Monitor**: Health checks and monitoring

## Monitoring

Access Grafana dashboard:
```bash
kubectl port-forward svc/grafana 3000:3000
# Open http://localhost:3000
```

## Cleanup

```bash
# Remove Kubernetes resources
kubectl delete -f k8s/

# Remove Docker containers
docker-compose down

# Destroy infrastructure
cd infrastructure
terraform destroy
```

## Next Steps

- Add more comprehensive tests
- Implement blue-green deployment
- Add service mesh (Istio/Linkerd)
- Implement GitOps with ArgoCD
- Add distributed tracing

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Terraform Documentation](https://www.terraform.io/docs/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
