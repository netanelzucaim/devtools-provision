# Woodpecker CI Installation Guide for Kind Cluster

This guide provides step-by-step instructions for installing Woodpecker CI on a Kind (Kubernetes in Docker) cluster with access via port-forward on localhost:9000.

## About Versions

This setup uses:
- **Woodpecker CI v2.7.0** - The actual Woodpecker CI application/software version
- **Helm Chart 1.6.0** - The packaging and deployment configuration version

These are two different version numbers:
- The **application version** (v2.7.0) is the version of Woodpecker CI software itself
- The **Helm chart version** (1.6.0) is the version of the Helm chart that packages and deploys Woodpecker CI

Both versions are tested to work together and are suitable for Kind cluster deployments.

## Security Considerations

**⚠️ Important Security Notes:**

1. **Agent Secret:** The default configuration includes a placeholder agent secret (`changeme`). This MUST be changed to a secure, randomly generated value before any production use. See Step 5 of the installation for instructions.

2. **Admin Username:** The default configuration specifies `netanelzucaim` as the admin user. Change this to your own GitHub username in `values.yaml` (WOODPECKER_ADMIN) to grant yourself admin privileges.

3. **GitHub OAuth:** If using GitHub OAuth, protect your client ID and secret. Never commit these to version control.

4. **Local Development Only:** This guide is optimized for local development with Kind. For production deployments, additional security measures are required (TLS, proper authentication, network policies, etc.).

## Table of Contents
- [Prerequisites](#prerequisites)
- [Kind Cluster Setup](#kind-cluster-setup)
- [Install Woodpecker CI with Helm](#install-woodpecker-ci-with-helm)
- [Access Woodpecker UI](#access-woodpecker-ui)
- [GitHub OAuth Setup (Optional)](#github-oauth-setup-optional)
- [Create Your First Pipeline](#create-your-first-pipeline)
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

Create a basic Kind cluster and use port-forward to access Woodpecker CI:

```bash
kind create cluster --name woodpecker-cluster
```

This is the simplest approach and works well with the port-forward method.

### Option 2: Kind Cluster with Extra Port Mappings (Alternative)

If you plan to use NodePort or LoadBalancer services in the future, you can create a cluster with extra port mappings:

```bash
cat <<EOF | kind create cluster --name woodpecker-cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30090
    hostPort: 30090
    protocol: TCP
EOF
```

**Note:** For this guide, we'll use port-forward (works with both options), so Option 1 is sufficient.

Verify the cluster is running:
```bash
kubectl cluster-info --context kind-woodpecker-cluster
kubectl get nodes
```

## Install Woodpecker CI with Helm

### Step 1: Create Namespace

```bash
kubectl create namespace woodpecker
```

### Step 2: Navigate to Charts Directory

```bash
cd devtools/woodpecker/charts
```

### Step 3: Configure Agent Secret (IMPORTANT!)

**⚠️ SECURITY WARNING:** The default `values.yaml` contains a placeholder agent secret (`changeme`) that MUST be changed before production use.

For local development/testing, you can proceed with the default value, but for any real use, generate a secure secret:

```bash
# Generate a secure random secret
AGENT_SECRET=$(openssl rand -hex 32)
echo "Generated Agent Secret: $AGENT_SECRET"

# Option 1: Set via command line during install (recommended)
helm install woodpecker . \
  --namespace woodpecker \
  --create-namespace \
  --set server.env.WOODPECKER_AGENT_SECRET="$AGENT_SECRET" \
  --set agent.env.WOODPECKER_AGENT_SECRET="$AGENT_SECRET" \
  --wait

# Option 2: Edit values.yaml manually
# Replace 'changeme' with your generated secret in both server and agent sections
```

**Note:** The agent secret must match between the server and agent configurations.

### Step 4: Install Woodpecker CI (Basic Installation)

If you're just testing locally and didn't set a custom secret in Step 3:

```bash
helm install woodpecker . \
  --namespace woodpecker \
  --create-namespace \
  --wait
```

**Alternative:** Install with custom release name:
```bash
helm install my-woodpecker . \
  --namespace woodpecker \
  --create-namespace \
  --wait
```

### Step 5: Wait for Pods to be Ready

```bash
kubectl wait --for=condition=ready pod \
  --all \
  --namespace woodpecker \
  --timeout=300s
```

Check pod status:
```bash
kubectl get pods -n woodpecker
```

You should see all pods in `Running` state:
```
NAME                                    READY   STATUS    RESTARTS   AGE
woodpecker-server-xxxx                  1/1     Running   0          2m
woodpecker-agent-xxxx                   1/1     Running   0          2m
woodpecker-agent-yyyy                   1/1     Running   0          2m
```

## Access Woodpecker UI

### Step 1: Set Up Port-Forward

Open a terminal and run:

```bash
kubectl port-forward svc/woodpecker-server -n woodpecker 9000:8000
```

Keep this terminal open. Port-forward will run in the foreground.

**Tip:** To run port-forward in the background on Linux/macOS:
```bash
kubectl port-forward svc/woodpecker-server -n woodpecker 9000:8000 > /dev/null 2>&1 &
```

### Step 2: Access the UI

1. Open your browser and navigate to: **http://localhost:9000**
2. On first access, you'll be prompted to log in
3. Since `WOODPECKER_OPEN` is set to `true`, you can register as a new user
4. The user specified in `WOODPECKER_ADMIN` (default: `netanelzucaim`) will automatically be granted admin privileges. **Update this in `values.yaml` to your GitHub username before installation.**

### Step 3: Log In

Woodpecker CI supports multiple authentication methods. By default, with the provided configuration, it runs in "open" mode where anyone can register. For GitHub OAuth setup, see the next section.

## GitHub OAuth Setup (Optional)

To enable GitHub OAuth authentication and automatic repository synchronization:

### Step 1: Create a GitHub OAuth Application

1. Go to GitHub Settings → Developer settings → OAuth Apps → New OAuth App
2. Fill in the application details:
   - **Application name:** Woodpecker CI Local
   - **Homepage URL:** http://localhost:9000
   - **Authorization callback URL:** http://localhost:9000/authorize
3. Click "Register application"
4. **Copy the Client ID** (you'll need this)
5. Click "Generate a new client secret"
6. **Copy the Client Secret** (you won't see it again)

### Step 2: Update Woodpecker Configuration

Edit the `values.yaml` file to enable GitHub OAuth:

```yaml
woodpecker:
  server:
    env:
      WOODPECKER_HOST: http://localhost:9000
      WOODPECKER_ADMIN: your-github-username  # Change to your GitHub username
      WOODPECKER_OPEN: false  # Disable open registration
      WOODPECKER_AGENT_SECRET: changeme  # Generate a secure secret!
      # GitHub OAuth
      WOODPECKER_GITHUB: "true"
      WOODPECKER_GITHUB_CLIENT: "your-github-client-id"
      WOODPECKER_GITHUB_SECRET: "your-github-client-secret"
```

### Step 3: Upgrade the Helm Release

```bash
cd devtools/woodpecker/charts
helm upgrade woodpecker . --namespace woodpecker
```

### Step 4: Restart Port-Forward and Log In

```bash
kubectl port-forward svc/woodpecker-server -n woodpecker 9000:8000
```

Navigate to http://localhost:9000 and click "Login" to authenticate via GitHub.

## Create Your First Pipeline

Once logged in and authenticated with GitHub, you can create your first pipeline:

### Step 1: Activate a Repository

1. In the Woodpecker UI, click on "Repositories"
2. You'll see a list of your GitHub repositories
3. Click the toggle switch next to a repository to activate it

### Step 2: Create a Pipeline Configuration

In your GitHub repository, create a file named `.woodpecker.yml` in the root:

```yaml
# .woodpecker.yml
pipeline:
  build:
    image: golang:1.21
    commands:
      - echo "Hello, Woodpecker CI!"
      - go version
      - go test ./...

  test:
    image: golang:1.21
    commands:
      - echo "Running tests..."
      - go test -v ./...
    when:
      branch: main
```

### Step 3: Trigger a Build

Push a commit to your repository:

```bash
git add .woodpecker.yml
git commit -m "Add Woodpecker CI pipeline"
git push
```

### Step 4: View the Build

1. Go back to the Woodpecker UI
2. Click on your repository
3. You should see the build running
4. Click on the build to view logs and details

## Verification

### 1. Check All Pods are Running

```bash
kubectl get pods -n woodpecker
```

All pods should be in `Running` state with `1/1` ready.

### 2. Check Services

```bash
kubectl get svc -n woodpecker
```

You should see the `woodpecker-server` service of type `ClusterIP`.

### 3. Access UI

- Open http://localhost:9000 (with port-forward running)
- Log in successfully
- See the Woodpecker dashboard

### 4. Check Agent Connection

In the Woodpecker UI:
1. Go to Admin → Agents (if you're an admin)
2. You should see 2 agents connected (as per `replicaCount: 2`)

Alternatively, check logs:
```bash
kubectl logs -n woodpecker -l app.kubernetes.io/component=agent
```

### 5. Verify Persistent Storage

```bash
kubectl get pvc -n woodpecker
```

You should see a PersistentVolumeClaim for Woodpecker data.

## Useful Commands

### Helm Commands

```bash
# List installed releases
helm list -n woodpecker

# Upgrade Woodpecker CI
helm upgrade woodpecker . -n woodpecker

# Uninstall Woodpecker CI
helm uninstall woodpecker -n woodpecker

# Get values
helm get values woodpecker -n woodpecker
```

### Kubectl Commands

```bash
# View logs for server
kubectl logs -n woodpecker -l app.kubernetes.io/component=server

# View logs for agents
kubectl logs -n woodpecker -l app.kubernetes.io/component=agent

# View logs for a specific pod
kubectl logs -n woodpecker <pod-name>

# Describe a pod
kubectl describe pod -n woodpecker <pod-name>

# Get all resources in woodpecker namespace
kubectl get all -n woodpecker

# Delete the woodpecker namespace (complete cleanup)
kubectl delete namespace woodpecker
```

### Port-Forward Management

```bash
# Find port-forward process
ps aux | grep "port-forward"

# Kill port-forward (if running in background)
pkill -f "port-forward.*woodpecker"

# Restart port-forward
kubectl port-forward svc/woodpecker-server -n woodpecker 9000:8000
```

### Woodpecker CLI (Optional)

Install the Woodpecker CLI for command-line management:

```bash
# On Linux
curl -L https://github.com/woodpecker-ci/woodpecker/releases/latest/download/woodpecker-cli_linux_amd64.tar.gz | tar xz
sudo mv woodpecker-cli /usr/local/bin/

# On macOS
brew tap woodpecker-ci/tap
brew install woodpecker-cli

# Configure CLI
export WOODPECKER_SERVER=http://localhost:9000
export WOODPECKER_TOKEN=<your-token>  # Get from UI: User Settings → API Token

# List repositories
woodpecker-cli repo ls

# View build info
woodpecker-cli build info <repo> <build-number>
```

## Troubleshooting

### Issue 1: Pods Not Starting

**Symptoms:** Pods stuck in `Pending`, `ImagePullBackOff`, or `CrashLoopBackOff` state.

**Solutions:**
```bash
# Check pod status
kubectl get pods -n woodpecker

# Describe problematic pod
kubectl describe pod -n woodpecker <pod-name>

# Check logs
kubectl logs -n woodpecker <pod-name>

# Common fixes:
# - Increase Docker memory (Docker Desktop: Settings → Resources → Memory → 4GB+)
# - Check if Docker is running
# - Delete and recreate the pod:
kubectl delete pod -n woodpecker <pod-name>
```

### Issue 2: Can't Access UI on localhost:9000

**Symptoms:** Browser shows "Connection refused" or "Unable to connect"

**Solutions:**
1. Verify port-forward is running:
   ```bash
   ps aux | grep port-forward
   ```

2. Check if port 9000 is already in use:
   ```bash
   # Linux/macOS
   lsof -i :9000
   
   # Windows
   netstat -ano | findstr :9000
   ```

3. Try a different port:
   ```bash
   kubectl port-forward svc/woodpecker-server -n woodpecker 9090:8000
   # Then access http://localhost:9090
   ```

4. Verify the service exists:
   ```bash
   kubectl get svc -n woodpecker woodpecker-server
   ```

### Issue 3: Agent Not Connecting to Server

**Symptoms:** No agents shown in admin panel, builds stuck in queue.

**Solutions:**
```bash
# Check agent logs
kubectl logs -n woodpecker -l app.kubernetes.io/component=agent

# Common causes:
# - WOODPECKER_AGENT_SECRET mismatch between server and agent
# - Network connectivity issues

# Verify secret matches in both server and agent
kubectl get pods -n woodpecker -o yaml | grep WOODPECKER_AGENT_SECRET

# Restart agents
kubectl rollout restart deployment/woodpecker-agent -n woodpecker
```

### Issue 4: GitHub OAuth Not Working

**Symptoms:** Can't log in with GitHub, redirect errors.

**Solutions:**
1. Verify OAuth app settings in GitHub:
   - Homepage URL must be `http://localhost:9000`
   - Callback URL must be `http://localhost:9000/authorize`

2. Check environment variables:
   ```bash
   kubectl get pods -n woodpecker -l app.kubernetes.io/component=server -o yaml | grep GITHUB
   ```

3. Ensure Client ID and Secret are correct in `values.yaml`

4. Restart the server:
   ```bash
   kubectl rollout restart deployment/woodpecker-server -n woodpecker
   ```

### Issue 5: Builds Not Triggering

**Symptoms:** Push to GitHub doesn't trigger builds.

**Solutions:**
1. Check repository is activated in Woodpecker UI
2. Verify webhook is configured in GitHub:
   - Go to repository → Settings → Webhooks
   - Should see a webhook pointing to Woodpecker
3. Check webhook deliveries for errors
4. Verify `.woodpecker.yml` exists in repository root
5. Check server logs:
   ```bash
   kubectl logs -n woodpecker -l app.kubernetes.io/component=server
   ```

### Issue 6: Persistent Volume Issues

**Symptoms:** Data loss after pod restart, PVC in `Pending` state.

**Solutions:**
```bash
# Check PVC status
kubectl get pvc -n woodpecker

# Describe PVC for issues
kubectl describe pvc -n woodpecker <pvc-name>

# For Kind clusters, ensure storage provisioner is available:
kubectl get storageclass

# If using custom storage class, update values.yaml:
woodpecker:
  server:
    persistentVolume:
      storageClass: "your-storage-class"
```

### Issue 7: Out of Resources (Memory/CPU)

**Symptoms:** Pods evicted, Kind cluster unresponsive, or OOMKilled errors.

**Solutions:**
1. Increase Docker resources (Settings → Resources):
   - Memory: 6GB+ recommended
   - CPU: 4+ cores recommended

2. Reduce Woodpecker resource requests in `values.yaml`:
   ```yaml
   server:
     resources:
       requests:
         cpu: 50m
         memory: 64Mi
   ```

3. Reduce agent count:
   ```yaml
   agent:
     replicaCount: 1
   ```

4. Restart Kind cluster:
   ```bash
   kind delete cluster --name woodpecker-cluster
   kind create cluster --name woodpecker-cluster
   ```

### Issue 8: Pipeline Fails with "Cannot connect to Docker daemon"

**Symptoms:** Builds fail with Docker daemon connection errors.

**Solutions:**
1. Woodpecker agents need access to Docker. In Kind, this requires:
   - Docker-in-Docker (DinD) setup, or
   - Host Docker socket mounting (not recommended in production)

2. For local development with Kind, you may need to use the Kubernetes backend instead:
   ```yaml
   agent:
     env:
       WOODPECKER_BACKEND: kubernetes
   ```

3. Ensure agents have proper RBAC permissions for Kubernetes backend

### Getting More Help

- **Woodpecker CI Documentation:** https://woodpecker-ci.org/docs/
- **Woodpecker CI GitHub:** https://github.com/woodpecker-ci/woodpecker
- **Kind Documentation:** https://kind.sigs.k8s.io/
- **Helm Documentation:** https://helm.sh/docs/

For issues specific to this installation, check:
- `kubectl logs` for pod errors
- `kubectl describe pod` for resource/scheduling issues
- Woodpecker server logs for application-level issues
