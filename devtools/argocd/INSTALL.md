# Argo CD Installation Guide for Kind Cluster

This guide provides step-by-step instructions for installing Argo CD on a Kind (Kubernetes in Docker) cluster with access via port-forward on localhost:8080.

## About Versions

This setup uses:
- **Argo CD v2.13.0** - The actual Argo CD application/software version
- **Helm Chart 7.7.5** - The packaging and deployment configuration version

These are two different version numbers:
- The **application version** (v2.13.0) is the version of Argo CD software itself
- The **Helm chart version** (7.7.5) is the version of the Helm chart that packages and deploys Argo CD

Both versions are tested to work together and are suitable for Kind cluster deployments.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Kind Cluster Setup](#kind-cluster-setup)
- [Install Argo CD with Helm](#install-argo-cd-with-helm)
- [Access Argo CD UI](#access-argo-cd-ui)
- [GitHub Authentication (Optional)](#github-authentication-optional)
- [Verification](#verification)
- [Useful Commands](#useful-commands)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before starting, ensure you have the following tools installed:

1. **Docker** - For running Kind
   ```bash
   docker --version
   # Should show Docker version 20.10+ or higher
   ```

2. **Kind** - Kubernetes in Docker
   ```bash
   kind version
   # Should show kind v0.20.0 or higher
   ```
   
   Install if needed:
   ```bash
   # On Linux
   curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   
   # On macOS
   brew install kind
   
   # On Windows (PowerShell)
   curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.20.0/kind-windows-amd64
   Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe
   ```

3. **kubectl** - Kubernetes CLI
   ```bash
   kubectl version --client
   # Should show version v1.27.0 or higher
   ```

4. **Helm 3.x** - Kubernetes package manager
   ```bash
   helm version
   # Should show version v3.12.0 or higher
   ```
   
   Install if needed:
   ```bash
   # On Linux
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   
   # On macOS
   brew install helm
   
   # On Windows (Chocolatey)
   choco install kubernetes-helm
   ```

## Kind Cluster Setup

You have two options for creating your Kind cluster:

### Option 1: Standard Kind Cluster (Recommended)

Create a basic Kind cluster and use port-forward to access Argo CD:

```bash
kind create cluster --name argocd-cluster
```

This is the simplest approach and works well with the port-forward method.

### Option 2: Kind Cluster with Extra Port Mappings (Alternative)

If you plan to use NodePort or LoadBalancer services in the future, you can create a cluster with extra port mappings:

```bash
cat <<EOF | kind create cluster --name argocd-cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 30080
    protocol: TCP
  - containerPort: 30443
    hostPort: 30443
    protocol: TCP
EOF
```

**Note:** For this guide, we'll use port-forward (works with both options), so Option 1 is sufficient.

Verify the cluster is running:
```bash
kubectl cluster-info --context kind-argocd-cluster
kubectl get nodes
```

## Install Argo CD with Helm

### Step 1: Add Argo CD Helm Repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### Step 2: Create Namespace

```bash
kubectl create namespace argocd
```

### Step 3: Navigate to Charts Directory

```bash
cd devtools/argocd/charts
```

### Step 4: Update Helm Dependencies

```bash
helm dependency update
```

This command downloads the Argo CD chart specified in `Chart.yaml`.

### Step 5: Install Argo CD

```bash
helm install argocd . \
  --namespace argocd \
  --create-namespace \
  --wait
```

**Alternative:** Install with custom release name:
```bash
helm install my-argocd . \
  --namespace argocd \
  --create-namespace \
  --wait
```

### Step 6: Wait for Pods to be Ready

```bash
kubectl wait --for=condition=ready pod \
  --all \
  --namespace argocd \
  --timeout=300s
```

Check pod status:
```bash
kubectl get pods -n argocd
```

You should see all pods in `Running` state:
```
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          2m
argocd-applicationset-controller-xxxx              1/1     Running   0          2m
argocd-redis-xxxx                                  1/1     Running   0          2m
argocd-repo-server-xxxx                            1/1     Running   0          2m
argocd-server-xxxx                                 1/1     Running   0          2m
```

## Access Argo CD UI

### Step 1: Get Initial Admin Password

The initial admin password is auto-generated and stored in a Kubernetes secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**Save this password!** You'll need it to log in.

**Note:** If the command doesn't work on Windows, try:
```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Step 2: Set Up Port-Forward

Open a terminal and run:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Keep this terminal open. Port-forward will run in the foreground.

**Tip:** To run port-forward in the background on Linux/macOS:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 > /dev/null 2>&1 &
```

### Step 3: Access the UI

1. Open your browser and navigate to: **http://localhost:8080**
2. You may see a certificate warning (this is expected with `--insecure` mode). Accept it.
3. Log in with:
   - **Username:** `admin`
   - **Password:** (the password from Step 1)

### Step 4: Change Admin Password (Recommended)

After logging in, change the admin password:

**Via UI:**
1. Click on the user icon (top right)
2. Select "User Info"
3. Click "Update Password"

**Via CLI:**
```bash
# Install Argo CD CLI first (optional)
# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# macOS
brew install argocd

# Then change password
argocd login localhost:8080
argocd account update-password
```

## GitHub Authentication (Optional)

To connect Argo CD to private GitHub repositories, you need to configure authentication.

### Option 1: Using Personal Access Token (PAT)

1. **Create a GitHub Personal Access Token:**
   - Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Click "Generate new token (classic)"
   - Give it a name (e.g., "Argo CD")
   - Select scopes: `repo` (full control of private repositories)
   - Click "Generate token"
   - **Copy the token immediately** (you won't see it again)

2. **Add Repository via CLI:**
   ```bash
   argocd repo add https://github.com/YOUR_USERNAME/YOUR_REPO.git \
     --username YOUR_GITHUB_USERNAME \
     --password YOUR_GITHUB_TOKEN
   ```

3. **Or via UI:**
   - Go to Settings → Repositories
   - Click "Connect Repo"
   - Choose "VIA HTTPS"
   - Enter repository URL
   - Enter GitHub username and token
   - Click "Connect"

### Option 2: Using SSH Key

1. **Generate SSH Key (if you don't have one):**
   ```bash
   ssh-keygen -t ed25519 -C "argocd@localhost" -f ~/.ssh/argocd_rsa
   ```

2. **Add SSH Public Key to GitHub:**
   - Copy the public key:
     ```bash
     cat ~/.ssh/argocd_rsa.pub
     ```
   - Go to GitHub Settings → SSH and GPG keys → New SSH key
   - Paste the public key and save

3. **Add Repository to Argo CD:**
   ```bash
   argocd repo add git@github.com:YOUR_USERNAME/YOUR_REPO.git \
     --ssh-private-key-path ~/.ssh/argocd_rsa
   ```

### Verify Repository Connection

Check connected repositories:
```bash
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

Or via UI: Settings → Repositories (should show "Successful" connection status)

## Verification

### 1. Check All Pods are Running

```bash
kubectl get pods -n argocd
```

All pods should be in `Running` state with `1/1` or more ready.

### 2. Check Services

```bash
kubectl get svc -n argocd
```

You should see the `argocd-server` service of type `ClusterIP`.

### 3. Access UI

- Open http://localhost:8080 (with port-forward running)
- Log in successfully with admin credentials

### 4. Check Pre-configured Repository

The `devtools-provision` repository should be pre-configured:
- Go to Settings → Repositories in the UI
- You should see `https://github.com/netanelzucaim/devtools-provision.git`

### 5. Create a Test Application (Optional)

Create a simple test application to verify Argo CD is working:

```yaml
# test-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/netanelzucaim/devtools-provision.git
    targetRevision: HEAD
    path: test-app  # You'll need to create this path with k8s manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it:
```bash
kubectl apply -f test-app.yaml
```

## Useful Commands

### Helm Commands

```bash
# List installed releases
helm list -n argocd

# Upgrade Argo CD
helm upgrade argocd . -n argocd

# Uninstall Argo CD
helm uninstall argocd -n argocd

# Get values
helm get values argocd -n argocd
```

### Kubectl Commands

```bash
# View logs for a specific pod
kubectl logs -n argocd deployment/argocd-server

# Describe a pod
kubectl describe pod -n argocd <pod-name>

# Get all resources in argocd namespace
kubectl get all -n argocd

# Delete the argocd namespace (complete cleanup)
kubectl delete namespace argocd
```

### Argo CD CLI Commands

```bash
# Login to Argo CD
argocd login localhost:8080

# List applications
argocd app list

# Get application details
argocd app get <app-name>

# Sync an application
argocd app sync <app-name>

# Delete an application
argocd app delete <app-name>

# List repositories
argocd repo list

# Add a repository
argocd repo add <repo-url>
```

### Port-Forward Management

```bash
# Find port-forward process
ps aux | grep "port-forward"

# Kill port-forward (if running in background)
pkill -f "port-forward.*argocd"

# Restart port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Troubleshooting

### Issue 1: Pods Not Starting

**Symptoms:** Pods stuck in `Pending`, `ImagePullBackOff`, or `CrashLoopBackOff` state.

**Solutions:**
```bash
# Check pod status
kubectl get pods -n argocd

# Describe problematic pod
kubectl describe pod -n argocd <pod-name>

# Check logs
kubectl logs -n argocd <pod-name>

# Common fixes:
# - Increase Docker memory (Docker Desktop: Settings → Resources → Memory → 4GB+)
# - Check if Docker is running
# - Delete and recreate the pod:
kubectl delete pod -n argocd <pod-name>
```

### Issue 2: Can't Access UI on localhost:8080

**Symptoms:** Browser shows "Connection refused" or "Unable to connect"

**Solutions:**
1. Verify port-forward is running:
   ```bash
   ps aux | grep port-forward
   ```

2. Check if port 8080 is already in use:
   ```bash
   # Linux/macOS
   lsof -i :8080
   
   # Windows
   netstat -ano | findstr :8080
   ```

3. Try a different port:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 9090:443
   # Then access http://localhost:9090
   ```

4. Verify the service exists:
   ```bash
   kubectl get svc -n argocd argocd-server
   ```

### Issue 3: Can't Retrieve Admin Password

**Symptoms:** `argocd-initial-admin-secret` not found or command returns empty.

**Solutions:**
```bash
# Check if secret exists
kubectl get secrets -n argocd

# If missing, the pod might still be initializing. Wait and try again:
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Alternatively, reset the admin password:
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": { "admin.password": "'$(htpasswd -bnBC 10 "" your-new-password | tr -d ':\n')'"}}'
```

### Issue 4: Certificate/TLS Errors in Browser

**Symptoms:** Browser shows security warnings or "NET::ERR_CERT_AUTHORITY_INVALID"

**Solutions:**
- This is expected with `--insecure` mode
- Click "Advanced" and proceed to the site (safe for local development)
- Or install Argo CD CLI and use `argocd login --insecure localhost:8080`

### Issue 5: Application Sync Fails

**Symptoms:** Applications show "OutOfSync" or sync fails with errors.

**Solutions:**
```bash
# Check application status
argocd app get <app-name>

# View sync errors in UI: Click on app → View sync details

# Common causes:
# - Repository authentication issues (check Settings → Repositories)
# - Invalid manifest path
# - RBAC issues

# Force sync
argocd app sync <app-name> --force
```

### Issue 6: Helm Dependency Update Fails

**Symptoms:** `helm dependency update` fails with connection errors.

**Solutions:**
```bash
# Check network connectivity
curl -I https://argoproj.github.io/argo-helm/

# Update Helm repos
helm repo update

# Clear Helm cache
rm -rf ~/.cache/helm
helm repo update

# Try updating dependencies again
helm dependency update
```

### Issue 7: Out of Resources (Memory/CPU)

**Symptoms:** Pods evicted, Kind cluster unresponsive, or OOMKilled errors.

**Solutions:**
1. Increase Docker resources (Settings → Resources):
   - Memory: 6GB+ recommended
   - CPU: 4+ cores recommended

2. Reduce Argo CD resource requests in `values.yaml`:
   ```yaml
   controller:
     resources:
       requests:
         cpu: 100m
         memory: 256Mi
   ```

3. Restart Kind cluster:
   ```bash
   kind delete cluster --name argocd-cluster
   kind create cluster --name argocd-cluster
   ```

### Issue 8: Cannot Connect to Private GitHub Repo

**Symptoms:** Repository shows "Failed" connection status or applications can't sync.

**Solutions:**
1. Verify credentials are correct
2. Check token/SSH key has proper permissions
3. For PAT: Ensure `repo` scope is enabled
4. For SSH: Verify key is added to GitHub and has access to repo
5. Test connection manually:
   ```bash
   # For HTTPS with token
   git clone https://YOUR_TOKEN@github.com/username/repo.git
   
   # For SSH
   ssh -T git@github.com
   ```

### Getting More Help

- **Argo CD Documentation:** https://argo-cd.readthedocs.io/
- **Argo CD GitHub Issues:** https://github.com/argoproj/argo-cd/issues
- **Kind Documentation:** https://kind.sigs.k8s.io/
- **Helm Documentation:** https://helm.sh/docs/

For issues specific to this installation, check:
- `kubectl logs` for pod errors
- `kubectl describe pod` for resource/scheduling issues
- Argo CD UI logs (Settings → Logs)
