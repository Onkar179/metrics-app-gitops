## Objective
Deploy a containerized app that exposes a /counter endpoint using Helm, ArgoCD, and KIND, and investigate any issues with the app behavior.

### Helm Chart Creation

#### Directory structure:

```bash
metrics-app/
├── charts/                  # Helm sub-charts (empty for now)
├── templates/               # Template files for K8s resources
│   ├── deployment.yaml      # Kubernetes Deployment
│   ├── service.yaml         # Kubernetes Service
│   ├── ingress.yaml         # Ingress to expose /counter
├── values.yaml              # Default configuration values
├── Chart.yaml               # Helm chart metadata
```

### Kind Cluster Setup on Mac

- **Install Kind:**

    This will install Kind on your local machine:

    ```bash
    brew install kind
    ```

- *We need to open some ports & mapping to make Ingress work on the Kind cluster setup on the local. Below is the config we are using to create a kind cluster.*

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 30443 #argocd-server node port access
    hostPort: 8080
    protocol: TCP
  - containerPort: 80  #nginx Ingress ports
    hostPort: 80
    protocol: TCP
  - containerPort: 443 #nginx Ingress ports
    hostPort: 443
    protocol: TCP

```

- **Create a kind cluster**
    ```bash
      kind create cluster --name argocd-kind --config kind-config.yaml
    ```

### ArgoCD Installation & Setup

  ```bash
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```

- *To create a NodePort service with ports 80 and 443, which will be mapped to ports 8080 on the ArgoCD server, use the following command:*
```bash
  kubectl patch svc argocd-server -n argocd -p \
  '{"spec": {"type": "NodePort", "ports": [{"name": "http", "nodePort": 30080, "port": 80, "protocol": "TCP", "targetPort": 8080}, {"name": "https", "nodePort": 30443, "port": 443, "protocol": "TCP", "targetPort": 8080}]}}'
```

- *Now we can access ArgoCD UI over localhost:8080 & login with the default admin user with password retrieved from the argocd secrets.*

```bash
kubectl get secret \
  -n argocd \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" |
  base64 -d &&
  echo
```

## Application Secret Management

- **App needs a secret PASSWORD set to MYPASSWORD, available as an environment variable**

As part of best security practices, we are storing the secret on a cloud provider's secret management solution. In our case, it's GCP secret manager & can be any secret management     solutions like Vault, etc. As we are performing the exercise over a local machine since we can't use workload identity or GCP secret store solution to fetch the secret. Here we are using the external-secrets solution, which supports multiple backend secret solutions.

- **GCP Secret Manager steps**

1. Creating the secrets
```bash
gcloud secrets create metrics-app-password \
  --replication-policy="automatic"
```

2. Storing the secret value
```
echo -n "MYPASSWORD" | gcloud secrets versions add metrics-app-password \
  --data-file=-
```

3. Set up GCP ADC login
```
gcloud auth application-default login
```

## External Secret Manager setup

### Installation using Helm

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm upgrade --install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

[Optional: If you are using workload identity over a cloud platform | In our case, we are on the local machine since it's required]

As we are on the local machine, since workload identity is unsupported. For the same, we are mounting the ADC login token as volume over the external-secrets container for GCP access.

- **Copy the ADC file into a Kubernetes secret**
```bash
kubectl create namespace external-secrets --dry-run=client -o yaml | kubectl apply -f -
```

- **Creating GCP ADC secret (Only required when on local machine)**
```bash
kubectl create secret generic gcp-credentials \
  --from-file=creds.json=$HOME/.config/gcloud/application_default_credentials.json \
  -n external-secrets
```

- **Patch the External Secrets Operator deployment**

```bash
kubectl edit deployment external-secrets -n external-secrets
```

- **Editing the external-secrets deployment to mount the GCP ADC secret volume**

```bash
Volume Mount
volumeMounts:
  - name: gcp-creds
    mountPath: /var/secrets/gcp
    readOnly: true

Env Variable
env:
  - name: GOOGLE_APPLICATION_CREDENTIALS
    value: /var/secrets/gcp/creds.json

Volumes section
Under spec.template.spec, add:
volumes:
  - name: gcp-creds
    secret:
      secretName: gcp-credentials
```

- **Restart External Secrets Deployment**
```bash
kubectl rollout restart deployment external-secrets -n external-secrets
```

- **Create the SecretStore using ADC-based auth**
```bash
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secretstore
  namespace: metrics-app
spec:
  provider:
    gcpsm:
      projectID: "champions-project-423913"
```

- **Create the ExternalSecret resource**
```bash
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: metrics-app-secret
  namespace: metrics-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secretstore
    kind: SecretStore
  target:
    name: metrics-app-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: metrics-app-password
```
```bash
kubectl describe secret metrics-app-secret
```
- **You should see the dynamically synced secret from GCP. (No Secret Leakage or local storage threat)**

## ArgoCD project & Application deployment

- **Create ArgoCD project**

```bash
kubectl apply -f project.yaml

apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: metrics-project
  namespace: argocd
spec:
  description: Project for Metrics App
  sourceRepos:
    - 'https://github.com/Onkar179/metrics-app-gitops.git'
  destinations:
    - namespace: metrics-app
      server: https://kubernetes.default.svc
  permitOnlyProjectScopedClusters: false
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  orphanedResources:
    warn: true
```

- **Create ArgoCD application**

```bash
kubectl apply -f application.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-app
  namespace: argocd
  labels:
    app.kubernetes.io/managed-by: argocd
spec:
  project: metrics-project
  source:
    repoURL: https://github.com/Onkar179/metrics-app-gitops.git
    targetRevision: HEAD
    path: helm-chart
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: metrics-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
      - Validate=true
      - PruneLast=true
```

## Application Behaviour

- ***Calling the /counter Ingress application endpoint multiple times & observed that /counter endpoint is fast on odd requests and slow on even ones (second, fourth, etc.).***

## Root Cause Analysis

- ***Based on the observation I first get into the metrics-app pod to see which python backend processes consuming more CPU. I found that apart from the main application there are few background python processes are running & consuming more CPU.***

```bash
root@metrics-app-6659485674-pcbzt:/app# top
top - 16:45:13 up  7:37,  0 user,  load average: 2.90, 2.54, 1.86
Tasks:   6 total,   3 running,   3 sleeping,   0 stopped,   0 zombie
%Cpu(s): 24.6 us,  0.2 sy,  0.0 ni, 75.1 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st 
MiB Mem :   7844.5 total,    152.1 free,   2713.3 used,   5203.6 buff/cache     
MiB Swap:   1024.0 total,   1018.7 free,      5.3 used.   5131.2 avail Mem 
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                                                                     
     14 root      20   0   13168   8704   5504 S  81.0   0.1  72:10.35 python3                                                                                                                                     
     17 root      20   0   13168   8704   5504 R  80.3   0.1  10:03.17 python3                                                                                                                                     
     20 root      20   0   13168   8704   5504 R  80.3   0.1   7:10.69 python3                                                                                                                                     
      1 root      20   0  187768  31488  10752 S   0.0   0.4   0:00.74 python                                                                                                                                      
     21 root      20   0    4476   3456   2944 S   0.0   0.0   0:00.02 bash                                                                                                                                        
    194 root      20   0    9080   4736   2688 R   0.0   0.1   0:00.00 top
```
          
```bash
root@metrics-app-6659485674-pcbzt:/app# ps -aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.3 187768 31488 ?        Ssl  15:12   0:00 python app.py
root          14 82.6  0.1  13168  8704 ?        S    15:17  72:13 python3 /tmp/updater.py
root          17 81.8  0.1  13168  8704 ?        R    16:32  10:06 python3 /tmp/metricsd.py
root          20 81.9  0.1  13168  8704 ?        S    16:36   7:13 python3 /tmp/heartbeat.py
root          21  0.0  0.0   4476  3456 pts/0    Ss   16:43   0:00 bash
root         195  0.0  0.0   8436  3968 pts/0    R+   16:45   0:00 ps -aux
```

- ***Upon checking the main application code (app.py) I discovered the below code block causing blocking background thread or process creation (This is triggered only when the counter is even).***

```bash
if counter % 2 == 0:
    metrics.trigger_background_collection()
```

- ***This means every even-numbered call triggers the trigger_background_collection() function. Which results in blocking background threads or processes, as we see in the ps -aux output.***

- ***I have also measured the network latency using the curl command to see how much the delay is in the request response from the application side.***

```bash
onkarnaik@Onkars-MacBook-Pro ~ % curl -w "time_total: %{time_total}\nhttp_code: %{http_code}\n" -o /dev/null -s http://localhost/counter  (Even Number Request)
time_total: 21.005726
http_code: 200
onkarnaik@Onkars-MacBook-Pro ~ % curl -w "time_total: %{time_total}\nhttp_code: %{http_code}\n" -o /dev/null -s http://localhost/counter  (Odd Number Request)
time_total: 0.007271
http_code: 200
```

## Possible Solution

- ***To make trigger_background_collection() not create any blocking threads & processes, we can implement threading to start the metrics.trigger_background_collection() asynchronously in the background as a separate non-blocking thread.***

- **Updated app.py**

```bash
from flask import Flask
import metrics
import utils
import threading  #for background threading

app = Flask(__name__)
counter = 0

@app.route('/')
def home():
    return "Metrics Dashboard 📈"

@app.route('/counter')
def counter_page():
    global counter
    counter += 1
    if counter % 2 == 0:
        # Run in a background thread to avoid blocking the response
        threading.Thread(target=metrics.trigger_background_collection).start()
    return f"Counter value: {counter}"

if __name__ == "__main__":
    utils.initialize_services()
    app.run(host="0.0.0.0", port=8080)
```

1.Threading.Thread(target=...): starts a separate thread that executes trigger_background_collection.

2.Start(): Begins execution.

3.This lets Flask return the response immediately, without waiting for the metrics collection to complete.


## References

External Secrets: https://external-secrets.io/latest/

Secret Manager: https://cloud.google.com/secret-manager/docs/overview

Kind: https://kind.sigs.k8s.io/

ArgoCD: https://argo-cd.readthedocs.io/en/stable/
