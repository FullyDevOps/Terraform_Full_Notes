### **17.1 Kubernetes Provider Configuration**  
*The foundation. Get this wrong, and *nothing* works.*

#### **Core Concepts**  
- **Purpose**: The `kubernetes` provider acts as Terraform's "driver" to interact with your Kubernetes cluster (like a database driver).  
- **Authentication**: Terraform *must* authenticate to the cluster API server. **This is NOT optional.**  

#### **Critical Configuration Methods**  
*(Choose ONE per environment)*  

1. **`config_path` (Most Common - Local/Dev)**  
   ```hcl
   provider "kubernetes" {
     config_path = "~/.kube/config" # Default location
     config_context = "my-cluster-context" # Optional: Specific context
   }
   ```
   - **How it works**: Reads your `~/.kube/config` file (like `kubectl` does).  
   - **When to use**: Local development, CI/CD pipelines with pre-configured kubeconfig.  
   - **⚠️ Critical Security Note**: *Never* commit kubeconfig files to Git! Use `.gitignore` + CI secrets management.

2. **Static Credentials (Ephemeral/CI)**  
   ```hcl
   provider "kubernetes" {
     host                   = "https://api.my-cluster.example.com"
     client_certificate     = file("~/.kube/client.crt")
     client_key             = file("~/.kube/client.key")
     cluster_ca_certificate = file("~/.kube/ca.crt")
   }
   ```
   - **When to use**: CI/CD pipelines where kubeconfig isn't pre-loaded.  
   - **⚠️ Security**: Use Terraform variables + CI secrets (e.g., GitHub Secrets, Vault) to inject these files. *Never hardcode*.

3. **IAM Auth (Cloud Clusters - EKS, GKE, AKS)**  
   ```hcl
   # EKS Example (Requires AWS provider)
   data "aws_eks_cluster" "cluster" {
     name = "my-eks-cluster"
   }

   data "aws_eks_cluster_auth" "auth" {
     name = "my-eks-cluster"
   }

   provider "kubernetes" {
     host                   = data.aws_eks_cluster.cluster.endpoint
     cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority[0].data)

     token = data.aws_eks_cluster_auth.auth.token
     load_config_file = false
   }
   ```
   - **Why**: Uses IAM roles (AWS) / Service Accounts (GCP/Azure) for auth. *No kubeconfig needed*.  
   - **Mandatory for cloud**: Avoids manual kubeconfig updates when cluster certs rotate.

#### **Key Pitfalls**  
- **Context Mismatch**: Terraform uses `current-context` in kubeconfig by default. Explicitly set `config_context` if multiple clusters exist.  
- **Certificate Rotation**: Cloud clusters auto-rotate CA certs. **Provider will break** if you hardcode `cluster_ca_certificate`. Use `base64decode()` + data sources (like EKS example above).  
- **Timeouts**: Add `timeout` block for slow clusters:  
  ```hcl
  provider "kubernetes" {
    # ... auth ...
    timeouts {
      read = "10m"
      create = "15m"
      update = "15m"
      delete = "15m"
    }
  }
  ```

---

### **17.2 Deploying Resources (Pods, Services, Deployments)**  
*Where Terraform shines for *declarative infrastructure* (but with caveats).*

#### **The Golden Rule**  
> **Terraform manages Kubernetes *resources*, NOT applications.**  
> Use it for *infrastructure* (networking, RBAC, storage) and *stable app manifests*. Avoid managing ephemeral app state.

#### **Resource Hierarchy & Best Practices**  
| Resource Type       | Terraform Resource          | When to Use                                                                 | Anti-Pattern Alert!                                                                 |
|---------------------|-----------------------------|-----------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| **Namespace**       | `kubernetes_namespace`      | Isolate environments (dev/staging/prod), enforce quotas                     | Don't manage app namespaces via Terraform if apps self-provision (e.g., via Helm) |
| **Deployment**      | `kubernetes_deployment`     | Stateless apps (web servers, APIs), scaling, rolling updates                | **NEVER** manage `Pods` directly! Always use Deployments/StatefulSets.            |
| **Service**         | `kubernetes_service`        | Expose apps internally (ClusterIP) or externally (LoadBalancer)             | Avoid `NodePort` for production; use Ingress instead.                             |
| **Ingress**         | `kubernetes_ingress`        | HTTP(S) routing (requires Ingress Controller like Nginx, ALB)               | Don't configure TLS certs here; use Cert-Manager (see 17.3).                      |

#### **Critical Deployment Example**  
```hcl
# Deployment (Stateless App)
resource "kubernetes_deployment" "nginx" {
  metadata {
    name      = "nginx"
    namespace = "default" # Better: Use a dedicated namespace!
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "nginx"
      }
    }

    template {
      metadata {
        labels = {
          app = "nginx"
        }
      }

      spec {
        container {
          image = "nginx:1.25"
          name  = "nginx"
          port {
            container_port = 80
          }
          # Liveness/Readiness Probes (MANDATORY for production)
          liveness_probe {
            http_get {
              path = "/"
              port = 80
            }
            initial_delay_seconds = 5
            period_seconds        = 10
          }
        }
      }
    }
  }
}

# Service (Exposes Deployment)
resource "kubernetes_service" "nginx" {
  metadata {
    name      = "nginx"
    namespace = "default"
  }

  spec {
    selector = {
      app = "nginx" # Must match Deployment labels
    }
    port {
      port        = 80
      target_port = 80
    }
    type = "ClusterIP" # For internal access
    # type = "LoadBalancer" # For external access (cloud only)
  }
}
```

#### **Why NOT to Manage Pods Directly**  
- **Pods are ephemeral**: Kubernetes recreates them automatically. Terraform will fight the cluster controller.  
- **No scaling/updates**: Deployments handle rolling updates, replicas, and health checks.  
- **Terraform will fail**: If a Pod crashes, Terraform sees it as "drift" and tries to recreate it (wasting resources).

---

### **17.3 Managing Namespaces, Secrets, ConfigMaps**  
*Where Terraform gets dangerous. Handle with extreme care.*

#### **Namespaces**  
```hcl
resource "kubernetes_namespace" "prod" {
  metadata {
    name = "production"
    labels = {
      environment = "prod"
    }
    annotations = {
      "cost-center" = "finance"
    }
  }
}
```
- **Use Case**: Environment separation, resource quotas (`ResourceQuota`), network policies.  
- **⚠️ Danger Zone**: If apps auto-create namespaces (e.g., Helm `--create-namespace`), Terraform will **delete them** on `terraform destroy`!  

#### **Secrets - The #1 Security Risk**  
```hcl
# WARNING: This stores plaintext secret IN TERRAFORM STATE!
resource "kubernetes_secret" "db_password" {
  metadata {
    name = "db-secret"
  }

  data = {
    password = "SUPER_SECRET" # NEVER DO THIS!
  }
}
```
- **Why it's dangerous**:  
  1. Secret value is in Terraform state (plaintext by default).  
  2. `terraform plan` shows diffs (leaking secrets in logs).  
  3. Accidental `git commit` of `.tf` files = catastrophic breach.  

- **SAFE Approach**:  
  ```hcl
  # Use SSM Parameter Store (AWS) or Vault
  data "aws_ssm_parameter" "db_password" {
    name = "/prod/db/password"
  }

  resource "kubernetes_secret" "db" {
    metadata {
      name = "db-secret"
    }

    data = {
      password = data.aws_ssm_parameter.db_password.value
    }

    # CRITICAL: Prevent Terraform from managing changes to the secret
    lifecycle {
      ignore_changes = [data]
    }
  }
  ```
  - **`ignore_changes` is non-negotiable**: Stops Terraform from overwriting secrets changed *outside* Terraform (e.g., by an app).  
  - **Encrypt state**: Always use remote state (S3 + KMS, HashiCorp Vault) with encryption.  

#### **ConfigMaps - Safer Alternative to Secrets**  
```hcl
resource "kubernetes_config_map" "app_config" {
  metadata {
    name = "app-config"
  }

  data = {
    log_level = "info"
    feature_x = "enabled"
  }
}
```
- **When to use**: Non-sensitive configuration (environment variables, feature flags).  
- **⚠️ Warning**: ConfigMaps are *not* encrypted at rest in etcd. Don't store API keys!  
- **Better Practice**: Use external config servers (Consul, Spring Cloud Config) for dynamic config.

---

### **17.4 Helm Provider: Installing Helm Charts**  
*The bridge between Terraform and Helm (the de facto K8s package manager).*

#### **Why Use Helm with Terraform?**  
- Helm manages complex app dependencies (e.g., Prometheus needs CRDs, RBAC, deployments).  
- Terraform manages infrastructure (VPC, IAM, K8s cluster).  
- **Together**: Terraform creates the cluster → Helm deploys apps *on* the cluster.

#### **Helm Provider Setup**  
```hcl
# Configure Kubernetes provider FIRST (17.1)
provider "helm" {
  kubernetes {
    host                   = "https://api.my-cluster.example.com"
    client_certificate     = file("certs/client.crt")
    client_key             = file("certs/client.key")
    cluster_ca_certificate = file("certs/ca.crt")
  }
}
```

#### **Installing a Chart - Critical Patterns**  
```hcl
# Install NGINX Ingress Controller (from official repo)
resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  version    = "4.8.0" # Pin versions!

  namespace = "ingress-nginx" # Helm creates this namespace

  set {
    name  = "controller.service.type"
    value = "LoadBalancer"
  }

  set {
    name  = "controller.replicaCount"
    value = "2"
  }

  # Prevent Terraform from deleting Helm-managed resources
  depends_on = [kubernetes_namespace.ingress_nginx]
}
```

#### **Must-Know Helm Provider Rules**  
1. **Pin Chart Versions**: `version = "x.y.z"` – Never use `latest`!  
2. **Namespaces**: Helm *can* create namespaces (`create_namespace = true`), but **Terraform should own namespaces** (see 17.3).  
3. **Values Management**:  
   - Use `set` for simple values (as above).  
   - For complex values, use `values = [file("values.yaml")]` (store in version control).  
4. **Drift Handling**: Helm tracks releases in `kube-system` ConfigMap. Terraform *only* manages the Helm release object.  
5. **Destroy Order**: `terraform destroy` uninstalls Helm charts → deletes namespaces. **Reverse order avoids errors**.  

#### **When NOT to Use Helm Provider**  
- Simple apps (use native K8s resources).  
- Apps requiring complex pre/post-install hooks (Helm hooks won't trigger via Terraform).

---

### **17.5 Managing CRDs and Operators**  
*Advanced territory. Proceed with caution.*

#### **CRDs (Custom Resource Definitions)**  
- **What**: Extend Kubernetes API (e.g., `Ingress`, `Certificate` from Cert-Manager).  
- **Terraform Management**:  
  ```hcl
  resource "kubernetes_manifest" "cert_manager_crd" {
    manifest = jsondecode(file("cert-manager.crds.yaml"))
  }
  ```
  - **Why `kubernetes_manifest`?** CRDs are cluster-scoped. The native `kubernetes_crd` resource is deprecated.  
  - **⚠️ Critical**: CRDs **MUST** be applied *before* custom resources (e.g., `Certificate`). Use `depends_on`.  

#### **Operators (e.g., Prometheus Operator, Postgres Operator)**  
- **What**: Controllers that manage complex stateful apps (e.g., `Prometheus`, `PostgresCluster`).  
- **Terraform Strategy**:  
  1. **Install Operator** via Helm (preferred) or YAML:  
     ```hcl
     resource "helm_release" "prometheus_operator" {
       chart = "prometheus-operator"
       # ...
     }
     ```
  2. **Create Custom Resources (CRs)** *after* the operator is ready:  
     ```hcl
     resource "kubernetes_manifest" "prometheus" {
       manifest = jsondecode(file("prometheus.yaml"))
       depends_on = [helm_release.prometheus_operator] # CRITICAL!
     }
     ```
- **The Landmine**:  
  - Operators often mutate CRs (e.g., adding status fields).  
  - Terraform will see this as "drift" and try to revert changes → **operator breaks**.  
  - **Solution**:  
    ```hcl
    resource "kubernetes_manifest" "prometheus" {
      manifest = jsondecode(file("prometheus.yaml"))
      field_manager = "Terraform" # Prevents conflicts with operator's field manager
      lifecycle {
        ignore_changes = [manifest] # ONLY if operator modifies spec (rare)
      }
    }
    ```
    **⚠️ `ignore_changes` is a last resort** – it means Terraform won't update the CR if you change `prometheus.yaml`!

---

### **17.6 Limitations of Terraform for Kubernetes (vs Helm/Kustomize)**  
*The most important section. Know when NOT to use Terraform.*

#### **Core Limitations**  
| Issue                          | Why It Happens                                                                 | Workaround                                  | Better Tool       |
|--------------------------------|--------------------------------------------------------------------------------|---------------------------------------------|-------------------|
| **Drift Detection**            | K8s controllers mutate resources (e.g., `status` fields). Terraform sees drift. | `lifecycle { ignore_changes = [...] }`      | **Helm** (tracks releases) |
| **No Hooks**                   | Terraform lacks pre/post-deploy hooks (e.g., run DB migration before deploy). | Use `local-exec` provisioners (fragile)     | **Helm** (hooks)  |
| **Immutable Fields**           | Changing `type=ClusterIP` → `type=LoadBalancer` requires destroy/recreate.     | Avoid changing immutable fields; use Helm   | **Kustomize** (patch) |
| **Complex App Dependencies**   | Apps need CRDs → RBAC → Deployments in strict order. Terraform parallelizes.   | `depends_on` (error-prone for large apps)   | **Helm** (dependency graph) |
| **Secret Rotation**            | Terraform replaces Secrets on every apply (causing app restarts).              | `ignore_changes` + external secret mgmt     | **External Secrets Operator** |
| **Stateful App Management**    | Terraform doesn't understand K8s stateful workflows (e.g., ordered pod startup). | Don't manage StatefulSets via Terraform     | **Operators**     |

#### **When to Use What**  
| Scenario                                  | Recommended Tool     | Why                                                                 |
|-------------------------------------------|----------------------|---------------------------------------------------------------------|
| **Cluster Infrastructure** (VPC, IAM, Nodes) | **Terraform**        | Terraform's core strength: cloud resources.                         |
| **Base Cluster Addons** (CNI, Ingress, Cert-Manager) | **Helm**             | Helm handles dependencies (CRDs → RBAC → Deployments) safely.       |
| **Application Deployment**                | **Helm** or **Kustomize** | Helm for templating/package management; Kustomize for env overlays. |
| **Secrets Management**                    | **External Secrets Operator** | Pulls secrets from Vault/SSM into K8s Secrets *without* Terraform. |
| **Custom Controllers/Operators**          | **Operator SDK**     | Terraform shouldn't manage operator-created resources.              |

#### **The Hard Truth**  
> **Terraform is for *infrastructure*, not *applications*.**  
> - ✅ **Use Terraform for**: Cluster creation, network setup, RBAC, namespaces, *stable* CRDs.  
> - ❌ **Avoid Terraform for**: Day-2 app management, complex app deployments, secrets rotation, stateful apps.  

#### **Real-World Workflow**  
1. **Terraform**: Provision EKS cluster + VPC + IAM roles.  
2. **Helm**: Deploy base layer (ArgoCD, Cert-Manager, Ingress Controller).  
3. **ArgoCD** (GitOps): Deploy applications from Git (using Helm Charts or Kustomize).  
   → *Terraform never touches application manifests.*

---

### **Key Takeaways for Your Future Reference**  
1. **Provider Auth**: Use cloud IAM (EKS/GKE/AKS) or secure kubeconfig. *Never hardcode secrets*.  
2. **Resources**: Manage Deployments/Services via Terraform, **NOT** Pods.  
3. **Secrets**: *Always* use `lifecycle { ignore_changes }` and external secret stores.  
4. **Helm Provider**: For installing *infrastructure* charts (Ingress, Cert-Manager), **NOT** apps.  
5. **CRDs/Operators**: Apply CRDs first; avoid managing operator-created resources.  
6. **Limitations**: Terraform ≠ Helm. Use Terraform for cluster infra, Helm/Kustomize for apps.  

> **Pro Tip**: Run `terraform state rm kubernetes_secret.db_password` if you ever accidentally commit a secret. Then rotate that secret *immediately*.
