# NBA Stats Kubernetes Dashboard

A comprehensive NBA statistics dashboard demonstrating cloud-native principles and Kubernetes capabilities. This application provides real-time NBA player statistics, predictive analytics, and advanced metrics, all wrapped in an interactive dashboard that automatically scales based on demand.

## Features
- Real-time NBA player statistics and performance metrics
- Predictive analytics using Linear Regression for player performance forecasting
- Advanced statistical analysis (True Shooting %, Usage Rate, Efficiency Rating)
- Interactive career progression visualizations
- Automatic scaling based on CPU utilization (HPA)
- Kubernetes-native configuration management
- Interactive and responsive web interface built with Streamlit

## Architecture
The application is built using:
- Frontend: Streamlit (Python web framework)
- Data Processing: Pandas, NumPy, Scikit-learn
- APIs: NBA API for real-time data
- Container: Docker
- Orchestration: Kubernetes
- Ingress: NGINX Ingress Controller
- Metrics: Resource metrics API

## Prerequisites

Before you begin, ensure you have the following tools installed:

1. **Docker**
   - [Docker Desktop for Windows/Mac](https://www.docker.com/products/docker-desktop/)
   - [Docker Engine for Linux](https://docs.docker.com/engine/install/)
   - Verify: `docker --version`

2. **kubectl**
   - [Official Installation Guide](https://kubernetes.io/docs/tasks/tools/)
   - Windows: `choco install kubernetes-cli`
   - Mac: `brew install kubectl`
   - Linux: 
     ```bash
     curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
     sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
     ```
   - Verify: `kubectl version --client`

3. **Minikube**
   - [Official Installation Guide](https://minikube.sigs.k8s.io/docs/start/)
   - Windows: `choco install minikube`
   - Mac: `brew install minikube`
   - Linux:
     ```bash
     curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
     sudo install minikube-linux-amd64 /usr/local/bin/minikube
     ```
   - Verify: `minikube version`

## Understanding the Components

### Kubernetes Concepts Used
1. **Deployment**
   - Manages the lifecycle of our application pods
   - Ensures desired number of replicas are running
   - Handles rolling updates and rollbacks
   - In our case: Manages the NBA stats application containers

2. **ConfigMap**
   - Stores configuration data as key-value pairs
   - Separates configuration from application code
   - Can be updated without rebuilding containers
   - Our usage: Configures app title, features, and display options

3. **Service**
   - Provides stable networking for pods
   - Load balances traffic between pod replicas
   - Enables pod discovery within cluster
   - Our implementation: Exposes the Streamlit application

4. **HorizontalPodAutoscaler (HPA)**
   - Automatically scales pods based on metrics
   - Monitors CPU utilization (in our case)
   - Scales between 2-5 pods as needed
   - Configured with custom scaling behavior

5. **Ingress**
   - Manages external access to services
   - Provides HTTP/HTTPS routing
   - Enables hostname-based routing
   - Our setup: Routes traffic to nba-stats.info

### Key Features Explained

1. **Automatic Scaling**
   ```plaintext
   High Load → CPU Usage ↑ → HPA Detects → Pods Scale Up
   Low Load → CPU Usage ↓ → HPA Detects → Pods Scale Down
   ```

2. **Configuration Management**
   ```plaintext
   ConfigMap → Environment Variables → Application
   Updates to ConfigMap → Rolling Update → New Configuration
   ```

3. **Network Flow**
   ```plaintext
   User Request → Ingress → Service → Pods
   Load Balancing → Multiple Pods → Even Distribution
   ```

## Educational Resources

### Kubernetes Learning Path
1. **Basics**
   - [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
   - [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

2. **Core Concepts**
   - [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
   - [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
   - [Services](https://kubernetes.io/docs/concepts/services-networking/service/)

3. **Advanced Topics**
   - [HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
   - [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress/)
   - [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)


## Setup Guide

### 1. Start Minikube
```bash
# Start Minikube cluster
minikube start

# Enable the Ingress addon
minikube addons enable ingress

# Verify Ingress controller is running
kubectl get pods -n ingress-nginx -w
# Wait until the ingress-nginx-controller pod is Running
```
### Enable Metrics Server
The HPA requires the Metrics Server to be running in your cluster for pod autoscaling to work:

```bash
# Enable metrics-server in Minikube
minikube addons enable metrics-server

# Verify metrics-server is running
kubectl get pods -n kube-system | grep metrics-server

# Verify metrics are being collected
kubectl top pods -n kube-stats
```

### 2. Create Namespace
```bash
# Create namespace for our application
kubectl create namespace nba-stats

# Optional: Set as default namespace
kubectl config set-context --current --namespace=nba-stats
```

### 3. Application Manifests

Create the following YAML files for deployment:

1. **ConfigMap** (`nba-configmap.yaml`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nba-stats-config
  namespace: nba-stats
data:
  APP_TITLE: "NBA Stats Kubernetes Edition :)"
  RECENT_GAMES_COUNT: "100"
  SHOW_PREDICTIONS: "true"
  SHOW_ADVANCED_STATS: "true"
```

2. **Deployment** (`nba-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nba-stats-app
  namespace: nba-stats
  labels:
    app: nba-stats-app 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nba-stats-app
  template:
    metadata:
      labels:
        app: nba-stats-app
    spec:
      containers:
      - name: nba-stats-app
        image: amirmalaeb/nba-stats-app:3
        ports:
        - containerPort: 8501
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        env:
        - name: HOST 
          value: "0.0.0.0" 
        envFrom:
        - configMapRef: 
            name: nba-stats-config
```

3. **Service** (`nba-service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nba-stats-service
  namespace: nba-stats
spec:
  type: NodePort
  selector:
    app: nba-stats-app
  ports:
  - port: 8501
    targetPort: 8501
```

4. **HorizontalPodAutoscaler** (`nba-hpa.yaml`):
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nba-stats-hpa
  namespace: nba-stats
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nba-stats-app
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60    
      policies:
      - type: Percent
        value: 50                       
        periodSeconds: 15 
    scaleUp:
      stabilizationWindowSeconds: 30    
      policies:
      - type: Percent
        value: 50                      
        periodSeconds: 15
```

5. **Ingress** (`nba-ingress.yaml`):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nba-stats-ingress
  namespace: nba-stats
spec:
  ingressClassName: nginx  
  rules:
  - host: nba-stats.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nba-stats-service
            port:
              number: 8501
```

### 4. Deploy the Application

Apply the manifests in order:
```bash
kubectl apply -f nba-configmap.yaml
kubectl apply -f nba-deployment.yaml
kubectl apply -f nba-service.yaml
kubectl apply -f nba-hpa.yaml
kubectl apply -f nba-ingress.yaml
```

### 5. Verify Deployment
```bash
# Check all resources
kubectl get all -n nba-stats

# Check pods are running
kubectl get pods -n nba-stats

# Check HPA status
kubectl get hpa -n nba-stats

# Check ingress
kubectl get ingress -n nba-stats

# Check pod logs if needed
kubectl logs <pod-name> -n nba-stats
```

### 6. Configure Local Access

1. Get the Ingress IP:
```bash
kubectl get ingress nba-stats-ingress -n nba-stats
```

2. Add entry to `/etc/hosts`:
```bash
# Add this line (use sudo)
127.0.0.1 nba-stats.info
```

3. Start Minikube tunnel in a separate terminal:
```bash
minikube tunnel
```

### 7. Access the Application
Open your browser and navigate to:
```
http://nba-stats.info
```

## Testing and Modifying ConfigMap

### 1. View Current ConfigMap
```bash
# View ConfigMap details
kubectl get configmap nba-stats-config -n nba-stats -o yaml

# Check environment variables in pods
kubectl exec -it <pod-name> -n nba-stats -- env | grep -E "TITLE|GAMES|SHOW"
```

### 2. Modify ConfigMap Values
There are several ways to update the ConfigMap:

#### Method 1: Edit directly
```bash
kubectl edit configmap nba-stats-config -n nba-stats
```
This opens your default editor. Modify values and save.

#### Method 2: Update and reapply
Modify your `nba-configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nba-stats-config
  namespace: nba-stats
data:
  APP_TITLE: "My Custom NBA Dashboard"  # Changed title
  RECENT_GAMES_COUNT: "50"              # Changed from 100
  SHOW_PREDICTIONS: "true"
  SHOW_ADVANCED_STATS: "true"
```
Then apply:
```bash
kubectl apply -f nba-configmap.yaml
```

### 3. Apply Changes
After updating the ConfigMap, you need to restart the pods to pick up new values:
```bash
# Method 1: Delete pods (they will automatically recreate)
kubectl get pods -n nba-stats -l app=nba-stats-app -o name | xargs kubectl delete -n nba-stats

# Method 2: Rollout restart deployment
kubectl rollout restart deployment nba-stats-app -n nba-stats
```

### 4. Verify Changes
```bash
# Watch pods recreate
kubectl get pods -n nba-stats -w

# Verify new environment variables
kubectl exec -it <new-pod-name> -n nba-stats -- env | grep -E "TITLE|GAMES|SHOW"

# Check application in browser
# The title should be updated if you changed APP_TITLE
```

## Testing Autoscaling

To test the Horizontal Pod Autoscaler, use this load generator:

```yaml
# load-generator.yaml
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
  namespace: nba-stats
spec:
  containers:
  - name: load-generator
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - "while true; do wget -q -O- http://nba-stats-service:8501; done"
```

Deploy and monitor:
```bash
# Apply load generator
kubectl apply -f load-generator.yaml

# Watch HPA
kubectl get hpa nba-stats-hpa -n nba-stats --watch

# Watch pods in another terminal
kubectl get pods -n nba-stats --watch
```

The HPA will:
- Scale up when CPU utilization exceeds 50%
- Add pods gradually (50% at a time)
- Scale down conservatively (60-second window)
- Maintain between 2-5 pods

## Troubleshooting

1. **Pods not starting:**
```bash
# Check pod details
kubectl describe pod <pod-name> -n nba-stats

# Check pod logs
kubectl logs <pod-name> -n nba-stats
```

2. **Service not accessible:**
```bash
# Check service
kubectl describe service nba-stats-service -n nba-stats

# Check endpoints
kubectl get endpoints nba-stats-service -n nba-stats
```

3. **Ingress issues:**
```bash
# Check ingress status
kubectl describe ingress nba-stats-ingress -n nba-stats

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

4. **HPA not scaling:**
```bash
# Check metrics availability
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"

# Check HPA status
kubectl describe hpa nba-stats-hpa -n nba-stats
```

## Cleanup

To remove all resources:
```bash
# Delete namespace and all resources
kubectl delete namespace nba-stats

# Stop minikube tunnel (Ctrl+C in tunnel terminal)

# Optional: Stop minikube
minikube stop
```

## Configuration Options

The application can be configured through the ConfigMap:

| Parameter | Description | Default |
|-----------|-------------|---------|
| APP_TITLE | Dashboard title | "NBA Stats Kubernetes Edition :)" |
| RECENT_GAMES_COUNT | Number of recent games | "100" |
| SHOW_PREDICTIONS | Enable predictions | "true" |
| SHOW_ADVANCED_STATS | Enable advanced stats | "true" |

## Contributing

Feel free to contribute by:
1. Forking the repository
2. Creating a feature branch
3. Committing your changes
4. Opening a pull request
