# Gitea Installation Guide for Kind Cluster

This guide provides step-by-step instructions for installing Gitea on a Kind (Kubernetes in Docker) cluster with access via port-forward on localhost:3000.

## About Versions

This setup uses:
- **Gitea v1.22.0** - The actual Gitea application/software version
- **Helm Chart 1.0.0** - This custom chart version

These are two different version numbers:
- The **application version** (v1.22.0) is the version of Gitea software itself
- The **Helm chart version** (1.0.0) is the version of this custom Helm chart that deploys Gitea

Both versions are tested to work together and are suitable for Kind cluster deployments.

## Security Considerations

**⚠️ Important Security Notes:**

1. **Admin Password:** The default configuration includes a placeholder admin password (`changeme`). **This is a well-known weak credential and MUST be changed immediately, even for local development.** Generate a secure password using the installation commands below.

2. **Secret Keys:** Gitea will auto-generate SECRET_KEY and INTERNAL_TOKEN on first startup. For production, you should set these explicitly.

3. **Registration:** By default, user registration is enabled. For production, consider setting `DISABLE_REGISTRATION: true` after creating necessary accounts.

4. **Local Development Only:** This guide is optimized for local development with Kind. For production deployments, additional security measures are required (TLS, proper authentication, network policies, etc.).

## Table of Contents
- [Prerequisites](#prerequisites)
- [Kind Cluster Setup](#kind-cluster-setup)
- [Install Gitea with Helm](#install-gitea-with-helm)
- [Access Gitea UI](#access-gitea-ui)
- [Initial Setup](#initial-setup)
- [Create Your First Repository](#create-your-first-repository)
- [SSH Access Configuration](#ssh-access-configuration)
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

Create a basic Kind cluster and use port-forward to access Gitea:

```bash
kind create cluster --name gitea-cluster
```

This is the simplest approach and works well with the port-forward method.

### Option 2: Kind Cluster with Extra Port Mappings (Alternative)

If you plan to use NodePort or LoadBalancer services in the future, you can create a cluster with extra port mappings:

```bash
cat <<EOF | kind create cluster --name gitea-cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30300
    hostPort: 30300
    protocol: TCP
  - containerPort: 30022
    hostPort: 30022
    protocol: TCP
EOF
```

**Note:** For this guide, we'll use port-forward (works with both options), so Option 1 is sufficient.

Verify the cluster is running:
```bash
kubectl cluster-info --context kind-gitea-cluster
kubectl get nodes
```

## Install Gitea with Helm

### Step 1: Create Namespace

```bash
kubectl create namespace gitea
```

### Step 2: Navigate to Gitea Directory

```bash
cd devtools/gitea
```

**Note:** This is a self-contained Helm chart with Gitea templates included. No external Helm repository dependencies are required.

### Step 3: Install Gitea

**⚠️ SECURITY WARNING:** Do not use the default admin password! Always generate a secure password.

**Recommended Installation (with secure password):**

Generate a secure password first:
```bash
GITEA_ADMIN_PASSWORD=$(openssl rand -base64 32)
echo "Generated Admin Password: $GITEA_ADMIN_PASSWORD"
echo "IMPORTANT: Save this password! You'll need it to log in."

helm install gitea . \
  --namespace gitea \
  --create-namespace \
  --set gitea.admin.password="$GITEA_ADMIN_PASSWORD" \
  --wait
```

**Alternative: Basic Installation** (will use default password - change immediately after install):

```bash
helm install gitea . \
  --namespace gitea \
  --create-namespace \
  --wait
```

**Custom Release Name:**
```bash
helm install my-gitea . \
  --namespace gitea \
  --create-namespace \
  --set gitea.admin.password="$(openssl rand -base64 32)" \
  --wait
```

### Step 4: Wait for Pods to be Ready

```bash
kubectl wait --for=condition=ready pod \
  --all \
  --namespace gitea \
  --timeout=300s
```

Check pod status:
```bash
kubectl get pods -n gitea
```

You should see all pods in `Running` state:
```
NAME                                READY   STATUS    RESTARTS   AGE
gitea-0                             1/1     Running   0          2m
```

## Access Gitea UI

### Step 1: Set Up Port-Forward for HTTP

Open a terminal and run:

```bash
kubectl port-forward svc/gitea-http -n gitea 3000:3000
```

Keep this terminal open. Port-forward will run in the foreground.

**Tip:** To run port-forward in the background on Linux/macOS:
```bash
kubectl port-forward svc/gitea-http -n gitea 3000:3000 > /dev/null 2>&1 &
```

### Step 2: Access the UI

1. Open your browser and navigate to: **http://localhost:3000**
2. You'll see the Gitea initial setup page (on first access)

## Initial Setup

### Step 1: Complete Initial Configuration

On first access, Gitea will show an installation page:

1. **Database Settings:**
   - Database Type: SQLite3 (already configured)
   - Path: `/data/gitea.db` (already configured)

2. **General Settings:**
   - Site Title: `Gitea: Git with a cup of tea` (or customize)
   - Server Domain: `localhost`
   - Gitea Base URL: `http://localhost:3000`
   - SSH Server Port: `22`

3. **Administrator Account Settings:**
   - Username: `gitea_admin` (or customize)
   - Password: (use the password you generated, or change the default)
   - Email: `admin@example.com` (or use your email)

4. Click **"Install Gitea"**

### Step 2: Log In

After installation completes:
1. You'll be redirected to the login page
2. Log in with the admin credentials you just created
3. You'll see the Gitea dashboard

### Step 3: Change Admin Password (If Using Default)

If you used the default password, change it immediately:

1. Click on your avatar (top right)
2. Select "Settings"
3. Go to "Account" tab
4. Click "Password" section
5. Enter old password and new password
6. Click "Update Password"

## Create Your First Repository

### Step 1: Create a New Repository

1. Click the **"+"** button in the top right corner
2. Select **"New Repository"**
3. Fill in repository details:
   - Owner: Select your username or organization
   - Repository Name: e.g., `my-first-repo`
   - Description: (optional)
   - Visibility: Public or Private
   - Initialize Repository: Check to add README
4. Click **"Create Repository"**

### Step 2: Clone Your Repository

You have two options for cloning:

**Option 1: Clone via HTTPS (Easier)**

```bash
# Clone the repository
git clone http://localhost:3000/gitea_admin/my-first-repo.git
cd my-first-repo

# Make a change
echo "Hello, Gitea!" >> hello.txt
git add hello.txt
git commit -m "Add hello.txt"

# Push the change
git push origin main
```

**Note:** You'll be prompted for your Gitea username and password when pushing.

**Option 2: Clone via SSH (More Secure)**

See the [SSH Access Configuration](#ssh-access-configuration) section below.

### Step 3: View Your Changes

1. Go back to the Gitea UI (http://localhost:3000)
2. Navigate to your repository
3. You should see the new file `hello.txt`

## SSH Access Configuration

To use Git over SSH (recommended for production):

### Step 1: Set Up Port-Forward for SSH

Open another terminal:

```bash
kubectl port-forward svc/gitea-ssh -n gitea 2222:22
```

Keep this terminal open.

### Step 2: Generate SSH Key (if you don't have one)

```bash
ssh-keygen -t ed25519 -C "your-email@example.com" -f ~/.ssh/gitea_ed25519
```

### Step 3: Add SSH Public Key to Gitea

1. Copy your public key:
   ```bash
   cat ~/.ssh/gitea_ed25519.pub
   ```

2. In Gitea UI:
   - Click on your avatar → Settings
   - Go to "SSH / GPG Keys" tab
   - Click "Add Key"
   - Paste your public key
   - Give it a name (e.g., "My Laptop")
   - Click "Add Key"

### Step 4: Configure Git to Use Custom SSH Port

Add to your `~/.ssh/config`:

```
Host localhost-gitea
    HostName localhost
    Port 2222
    User git
    IdentityFile ~/.ssh/gitea_ed25519
```

### Step 5: Clone via SSH

```bash
git clone ssh://git@localhost-gitea/gitea_admin/my-first-repo.git
```

Or for existing clones, update the remote:
```bash
git remote set-url origin ssh://git@localhost-gitea/gitea_admin/my-first-repo.git
```

## Verification

### 1. Check All Pods are Running

```bash
kubectl get pods -n gitea
```

All pods should be in `Running` state with `1/1` ready.

### 2. Check Services

```bash
kubectl get svc -n gitea
```

You should see the `gitea-http` and `gitea-ssh` services.

### 3. Access UI

- Open http://localhost:3000 (with HTTP port-forward running)
- Log in successfully with admin credentials
- See the Gitea dashboard

### 4. Verify Persistent Storage

```bash
kubectl get pvc -n gitea
```

You should see a PersistentVolumeClaim for Gitea data.

### 5. Create and Clone a Test Repository

Follow the [Create Your First Repository](#create-your-first-repository) section to verify Git operations work correctly.

## Useful Commands

### Helm Commands

```bash
# List installed releases
helm list -n gitea

# Upgrade Gitea
helm upgrade gitea . -n gitea

# Uninstall Gitea
helm uninstall gitea -n gitea

# Get values
helm get values gitea -n gitea
```

### Kubectl Commands

```bash
# View logs
kubectl logs -n gitea -l app=gitea

# View logs for specific pod
kubectl logs -n gitea <pod-name>

# Describe a pod
kubectl describe pod -n gitea <pod-name>

# Get all resources in gitea namespace
kubectl get all -n gitea

# Delete the gitea namespace (complete cleanup)
kubectl delete namespace gitea
```

### Port-Forward Management

```bash
# Find port-forward processes
ps aux | grep "port-forward"

# Kill port-forward (if running in background)
pkill -f "port-forward.*gitea"

# Restart HTTP port-forward
kubectl port-forward svc/gitea-http -n gitea 3000:3000

# Restart SSH port-forward
kubectl port-forward svc/gitea-ssh -n gitea 2222:22
```

### Database Operations

```bash
# Access Gitea pod
kubectl exec -it -n gitea gitea-0 -- /bin/sh

# Inside the pod, access SQLite database
sqlite3 /data/gitea.db

# List tables
.tables

# Exit
.quit
exit
```

### Backup and Restore

```bash
# Backup Gitea data
kubectl exec -n gitea gitea-0 -- tar czf /tmp/gitea-backup.tar.gz /data
kubectl cp gitea/gitea-0:/tmp/gitea-backup.tar.gz ./gitea-backup.tar.gz

# Restore Gitea data (be careful!)
kubectl cp ./gitea-backup.tar.gz gitea/gitea-0:/tmp/gitea-backup.tar.gz
kubectl exec -n gitea gitea-0 -- tar xzf /tmp/gitea-backup.tar.gz -C /
kubectl delete pod -n gitea gitea-0  # Restart to apply
```

## Troubleshooting

### Issue 1: Pods Not Starting

**Symptoms:** Pods stuck in `Pending`, `ImagePullBackOff`, or `CrashLoopBackOff` state.

**Solutions:**
```bash
# Check pod status
kubectl get pods -n gitea

# Describe problematic pod
kubectl describe pod -n gitea <pod-name>

# Check logs
kubectl logs -n gitea <pod-name>

# Common fixes:
# - Increase Docker memory (Docker Desktop: Settings → Resources → Memory → 4GB+)
# - Check if Docker is running
# - Delete and recreate the pod:
kubectl delete pod -n gitea <pod-name>
```

### Issue 2: Can't Access UI on localhost:3000

**Symptoms:** Browser shows "Connection refused" or "Unable to connect"

**Solutions:**
1. Verify port-forward is running:
   ```bash
   ps aux | grep port-forward
   ```

2. Check if port 3000 is already in use:
   ```bash
   # Linux/macOS
   lsof -i :3000
   
   # Windows
   netstat -ano | findstr :3000
   ```

3. Try a different port:
   ```bash
   kubectl port-forward svc/gitea-http -n gitea 3030:3000
   # Then access http://localhost:3030
   ```

4. Verify the service exists:
   ```bash
   kubectl get svc -n gitea gitea-http
   ```

### Issue 3: Installation Page Not Appearing

**Symptoms:** Blank page or errors when accessing http://localhost:3000

**Solutions:**
```bash
# Check Gitea logs
kubectl logs -n gitea -l app=gitea

# Check if Gitea is already initialized
kubectl exec -n gitea gitea-0 -- ls -la /data

# If app.ini exists, Gitea is already set up
# To reset, delete the PVC and reinstall:
helm uninstall gitea -n gitea
kubectl delete pvc -n gitea --all
helm install gitea . -n gitea
```

### Issue 4: Can't Push to Repository via HTTPS

**Symptoms:** Authentication errors when pushing via HTTPS.

**Solutions:**
1. Verify your username and password are correct
2. Check if user has write access to the repository
3. Try using a personal access token instead:
   - In Gitea UI: Settings → Applications → Generate New Token
   - Use the token as your password when prompted

### Issue 5: SSH Connection Refused

**Symptoms:** SSH connection fails or times out.

**Solutions:**
1. Verify SSH port-forward is running:
   ```bash
   ps aux | grep "port-forward.*gitea-ssh"
   ```

2. Check if SSH service is running:
   ```bash
   kubectl get svc -n gitea gitea-ssh
   ```

3. Test SSH connection:
   ```bash
   ssh -p 2222 git@localhost -v
   ```

4. Verify SSH key is added in Gitea UI

### Issue 6: Persistent Volume Issues

**Symptoms:** Data loss after pod restart, PVC in `Pending` state.

**Solutions:**
```bash
# Check PVC status
kubectl get pvc -n gitea

# Describe PVC for issues
kubectl describe pvc -n gitea

# For Kind clusters, ensure storage provisioner is available:
kubectl get storageclass

# If using custom storage class, update values.yaml:
gitea:
  persistence:
    storageClass: "your-storage-class"
```

### Issue 7: Out of Resources (Memory/CPU)

**Symptoms:** Pods evicted, Kind cluster unresponsive, or OOMKilled errors.

**Solutions:**
1. Increase Docker resources (Settings → Resources):
   - Memory: 4GB+ recommended
   - CPU: 2+ cores recommended

2. Reduce Gitea resource requests in `values.yaml`:
   ```yaml
   resources:
     requests:
       cpu: 50m
       memory: 64Mi
   ```

3. Restart Kind cluster:
   ```bash
   kind delete cluster --name gitea-cluster
   kind create cluster --name gitea-cluster
   ```

### Issue 8: Migration from Another Git Service

**Symptoms:** Need to migrate repositories from GitHub, GitLab, etc.

**Solutions:**
1. Use Gitea's built-in migration feature:
   - In Gitea UI: Click "+" → "New Migration"
   - Select source (GitHub, GitLab, etc.)
   - Enter repository URL and credentials
   - Click "Migrate Repository"

2. For bulk migrations, use Gitea API or CLI tools

### Issue 9: Email Not Working

**Symptoms:** Users not receiving emails for password reset, notifications, etc.

**Solutions:**
1. Email is disabled by default in this configuration (for local dev)
2. To enable, update `values.yaml`:
   ```yaml
   mailer:
     ENABLED: true
     PROTOCOL: smtp
     SMTP_ADDR: smtp.example.com
     SMTP_PORT: 587
     FROM: gitea@example.com
     USER: your-smtp-user
     PASSWD: your-smtp-password
   ```

3. Upgrade the Helm release:
   ```bash
   helm upgrade gitea . -n gitea
   ```

### Issue 10: Webhook Not Triggering

**Symptoms:** Webhooks to external services not working.

**Solutions:**
1. Check webhook settings in repository settings
2. For localhost webhooks, they won't work from Gitea pod
3. Check Gitea logs for webhook delivery errors:
   ```bash
   kubectl logs -n gitea -l app=gitea | grep webhook
   ```

### Getting More Help

- **Gitea Documentation:** https://docs.gitea.io/
- **Gitea GitHub Issues:** https://github.com/go-gitea/gitea/issues
- **Kind Documentation:** https://kind.sigs.k8s.io/
- **Helm Documentation:** https://helm.sh/docs/

For issues specific to this installation, check:
- `kubectl logs` for pod errors
- `kubectl describe pod` for resource/scheduling issues
- Gitea logs in the UI (Site Administration → Monitoring → Logs)
