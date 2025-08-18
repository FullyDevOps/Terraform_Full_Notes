### 2.1 What is Terraform?
**Core Definition:**  
Terraform is an **open-source Infrastructure as Code (IaC) tool** created by HashiCorp. It enables you to **define, provision, and manage cloud and on-premises infrastructure** using human-readable configuration files (written in HashiCorp Configuration Language - HCL or JSON).  

**Key Principles Explained:**  
1.  **Declarative Syntax:**  
    *   You describe the *desired end-state* of your infrastructure (e.g., "I need 3 AWS EC2 instances with 8GB RAM"), **not** the step-by-step commands to build it.  
    *   *Why it matters:* Terraform calculates the necessary steps *itself*, making configurations portable and reducing human error. You focus on *what*, not *how*.  

2.  **Idempotency:**  
    *   Running `terraform apply` multiple times with the same config produces the *same result* without duplicate resources or errors (if the infrastructure is already in the desired state).  
    *   *Why it matters:* Enables safe, repeatable deployments. Critical for CI/CD pipelines and team collaboration.  

3.  **Provider-Based Architecture:**  
    *   Terraform **itself** doesn't interact directly with AWS, Azure, GCP, etc. It uses **providers** (plugins) that translate Terraform configurations into API calls for specific platforms/services (e.g., `azurerm` for Azure, `aws` for AWS).  
    *   *Why it matters:* Makes Terraform **cloud-agnostic**. Learn Terraform once, deploy anywhere. Over 3,000+ community providers exist.  

4.  **State Management:**  
    *   Terraform maintains a **state file** (`terraform.tfstate`) that maps real-world resources to your configuration. It tracks metadata (IPs, IDs, dependencies).  
    *   *Why it matters:* State is the **single source of truth**. Without it, Terraform can't manage existing resources or detect drift. *Critical for collaboration (use remote state!)*.  

5.  **Resource Graph:**  
    *   Terraform builds a **dependency graph** of all resources. It automatically determines the correct order for creation/destroy (e.g., create a Virtual Network *before* a VM inside it).  
    *   *Why it matters:* Eliminates manual sequencing errors. Enables parallel operations where possible for speed.  

**Terraform vs. Alternatives (Key Differentiators):**  
*   **vs. Ansible/Chef/Puppet:** These are *configuration management* tools (manage *inside* servers). Terraform is *provisioning* (build the servers/networks *first*). Often used *together*.  
*   **vs. CloudFormation/ARM Templates:** Terraform is multi-cloud; native tools are single-cloud. Terraform uses declarative HCL (more readable than JSON/YAML); native tools often use verbose JSON/YAML. Terraform state enables better drift detection.  

**Why Terraform? (The Big Picture):**  
*   **Version Control Infrastructure:** Store configs in Git – track changes, roll back, audit.  
*   **Eliminate "Works on My Machine":** Consistent environments (Dev, Staging, Prod).  
*   **Self-Documenting:** Config files *are* the documentation.  
*   **Cost Optimization:** Easily spin up/down non-prod environments.  
*   **Disaster Recovery:** Rebuild entire infrastructure from config + state.  

---

### 2.2 Terraform Architecture Overview
Terraform's architecture is modular and plugin-driven. Here's the complete breakdown:

![Terraform Architecture Diagram](https://miro.medium.com/v2/resize:fit:4800/format:webp/0*bJzMGdZBo0zKfbvQ)  
*(Simplified visualization - core components interact as described below)*

1.  **Terraform CLI (Command Line Interface):**  
    *   The user-facing executable (`terraform` command).  
    *   **Responsibilities:** Parses configs, manages state, invokes Terraform Core, communicates with providers, outputs results.  
    *   *Key Detail:* It's a **thin orchestrator**. Most logic lives in Core/Providers.

2.  **Terraform Core:**  
    *   The **engine** written in Go. The "brain" of Terraform.  
    *   **Critical Responsibilities:**  
        *   **Reading & Parsing Configs:** Processes `.tf` files (HCL) into an in-memory representation.  
        *   **State Management:** Reads/writes the state file (`terraform.tfstate`).  
        *   **Dependency Graph Construction:** Builds the resource dependency graph.  
        *   **Execution Plan Generation:** Calculates the difference between config and state to create the "plan".  
        *   **Plan Execution:** Executes the plan by calling provider APIs.  
        *   **Provider/plugin Interface:** Manages the lifecycle of provider plugins (download, execute, communicate via RPC).  

3.  **Providers:**  
    *   **Plugins** (separate binaries) that interact with *specific* APIs (cloud, SaaS, on-prem).  
    *   **Responsibilities:**  
        *   Implement CRUD (Create, Read, Update, Delete) operations for resources defined in the config (e.g., `azurerm_virtual_machine`).  
        *   Translate Terraform's generic resource actions into vendor-specific API calls.  
        *   Validate resource configuration arguments.  
    *   *Key Details:*  
        *   Providers are **not** part of Terraform Core. They are downloaded automatically by `terraform init` into `.terraform/providers`.  
        *   Providers define **Resources** (things you create/manage, e.g., `azurerm_resource_group`) and **Data Sources** (things you read/lookup, e.g., `azurerm_resource_group` to get an existing group's ID).  
        *   Providers have their own versioning, independent of Terraform Core. Pin provider versions in configs! (`required_providers` block).

4.  **State (`terraform.tfstate`):**  
    *   **JSON file** (usually) storing the **current state** of your managed infrastructure.  
    *   **Critical Contents:**  
        *   Mapping of resource instances in config to real-world IDs (e.g., `azurerm_virtual_machine.vm["prod"]` -> `Azure VM ID: /subscriptions/.../Microsoft.Compute/virtualMachines/prod-vm`).  
        *   Resource attributes (e.g., `vm.public_ip`, `vm.private_ip`).  
        *   Dependency relationships.  
        *   Provider configuration used.  
    *   **Why State is Non-Negotiable:**  
        *   Terraform **cannot** manage resources without state. It wouldn't know what exists or what to update.  
        *   **Local vs. Remote State:**  
            *   *Local:* `terraform.tfstate` file (default). **Dangerous for teams!** (Concurrency issues, accidental commits to Git).  
            *   *Remote:* **MANDATORY for production** (e.g., Azure Storage Account, AWS S3, Terraform Cloud). Enables:  
                *   **State Locking:** Prevents concurrent `apply`/`destroy` causing corruption.  
                *   **Centralized Access:** Team members share the same state.  
                *   **Auditability:** State history/versioning.  
        *   **NEVER edit state manually!** Use `terraform state` commands or `terraform import`.

**How It All Fits Together (The Flow):**  
1.  You write config (`.tf` files).  
2.  Run `terraform init` -> Core downloads Providers.  
3.  Run `terraform plan` -> Core:  
    *   Reads config + current state (local/remote).  
    *   Asks Providers to *read* current real-world state (via API).  
    *   Compares desired state (config) vs. real state (from Providers) vs. last known state (state file).  
    *   Generates an execution plan (what to add/change/destroy).  
4.  Run `terraform apply` -> Core executes the plan:  
    *   For each resource action (create/update/delete), Core instructs the relevant Provider.  
    *   Provider makes API calls to the target platform (Azure, AWS, etc.).  
    *   On success, Core updates the state file with new resource IDs/attributes.  

---

### 2.3 Terraform Workflow (Write → Plan → Apply → Destroy)
This is the **golden cycle** of Terraform. Skipping steps causes disasters.

1.  **Write (Author Configuration):**  
    *   **What:** Create/modify `.tf` files (HCL) defining your infrastructure.  
    *   **Key Files:**  
        *   `main.tf`: Primary resource definitions.  
        *   `variables.tf`: Input variables (e.g., `vm_size = "Standard_B2s"`).  
        *   `outputs.tf`: Values to expose after apply (e.g., `public_ip = azurerm_public_ip.vm.ip_address`).  
        *   `providers.tf` (Optional): Explicit provider configuration/versioning.  
        *   `terraform.tfvars` (Optional): Default values for variables.  
    *   **Best Practices:**  
        *   Use modules for reusability (even simple ones).  
        *   Pin provider & module versions (`required_providers`, `source` with version).  
        *   Use `description` in variables.  
        *   **`terraform fmt`** - Run this *before* committing! Auto-formats HCL.  

2.  **Plan (Preview Changes):**  
    *   **Command:** `terraform plan`  
    *   **What Happens:**  
        1.  Core refreshes state (reads current real-world state via Providers).  
        2.  Compares desired state (your config) against:  
            *   The refreshed real-world state.  
            *   The last recorded state (in state file).  
        3.  Calculates the **execution plan** - the exact set of actions needed.  
    *   **Output Interpretation (CRITICAL):**  
        *   `+` = Resource to be **created**  
        *   `~` = Resource to be **updated in-place**  
        *   `-` = Resource to be **destroyed**  
        *   `+/-` = Resource to be **replaced** (destroyed first, then created - e.g., changing a VM size that requires rebuild).  
        *   `(forces new resource)` - Explicit note why replacement is needed.  
        *   **Red Text:** Destruction! Scrutinize this.  
        *   **Blue Text:** Updates. Check if intended.  
    *   **Why PLAN is MANDATORY:**  
        *   **Safety Net:** See *exactly* what will happen *before* it happens. Prevents accidental deletion of production DBs!  
        *   **Drift Detection:** Shows if real-world state drifted from Terraform's state (e.g., someone manually changed a security group).  
        *   **Review:** Essential for PRs/code reviews. Never `apply` without reviewing `plan`.  
        *   **`-out=tfplan` Flag:** Save the plan to a file for later `apply` (ensures *exactly* what was planned gets applied, no config drift in between).  

3.  **Apply (Execute Changes):**  
    *   **Command:** `terraform apply` (Prompts for confirmation) OR `terraform apply -auto-approve` (Use cautiously!)  
    *   **What Happens:**  
        1.  (Optional) If `-out=tfplan` was used with `plan`, applies *that specific plan*.  
        2.  Otherwise, runs `plan` implicitly first (unless `-skip-refresh` is used - **rarely safe**).  
        3.  **On Confirmation:** Core executes the plan step-by-step:  
            *   For each action, calls the relevant Provider's API.  
            *   Waits for resource creation/update/deletion to complete (respects dependencies).  
            *   **Immediately updates the state file** after *each* resource operation succeeds.  
        4.  Outputs declared `output` values.  
    *   **Critical Notes:**  
        *   **State is Updated Incrementally:** If `apply` fails midway, state is partially updated. You *must* run `apply` again to resolve.  
        *   **Concurrency:** Remote state backends lock state during `apply`/`destroy`. Never run concurrent operations!  
        *   **Rollbacks:** Terraform has **no built-in rollback**. If `apply` fails:  
            *   Fix the config.  
            *   Run `apply` again (it will retry the failed step + subsequent steps).  
            *   *Prevention is key:* Rigorous `plan` review, small incremental changes, robust testing.  

4.  **Destroy (Tear Down Infrastructure):**  
    *   **Command:** `terraform destroy` (Prompts for confirmation)  
    *   **What Happens:**  
        1.  Generates a `plan` showing **ALL** resources managed by this config will be destroyed (`-`).  
        2.  **On Confirmation:** Executes deletion in **reverse dependency order** (e.g., deletes VMs *before* the Virtual Network they reside in).  
        3.  **Empties the state file** (all resources removed from state).  
    *   **Why DESTROY is Part of the Workflow:**  
        *   **Cost Control:** Easily clean up non-production environments (dev/staging).  
        *   **Testing:** Validate your config can be fully recreated.  
        *   **Project End:** Decommission resources safely.  
    *   **WARNING:**  
        *   `destroy` is **irreversible** (unless your cloud provider has backups/snapshots).  
        *   **ALWAYS run `terraform plan -destroy` first** to confirm *exactly* what will be deleted.  
        *   Use `prevent_destroy` lifecycle rule for critical resources (e.g., prod databases) as a safety net.  

**Workflow Summary Diagram:**  
```
[Write Config] --> [terraform init] --> [terraform plan] --> (Review Plan!) --> [terraform apply] --> [terraform destroy (When Needed)]
          ^                                                                                         |
          |_________________________________________________[terraform state]_________________________|
```
*   **`init` is prerequisite** for `plan`/`apply`/`destroy` (sets up providers/modules).  
*   **`plan` is the critical safety gate** before `apply`.  
*   **`destroy` is the controlled teardown.**  
*   **`state` is the persistent record** enabling all operations.  

---

### 2.4 Installing Terraform (CLI)
**Official Source:** [https://developer.hashicorp.com/terraform/downloads](https://developer.hashicorp.com/terraform/downloads)  
**Golden Rule:** **Always install the specific version required by your project** (check `required_version` in config). Don't assume latest is best!

#### Per Operating System:
**A. Windows:**  
1.  **Recommended (Scoop/Brew):**  
    *   *Scoop (Preferred):*  
        ```powershell
        scoop bucket add main
        scoop install terraform
        ```  
    *   *Chocolatey:* `choco install terraform`  
2.  **Manual:**  
    *   Download ZIP for Windows (64-bit usually) from HashiCorp site.  
    *   Extract `terraform.exe`.  
    *   Place it in a directory in your `PATH` (e.g., `C:\Windows\System32` or `C:\terraform` then add to PATH via System Properties).  
    *   **Verify:** Open **New** PowerShell/CMD, run `terraform -help`.  

**B. macOS:**  
1.  **Recommended (Homebrew):**  
    ```bash
    brew tap hashicorp/tap
    brew install hashicorp/tap/terraform
    brew update && brew upgrade hashicorp/tap/terraform # To upgrade later
    ```  
2.  **Manual:**  
    *   Download ZIP for macOS (Intel/Apple Silicon).  
    *   Unzip: `unzip terraform_*.zip` (creates `terraform` binary).  
    *   Move to PATH: `sudo mv terraform /usr/local/bin/`  
    *   *(If SIP blocks execution):* `xattr -d com.apple.quarantine /usr/local/bin/terraform`  

**C. Linux (Debian/Ubuntu):**  
1.  **Official Repository (Recommended):**  
    ```bash
    # Install prerequisites
    sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
    # Add HashiCorp GPG key
    curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
    # Verify fingerprint (MUST match)
    echo "7473-36F9-CF89-02B7-7FCD-  8FF7-59D7-346E-D736-7D6E" | tr -d '[:space:]' | grep -q "$(gpg --show-keys --fingerprint --show-keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg)"
    # Add repository
    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
    # Install
    sudo apt update && sudo apt install terraform
    ```  
2.  **Manual (Generic Linux):**  
    ```bash
    wget https://releases.hashicorp.com/terraform/<VERSION>/terraform_<VERSION>_linux_amd64.zip
    unzip terraform_<VERSION>_linux_amd64.zip
    sudo mv terraform /usr/local/bin/
    ```

**D. Linux (RHEL/CentOS/Fedora):**  
```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install terraform
```

**Critical Post-Install Steps:**  
1.  **Verify Installation (See 2.5)** - **DO NOT SKIP.**  
2.  **Configure PATH:** Ensure the directory containing `terraform` is in your system `PATH` environment variable.  
3.  **Permissions:** The `terraform` binary must be executable (`chmod +x /usr/local/bin/terraform` on Linux/macOS).  
4.  **Version Pinning (For Teams):** Use tools like `tfenv` (Terraform Version Manager) to enforce consistent versions across team members.  

---

### 2.5 Verifying Installation & Terraform Version
**This is non-optional. Always verify after install or version change.**

1.  **Basic Verification (Is it in PATH?):**  
    ```bash
    terraform -help
    ```  
    *   **Expected Output:** A list of available Terraform commands (`init`, `plan`, `apply`, etc.).  
    *   **Failure:** `command not found: terraform` -> PATH is misconfigured. Fix PATH.  

2.  **Verify Version & Check for Updates:**  
    ```bash
    terraform version
    ```  
    *   **Expected Output:**  
        ```bash
        Terraform v1.7.5
        on linux_amd64
        ```  
        *   Shows the **exact installed version** (v1.7.5).  
        *   Shows the **platform** (os_cpu - linux_amd64).  
    *   **Why Version Matters:**  
        *   Config syntax/features change between versions (e.g., `for_each` vs `count`).  
        *   Provider compatibility varies.  
        *   **Always match the version specified in your project's `terraform { required_version = ">= 1.6.0, < 2.0.0" }` block!**  
    *   **Check for Updates:**  
        *   `terraform version -json` (Machine-readable JSON output - useful for scripts).  
        *   `terraform version` output often includes:  
            ```bash
            Your version of Terraform is out of date! The latest version
            is 1.8.0. You can update by downloading from https://www.terraform.io/downloads.html
            ```  
        *   **Do NOT blindly upgrade!** Check project requirements first.  

3.  **Verify Provider Installation (After `terraform init`):**  
    ```bash
    terraform providers
    ```  
    *   **Expected Output:** Lists all providers required by the config and their installed versions.  
        ```bash
        Providers required by configuration:
          .
          ├── provider[registry.terraform.io/hashicorp/azurerm] >= 3.0.0
          └── provider[registry.terraform.io/hashicorp/random] >= 3.0.0
        ```  
    *   **Critical:** Confirms `init` downloaded the correct providers.  

**Troubleshooting Verification Failures:**  
*   `command not found`: PATH issue. Reinstall or fix PATH.  
*   `permission denied`: Missing execute bit (`chmod +x /path/to/terraform`).  
*   Wrong version: Uninstall and install the correct version. Use `tfenv` for management.  
*   `terraform: no such file or directory`: Wrong architecture (e.g., downloaded macOS binary on Linux). Redownload correct ZIP.  

---

### 2.6 Basic Terraform Commands (init, validate, plan, apply, destroy)
**Essential Command Reference - Master These!**

| Command          | Syntax                     | Purpose & Deep Explanation                                                                                                                               | Critical Flags & Tips                                                                                                                                 |
| :--------------- | :------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`init`**       | `terraform init`           | **Bootstraps Terraform configuration.** **MUST be run FIRST** in a new config directory or after adding providers/modules.<br>**What it does:**<br>- Initializes working directory.<br>- Downloads required **providers** (to `.terraform/providers`).<br>- Downloads **modules** (to `.terraform/modules`).<br>- Configures **backend** (remote state storage).<br>- Sets up local state file (if using local backend). | `-upgrade`: Re-download providers/modules even if already present (use if versions changed).<br>`-backend-config="key=value"`: Pass config to backend (e.g., `-backend-config="storage_account_name=mystorage"` for AzureRM).<br>**Run this every time you clone a repo or change providers/modules!** |
| **`validate`**   | `terraform validate`       | **Checks syntax and basic configuration correctness *without* accessing cloud APIs or state.**<br>**What it does:**<br>- Parses all `.tf` files.<br>- Checks for HCL syntax errors.<br>- Validates resource argument types/values (e.g., is `vm_size` a valid string?).<br>- **Does NOT check:**<br>  - Real cloud permissions.<br>  - Resource existence.<br>  - State consistency.<br>  - Provider-specific constraints (e.g., "this VM size isn't available in this region"). | **Run this *before* `plan`** as a quick sanity check.<br>Fast feedback on config errors.<br>Essential in CI/CD pipelines *before* `plan`.                                                                      |
| **`plan`**       | `terraform plan`           | **Generates and shows the execution plan (what will happen on `apply`).** **THE MOST IMPORTANT COMMAND.**<br>**What it does:**<br>- **Refreshes state:** Queries cloud APIs (via Providers) to get *current real-world state*.<br>- **Compares:** Desired state (config) vs. Real state (from API) vs. Last known state (state file).<br>- **Calculates:** Precise set of create/update/destroy actions needed.<br>- **Outputs:** Color-coded plan showing `+`, `~`, `-`, `+/-`. | `-out=tfplan`: Save plan to file for later `apply` (ensures *exactly* this plan runs).<br>`-var="name=value"`: Pass input variables.<br>`-refresh=false`: **DANGEROUS** - skips state refresh (uses last state). Only use if you *know* state is current.<br>`-destroy`: Shows what `destroy` would delete (use before actual destroy!). |
| **`apply`**      | `terraform apply`          | **Executes the actions proposed in a Terraform plan.**<br>**What it does:**<br>- If no `-out` file: Runs `plan` implicitly first.<br>- **PROMPTS for confirmation** (unless `-auto-approve`).<br>- Executes resource operations **in dependency order**.<br>- **Updates state file IMMEDIATELY** after *each* resource operation succeeds.<br>- Outputs declared `output` values. | `-auto-approve`: Skips confirmation prompt (**Use ONLY in CI/CD with extreme caution!**).<br>`-out=tfplan`: Apply a previously saved plan file.<br>`-var="name=value"`: Pass input variables.<br>**ALWAYS review `plan` output before confirming `apply`!** |
| **`destroy`**    | `terraform destroy`        | **Tears down *all* infrastructure managed by the current configuration.**<br>**What it does:**<br>- Generates a `plan` showing **ALL** resources to be destroyed (`-`).<br>- **PROMPTS for confirmation**.<br>- Executes deletions in **REVERSE dependency order**.<br>- **Empties the state file** (all resources removed). | **RUN `terraform plan -destroy` FIRST!**<br>`-auto-approve`: Skips confirmation (**Extreme caution!**).<br>`-var="name=value"`: Pass variables.<br>**Never run on prod without multiple confirmations!** |

**Command Workflow Order is SACRED:**  
`init` → (`validate`) → `plan` → (`apply` OR `destroy`)  
*   Skipping `init` = Missing providers/modules → Fail.  
*   Skipping `plan` = Blind `apply` → Production outages (common cause!).  
*   Running `apply` before `init` = Fail.  

**State-Related Commands (Crucial for Recovery):**  
*   `terraform state list`: Show all resources in state.  
*   `terraform state show <resource>`: Show detailed state of one resource.  
*   `terraform import <resource> <id>`: Import existing real-world resource into Terraform state (e.g., `terraform import azurerm_resource_group.rg /subscriptions/.../resourceGroups/my-rg`). **MUST match config!**  
*   `terraform state mv <old> <new>`: Rename/move resource in state (e.g., after refactoring config).  

---

### 2.7 Writing Your First Terraform Configuration (Hello World with Azure VM)
**Goal:** Create a minimal, functional Azure VM configuration. Demonstrates core concepts.  
**Prerequisites:**  
1.  Azure Subscription (Free Trial works).  
2.  Azure CLI installed & logged in (`az login`).  
3.  Terraform installed & verified (2.4, 2.5).  

#### Step 1: Directory Setup & Auth
```bash
mkdir terraform-azure-hello && cd terraform-azure-hello
# Create credentials file (SAFER than env vars for local dev)
az ad sp create-for-rbac --name "terraform-hello" --role="Contributor" --scopes="/subscriptions/YOUR_SUB_ID" --sdk-auth > sp.json
```
*   **Why Service Principal?** Terraform needs Azure credentials. Using a dedicated SP with limited scope (`Contributor` on sub) is best practice.  
*   **SAFETY:** `sp.json` contains secrets! **ADD TO `.gitignore` IMMEDIATELY!** Never commit it.  

#### Step 2: Write Configuration Files
**File: `providers.tf`** (Explicit provider config & version pinning)
```hcl
terraform {
  required_version = ">= 1.6.0" # Pin Terraform version
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85.0" # Pin Azure Provider version (CHECK LATEST!)
    }
  }
}

# Configure Azure Provider
provider "azurerm" {
  features {} # Enable all AzureRM features (simplifies first config)
  # Credentials loaded from sp.json (see below)
}
```
*   **Why Pin Versions?** Prevents breaking changes from new Terraform/Provider releases. Critical for stability.  
*   `features {}`: AzureRM requires this block (even empty) since v3.0.  

**File: `variables.tf`** (Define inputs)
```hcl
variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "eastus" # Change to your preferred region
}

variable "vm_name" {
  description = "Name of the VM"
  type        = string
  default     = "hello-vm"
}
```
*   **Why Variables?** Makes config reusable. Change `vm_name` without editing core logic.  

**File: `main.tf`** (The "Hello World" Infrastructure)
```hcl
# 1. Resource Group (Top-level container)
resource "azurerm_resource_group" "rg" {
  name     = "${var.vm_name}-rg" # Uses variable
  location = var.location
}

# 2. Virtual Network
resource "azurerm_virtual_network" "vnet" {
  name                = "${var.vm_name}-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name # Reference RG by name (dependency!)
}

# 3. Subnet
resource "azurerm_subnet" "subnet" {
  name                 = "${var.vm_name}-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name # Depends on VNet
  address_prefixes     = ["10.0.1.0/24"]
}

# 4. Public IP (For SSH access)
resource "azurerm_public_ip" "pip" {
  name                = "${var.vm_name}-pip"
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  allocation_method   = "Dynamic"
  sku                 = "Basic"
}

# 5. Network Security Group (Firewall)
resource "azurerm_network_security_group" "nsg" {
  name                = "${var.vm_name}-nsg"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name

  # Allow SSH (Port 22)
  security_rule {
    name                       = "AllowSSH"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*" # WARNING: Open to internet! For demo ONLY.
    destination_address_prefix = "*"
  }
}

# 6. Network Interface (NIC)
resource "azurerm_network_interface" "nic" {
  name                = "${var.vm_name}-nic"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id # Reference Subnet ID (dependency!)
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id # Attach Public IP
  }

  # Associate NSG with NIC
  network_security_group_id = azurerm_network_security_group.nsg.id
}

# 7. The Virtual Machine (Hello World!)
resource "azurerm_linux_virtual_machine" "vm" {
  name                = var.vm_name
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  size                = "Standard_B1s" # Cheap dev/test size
  network_interface_ids = [
    azurerm_network_interface.nic.id # Critical dependency!
  ]
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }
  admin_username = "azureuser"
  # NEVER hardcode passwords! Use SSH key for real use.
  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub") # Uses your local public key
  }
  disable_password_authentication = true
}
```
*   **Why This Structure?** Shows core Azure resources and dependencies (RG → VNet → Subnet → NIC → VM).  
*   **Critical Dependencies:** Notice how resources reference each other's `.id` or `.name` (e.g., `subnet_id = azurerm_subnet.subnet.id`). Terraform uses this to build the dependency graph.  
*   **Security Note:** `source_address_prefix = "*"` for SSH is **insecure** (open to internet). For real use, restrict to your IP (`source_address_prefix = "YOUR.IP.ADD.RESS"`).  

**File: `outputs.tf`** (Expose useful values)
```hcl
output "public_ip" {
  description = "Public IP address of the VM"
  value       = azurerm_public_ip.pip.ip_address
}
```
*   **Why Outputs?** After `apply`, Terraform prints these. Makes it easy to get key info (like the IP to SSH to).  

**File: `.gitignore`** (ESSENTIAL)
```
*.tfstate
*.tfstate.backup
.terraform/
sp.json
```
*   **Why?** Prevents committing state files (secrets, resource IDs) and provider binaries. `sp.json` contains Azure credentials!  

#### Step 3: Execute the Workflow
1.  **Initialize (Get Providers):**  
    ```bash
    terraform init
    ```  
    *   *Expected:* Downloads `azurerm` provider (~30MB). Success message.  

2.  **Validate (Syntax Check):**  
    ```bash
    terraform validate
    ```  
    *   *Expected:* `Success! The configuration is valid.`  

3.  **Plan (Preview):**  
    ```bash
    terraform plan -var="vm_name=myfirstvm" # Optional: Override default name
    ```  
    *   **AUTHENTICATION:** Terraform finds `sp.json` automatically (by default).  
        *   *How?* AzureRM provider checks `AZURE_SERVICE_PRINCIPAL` env vars OR `sp.json` in current dir.  
    *   *Expected:* Long output showing **+7 resources** to add (RG, VNet, Subnet, PIP, NSG, NIC, VM). **Scrutinize this!**  

4.  **Apply (Deploy):**  
    ```bash
    terraform apply -var="vm_name=myfirstvm"
    ```  
    *   Type `yes` when prompted.  
    *   *Expected:* Slow output as Azure creates resources (5-10 mins). Ends with `Apply complete!` and your `public_ip` output.  
    *   **Verify in Azure Portal:** Go to Portal → Resource Groups → Find `myfirstvm-rg`. See all resources.  

5.  **Destroy (Clean Up - CRITICAL STEP!):**  
    ```bash
    terraform destroy -var="vm_name=myfirstvm"
    ```  
    *   **RUN PLAN FIRST (Optional but good habit):** `terraform plan -destroy`  
    *   Type `yes` when prompted.  
    *   *Expected:* Resources deleted in reverse order. State file emptied. **Costs stop accruing!**  

#### Key Lessons from "Hello World"
1.  **Declarative Power:** You defined the *end state* (VM + network), Terraform handled the complex API sequence.  
2.  **Dependency Magic:** References like `azurerm_subnet.subnet.id` auto-created the correct order (VNet before Subnet, Subnet before NIC, NIC before VM).  
3.  **State is Everything:** `terraform destroy` worked because state tracked all 7 resources.  
4.  **Plan is Safety:** Seeing `+7 to add` before `apply` prevented accidental overwrites.  
5.  **Variables Enable Reuse:** Changing `vm_name` deploys a *new*, isolated environment.  
6.  **Destroy is Essential:** Avoids "cloud bill shock" from forgotten test resources.  

**Next Steps Beyond Hello World:**  
*   Replace hardcoded SSH key path with a variable.  
*   Add a user-data script to install a web server (real "Hello World" page).  
*   Move to **Remote State** (Azure Storage Account).  
*   Create a **Module** for the VM pattern.  
*   Implement **Input Validation** (e.g., `validation` block on `location`).  

---

**This guide is your complete Terraform fundamentals reference.** Bookmark it. Revisit sections as needed. Remember:  
1.  **Write** declarative configs.  
2.  **INIT** before anything else.  
3.  **PLAN** religiously – it's your safety net.  
4.  **APPLY** only after reviewing the plan.  
5.  **DESTROY** test environments.  
6.  **STATE** is sacred – use remote backends.  
