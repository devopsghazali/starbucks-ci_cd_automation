# 🚀 Starbucks CI/CD + GitOps (End-to-End Guide)

This document explains the **complete CI → CD → GitOps deployment flow**
for the **Starbucks React application**, starting from code commit
to a running application inside Kubernetes, using **Jenkins**, **Nexus**, **Argo CD**, and **GitOps** principles.

---

## 🧠 High-Level Architecture

```
Developer
   ↓
GitHub (Application Code)
   ↓
Jenkins CI Pipeline
   ↓
Docker Image
   ↓
Nexus (Private Docker Registry)
   ↓
GitOps Repository (Kubernetes YAML)
   ↓
Argo CD
   ↓
Kubernetes Cluster
   ↓
Service + Ingress
   ↓
Browser (starbucks.com)
```

---

## 🟢 PART 1: Continuous Integration (CI)

### 🔹 Tools Used

- **GitHub**
- **Jenkins**
- **Node.js**
- **Docker**
- **Nexus Repository** (Docker Hosted)
- **Trivy / Dependency-Check** (optional security scans)

---

### 🔹 CI Responsibilities (Very Important)

**CI DOES NOT deploy anything**

CI only:
- Pulls code from GitHub
- Installs dependencies
- Runs lint & tests
- Builds React application
- Builds Docker image
- Pushes Docker image to Nexus

---

### 🔹 Jenkins Pipeline Responsibilities

✔ Checkout code  
✔ `npm install`  
✔ ESLint (warnings allowed during CI)  
✔ `npm run build`  
✔ Docker image build  
✔ Docker image push to Nexus  

**Note:**  
In CI environment, React treats lint warnings as errors.  
Therefore, build is executed with:

```bash
CI=false npm run build
```

---

### 🔹 Docker Image Naming Convention

```text
<NEXUS_IP>:5000/devops/starbucks-react:<BUILD_NUMBER>
```

**Explanation:**

- `<NEXUS_IP>:5000` → Nexus Docker registry (5000 = Docker connector)
- `devops` → Docker hosted repository name
- `starbucks-react` → application name
- `<BUILD_NUMBER>` → unique Jenkins build number

---

### 🔹 CI Output

After Jenkins completes successfully:

```text
<NEXUS_IP>:5000/devops/starbucks-react:9
```

✔ Image exists in Nexus  
✔ Ready for deployment (but not deployed yet)

---

## 🟢 PART 2: GitOps Repository

### 🔹 Purpose of GitOps Repo

The GitOps repository defines:

**"What should be running inside Kubernetes"**

This repo:
- Contains only Kubernetes YAML
- Has no application code
- Is the single source of truth

---

### 🔹 GitOps Repo Used
```
k8s-for-starbucks
```

---

### 🔹 Repository Structure
```
k8s-for-starbucks/
├── namespace.yaml
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

All YAML files are stored in the root directory.

---

## 🟢 PART 3: Kubernetes Core Resources

### 🔹 Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

**Purpose:**
- Environment isolation
- Logical separation (dev environment)

---

### 🔹 Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: starbucks-react
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: starbucks-react
  template:
    metadata:
      labels:
        app: starbucks-react
    spec:
      imagePullSecrets:
        - name: nexus-regcred
      containers:
        - name: starbucks-react
          image: <NEXUS_IP>:5000/devops/starbucks-react:9
          ports:
            - containerPort: 3000
```

**Key Concepts Explained:**
- `replicas`: desired number of pods
- `labels ↔ selector`: binds Deployment to Pods
- `image`: pulled from private Nexus registry
- `imagePullSecrets`: authentication for Nexus
- `containerPort: 3000`: React app runs here

---

### 🔹 Service (Internal Networking)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: starbucks-svc
  namespace: dev
spec:
  type: ClusterIP
  selector:
    app: starbucks-react
  ports:
    - port: 80
      targetPort: 3000
```

**Purpose:**
- Stable internal endpoint
- Load-balances traffic to pods
- Decouples pod IPs from access

**Traffic Flow:**
```
Service:80 → Pod:3000
```

---

## 🟢 PART 4: Private Registry Access (Nexus)

### 🔹 Kubernetes Docker Registry Secret
```bash
kubectl create secret docker-registry nexus-regcred \
  --docker-server=<NEXUS_IP>:5000 \
  --docker-username=<USERNAME> \
  --docker-password=<PASSWORD> \
  --docker-email=<EMAIL> \
  -n dev
```

**Why required:**
- Nexus is a private registry
- Kubernetes must authenticate before pulling images

**Important Rules:**
- Secret type: `docker-registry`
- Secret namespace must match pod namespace
- Email is a legacy field (no email is sent)

---

## 🟢 PART 5: Argo CD (GitOps Engine)

### 🔹 What Argo CD Does

Argo CD:
- Watches GitOps repo
- Compares Git vs cluster state
- Automatically applies changes
- Self-heals configuration drift

**Humans do not deploy — Git does**

---

### 🔹 Argo CD Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: starbucks-dev
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/<YOUR_ORG>/k8s-for-starbucks.git
    targetRevision: main
    path: .

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Explanation:**
- `repoURL`: GitOps repository
- `path: .`: YAML files are in repo root
- `automated`: no manual sync required
- `prune`: deletes removed resources
- `selfHeal`: fixes manual cluster changes

---

## 🟢 PART 6: Ingress (External Access)

### 🔹 Ingress Configuration
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: starbucks-ingress
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
  - host: starbucks.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: starbucks-svc
            port:
              number: 80
```

---

### 🔹 Ingress Traffic Flow
```
Browser
  ↓ starbucks.com
Ingress (NGINX)
  ↓
Service (starbucks-svc)
  ↓
Pod (React on port 3000)
```

---

### 🔹 Local Domain Mapping
```bash
# /etc/hosts
<INGRESS_IP> starbucks.com
```

---

## 🟢 PART 7: End-to-End CI → GitOps → CD Flow

### 🔁 Complete Flow

1. Developer pushes code
2. Jenkins CI pipeline runs
3. Docker image is built
4. Image is pushed to Nexus
5. GitOps repo image tag is updated
6. Argo CD detects Git change
7. Kubernetes updates automatically
8. Application is accessible via Ingress

---

### 🔑 Golden Principles

- **Jenkins** builds artifacts
- **Git** defines desired state
- **Argo CD** enforces reality
- **Kubernetes** runs workloads

---

## ✅ Final Result

✔ Fully automated CI  
✔ Fully automated CD  
✔ No `kubectl apply`  
✔ No manual deployment  
✔ Rollback via Git  

---

## 📝 Important Notes

### Security Best Practices
- Replace all placeholders (`<NEXUS_IP>`, `<USERNAME>`, `<PASSWORD>`, `<EMAIL>`, `<YOUR_ORG>`, `<INGRESS_IP>`) with actual values
- Never commit credentials to Git
- Use Kubernetes Secrets for sensitive data
- Consider using external secret management (HashiCorp Vault, AWS Secrets Manager)

### CI/CD Best Practices
- Use `CI=false npm run build` to handle React ESLint warnings in CI environment
- Keep GitOps repo separate from application code
- Use semantic versioning or build numbers for Docker image tags
- Enable Argo CD auto-sync for continuous deployment
- Implement proper monitoring and alerting

### Troubleshooting Tips
- Verify Nexus Docker registry is accessible from Kubernetes cluster
- Ensure `imagePullSecrets` exists in the correct namespace
- Check Argo CD sync status for deployment issues
- Verify NGINX Ingress Controller is installed and running
- Test domain resolution before accessing via browser

---

## 🎯 Summary

This setup provides:
- **Zero-touch deployment**: Code push → automatic production deployment
- **Git as single source of truth**: All infrastructure defined in Git
- **Self-healing**: Argo CD automatically corrects any manual changes
- **Rollback capability**: Simply revert Git commit to rollback
- **Audit trail**: All changes tracked in Git history



