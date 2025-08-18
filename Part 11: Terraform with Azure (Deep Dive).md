### **11.1 Azure Provider Setup**  
**What it is**: The foundation for *all* Azure Terraform operations. Configures the connection between Terraform and Azure.  
**Why it matters**: Without this, Terraform can't interact with Azure. Uses the **`azurerm` provider** (formerly `azureRM`).  

#### **Key Configuration**  
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0" # Always pin versions!
    }
  }
}

provider "azurerm" {
  features {} # REQUIRED (even if empty) for v3+ of provider
  # Authentication handled separately (see 11.2)
}
```

#### **Critical Details**  
- **`features {}`**: **MANDATORY** in provider v3.0+. Omitting it causes errors. Controls Azure resource deletion behavior (e.g., `virtual_machine` deletion).  
- **Version Pinning**: Always specify `version = "~> 3.0"`. AzureRM changes rapidly; unpinned versions break deployments.  
- **Alias Support**: For multi-environment (e.g., `provider "azurerm" { alias = "prod" }`).  
- **Environment**: Set `environment = "public"` (default), `usgovernment`, `german`, or `china` for sovereign clouds.  
- **Subscription ID**: Can be set here (`subscription_id = "xxx"`) or via auth (11.2).  

> ⚠️ **Gotcha**: `features {}` must be **empty** if you don't need special deletion behaviors. Never remove it entirely.

---

### **11.2 Authentication (Service Principal, CLI, Managed Identity)**  
**Why it matters**: Securely authenticates Terraform to Azure. **Never hardcode credentials!**  

#### **Option 1: Service Principal (SP) - *Recommended for CI/CD***  
- **What**: Azure AD application + password/certificate.  
- **Setup**:  
  ```bash
  az ad sp create-for-rbac --name "tf-sp" --role "Contributor" --scopes "/subscriptions/SUB_ID"
  ```
- **Terraform Config**:  
  ```hcl
  provider "azurerm" {
    client_id     = "APP_ID"       # From `appId` in CLI output
    client_secret = "PASSWORD"     # From `password`
    tenant_id     = "TENANT_ID"    # From `tenant`
    subscription_id = "SUB_ID"
  }
  ```
- **Best Practices**:  
  - Assign **least-privilege RBAC** (e.g., `Contributor` at Resource Group level, not Subscription).  
  - Store secrets in **Azure Key Vault** + Terraform Cloud/Enterprise variables.  
  - Use **certificates** (not passwords) for SPs in production.  

#### **Option 2: Azure CLI (az login) - *For local dev only***  
- **How it works**: Uses your `az login` session token.  
- **Config**: Zero config needed! Just run `az login` before `terraform apply`.  
- **Limitations**:  
  - **NOT for CI/CD** (ephemeral tokens expire).  
  - Uses **your personal permissions** (security risk in teams).  

#### **Option 3: Managed Identity (MI) - *For Azure-hosted Terraform***  
- **What**: Azure resource (VM, Function) has auto-managed identity.  
- **Use Case**: Terraform runs **inside Azure** (e.g., Azure DevOps agent on VM, GitHub Actions with Azure login).  
- **Config**:  
  ```hcl
  provider "azurerm" {
    use_msi = true
    # msi_endpoint = "http://169.254.169.254/metadata/identity" (auto-detected)
  }
  ```
- **Critical Steps**:  
  1. Enable **System-Assigned Identity** on the compute resource (VM, etc.).  
  2. Grant the MI **RBAC permissions** in Azure (e.g., `Contributor` on RG).  
- **Why it's secure**: No secrets stored; identity is Azure-managed.  

> 🔑 **Golden Rule**: **SP for CI/CD, CLI for local dev, MI for Azure-hosted runners.** Avoid environment variables for secrets.

---

### **11.3 Resource Groups, VNets, Subnets**  
**Core Networking Building Blocks**  

#### **Resource Group (`azurerm_resource_group`)**  
- **Purpose**: Top-level container for Azure resources (billing, RBAC, lifecycle).  
- **Critical Config**:  
  ```hcl
  resource "azurerm_resource_group" "rg" {
    name     = "prod-rg"
    location = "eastus2" # Must be valid Azure region
    tags = { environment = "prod" }
  }
  ```
- **Gotcha**: **Location is immutable**. Changing it requires `terraform taint` or recreation.  

#### **Virtual Network (`azurerm_virtual_network`)**  
- **Purpose**: Isolated private network in Azure.  
- **Key Config**:  
  ```hcl
  resource "azurerm_virtual_network" "vnet" {
    name                = "core-vnet"
    address_space       = ["10.0.0.0/16"]
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
  }
  ```
- **Critical Notes**:  
  - `address_space` must be **non-overlapping** CIDR blocks.  
  - **DNS servers** can be customized via `dns_servers`.  

#### **Subnet (`azurerm_subnet`)**  
- **Purpose**: Segment of VNet for security/scalability.  
- **Config**:  
  ```hcl
  resource "azurerm_subnet" "app_subnet" {
    name                 = "app-subnet"
    resource_group_name  = azurerm_resource_group.rg.name
    virtual_network_name = azurerm_virtual_network.vnet.name
    address_prefixes     = ["10.0.1.0/24"]
    
    # Specialized subnet (e.g., for Azure Firewall)
    service_endpoints    = ["Microsoft.Storage", "Microsoft.Sql"]
  }
  ```
- **Advanced Use Cases**:  
  - **Delegation**: For AKS, App Gateway (`delegation { ... }`).  
  - **Network Security Group (NSG)**: Attach via `network_security_group_id`.  
  - **Route Table**: Attach via `route_table_id`.  
- **Gotcha**: Subnet **address prefixes must be within VNet's CIDR**. Terraform won't validate this!  

> 🌐 **Best Practice**: Always deploy resources in the **same region** as their Resource Group. Cross-region resources cause latency and complexity.

---

### **11.4 Virtual Machines & Availability Sets**  
**Compute Infrastructure**  

#### **Virtual Machine (`azurerm_linux_virtual_machine` / `azurerm_windows_virtual_machine`)**  
- **Minimal Config**:  
  ```hcl
  resource "azurerm_linux_virtual_machine" "vm" {
    name                = "web-vm"
    resource_group_name = azurerm_resource_group.rg.name
    location            = azurerm_resource_group.rg.location
    size                = "Standard_B2s"
    admin_username      = "adminuser"
    network_interface_ids = [azurerm_network_interface.nic.id]
    
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
    
    admin_ssh_key {
      username   = "adminuser"
      public_key = file("~/.ssh/id_rsa.pub")
    }
  }
  ```
- **Critical Components**:  
  - **Network Interface (NIC)**: Must be created separately (`azurerm_network_interface`).  
  - **OS Disk**: Type (`Standard_LRS`, `Premium_LRS`), caching.  
  - **Image**: Use `source_image_id` for custom images.  
  - **SSH/RDP**: Linux uses `admin_ssh_key`, Windows uses `admin_password`.  

#### **Availability Set (`azurerm_availability_set`)**  
- **Purpose**: Protect against **datacenter rack failures** (not zone failures).  
- **Config**:  
  ```hcl
  resource "azurerm_availability_set" "avset" {
    name                = "web-avset"
    resource_group_name = azurerm_resource_group.rg.name
    location            = azurerm_resource_group.rg.location
    platform_fault_domain_count   = 2
    platform_update_domain_count  = 2
    managed = true # Always use managed disks!
  }
  ```
- **Attach VM to AS**:  
  ```hcl
  resource "azurerm_linux_virtual_machine" "vm" {
    # ... other config ...
    availability_set_id = azurerm_availability_set.avset.id
  }
  ```
- **Key Notes**:  
  - **Fault Domains**: Physical server racks (max 3).  
  - **Update Domains**: Groups updated together (max 20).  
  - **Managed = true**: **MANDATORY** for modern Azure. Unmanaged disks are deprecated.  
  - **Not for Zones**: Use **Availability Zones** (separate resource) for zone-level redundancy.  

> ⚠️ **Gotcha**: You **cannot add a VM to an AS after creation**. Plan early! AS has no cost, but limits VM SKUs.

---

### **11.5 Azure Load Balancer & Application Gateway**  
**Traffic Distribution**  

#### **Azure Load Balancer (L4 - TCP/UDP) (`azurerm_lb`)**  
- **Purpose**: Distribute traffic across VMs (high availability).  
- **Key Resources**:  
  ```hcl
  # Public IP for LB
  resource "azurerm_public_ip" "lb_ip" {
    name                = "lb-pip"
    allocation_method   = "Static"
    sku                 = "Standard"
  }
  
  # Load Balancer
  resource "azurerm_lb" "lb" {
    name                = "web-lb"
    sku                 = "Standard"
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    frontend_ip_configuration {
      name                 = "public"
      public_ip_address_id = azurerm_public_ip.lb_ip.id
    }
  }
  
  # Backend Pool (VMs)
  resource "azurerm_lb_backend_address_pool" "bepool" {
    loadbalancer_id = azurerm_lb.lb.id
    name            = "bepool"
  }
  
  # Health Probe
  resource "azurerm_lb_probe" "probe" {
    loadbalancer_id = azurerm_lb.lb.id
    protocol        = "http"
    request_path    = "/health"
    port            = 80
  }
  
  # Rule (Port 80 -> VMs)
  resource "azurerm_lb_rule" "rule" {
    loadbalancer_id            = azurerm_lb.lb.id
    frontend_ip_configuration_name = "public"
    protocol                   = "tcp"
    frontend_port              = 80
    backend_port               = 80
    backend_address_pool_id    = azurerm_lb_backend_address_pool.bepool.id
    probe_id                   = azurerm_lb_probe.probe.id
  }
  ```
- **Critical Notes**:  
  - **Standard SKU**: Required for HA, zone redundancy. Basic SKU is legacy.  
  - **Health Probe**: **Mandatory** for backend pool. Without it, traffic isn't routed.  
  - **Outbound Rules**: Needed for VMs to access internet (configure via `azurerm_lb_outbound_rule`).  

#### **Application Gateway (L7 - HTTP/S) (`azurerm_application_gateway`)**  
- **Purpose**: Web traffic routing, SSL termination, WAF.  
- **Simplified Config**:  
  ```hcl
  resource "azurerm_application_gateway" "appgw" {
    name                = "web-appgw"
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    sku {
      name     = "WAF_v2"
      tier     = "WAF_v2"
      capacity = 2
    }
    
    # Frontend (Public IP)
    frontend_ip_configuration {
      name      = "public"
      public_ip_address_id = azurerm_public_ip.appgw_pip.id
    }
    
    # Backend Pool (VMs or IPs)
    backend_address_pool {
      name = "pool1"
      fqdns = ["webapp1.azurewebsites.net"]
    }
    
    # HTTP Settings (Port, protocol, probe)
    backend_http_settings {
      name                  = "setting1"
      port                  = 80
      protocol              = "Http"
      cookie_based_affinity = "Enabled"
      request_timeout       = 30
      probe_name            = "probe1"
    }
    
    # Health Probe
    probe {
      name     = "probe1"
      protocol = "Http"
      path     = "/health"
      interval = 30
      timeout  = 30
      unhealthy_threshold = 3
    }
    
    # Listener (Port 80)
    http_listener {
      name                           = "listener1"
      frontend_ip_configuration_name = "public"
      frontend_port_name             = "port80"
      protocol                       = "Http"
    }
    
    # Routing Rule
    request_routing_rule {
      name      = "rule1"
      rule_type = "Basic"
      http_listener_name = "listener1"
      backend_address_pool_name = "pool1"
      backend_http_settings_name = "setting1"
    }
  }
  ```
- **Key Differences from LB**:  
  - **SSL Termination**: Handles HTTPS decryption at gateway.  
  - **URL Routing**: Route based on path (e.g., `/api` vs `/web`).  
  - **WAF**: Built-in Web Application Firewall (enable via `web_application_firewall_configuration`).  
  - **Autoscaling**: Supported in v2 SKUs.  

> 🔥 **Critical Gotcha**: App Gateway **requires a dedicated subnet** (`azurerm_subnet` with delegation to `Microsoft.Network/applicationGateways`). Don't reuse subnets!

---

### **11.6 Azure SQL & Storage Accounts**  
**Managed Data Services**  

#### **Azure SQL Database (`azurerm_mssql_server` + `azurerm_mssql_database`)**  
- **Server (Instance)**:  
  ```hcl
  resource "azurerm_mssql_server" "sql" {
    name                         = "prod-sqlserver"
    resource_group_name          = azurerm_resource_group.rg.name
    location                     = azurerm_resource_group.rg.location
    version                      = "12.0"
    administrator_login          = "sqladmin"
    administrator_login_password = "P@ssw0rd123!" # NEVER hardcode! Use Key Vault
    minimum_tls_version          = "1.2"
    
    azuread_administrator {
      login_username = "AzureADAdmin"
      object_id      = "AAD_GROUP_OBJECT_ID"
      tenant_id      = data.azurerm_client_config.current.tenant_id
    }
  }
  ```
- **Database**:  
  ```hcl
  resource "azurerm_mssql_database" "db" {
    name           = "appdb"
    server_id      = azurerm_mssql_server.sql.id
    collation      = "SQL_Latin1_General_CP1_CI_AS"
    license_type   = "LicenseIncluded"
    max_size_gb    = 5
    sku_name       = "GP_S_Gen5_2" # General Purpose, Serverless
  }
  ```
- **Critical Security**:  
  - **Firewall Rules**: Must allow Terraform runner IP (`azurerm_mssql_firewall_rule`).  
  - **AD Auth**: Use `azuread_administrator` for secure access (requires AAD group).  
  - **Never expose admin password** in state! Use `sensitive = true` + remote state.  

#### **Storage Account (`azurerm_storage_account`)**  
- **Purpose**: Blob, File, Queue, Table storage.  
- **Config**:  
  ```hcl
  resource "azurerm_storage_account" "storage" {
    name                     = "prodstorage123" # Must be globally unique!
    resource_group_name      = azurerm_resource_group.rg.name
    location                 = azurerm_resource_group.rg.location
    account_tier             = "Standard"
    account_replication_type = "LRS" # Or GRS, RA-GRS
    
    # Blob-specific
    is_hns_enabled           = true # For ADLS Gen2
    blob_properties {
      versioning_enabled = true
      delete_retention_policy { days = 7 }
    }
    
    # Secure by default
    allow_blob_public_access = false
    min_tls_version          = "TLS1_2"
  }
  ```
- **Key Options**:  
  - **`account_kind`**: `StorageV2` (most versatile), `BlobStorage`, `FileStorage`.  
  - **`access_tier`**: `Hot` or `Cool` for cost optimization.  
  - **`is_hns_enabled`**: Enable Azure Data Lake Storage Gen2 (hierarchical namespace).  
  - **Private Endpoints**: Use `azurerm_private_endpoint` for secure access.  

> 🔒 **Security Must-Dos**:  
> - Set `allow_blob_public_access = false` (default in new accounts).  
> - Use **SAS tokens** (short-lived) or **Managed Identities** for app access, not account keys.  
> - Enable **encryption at rest** (enabled by default).  

---

### **11.7 Azure AD Applications & Service Principals**  
**Identity for Applications**  

#### **Azure AD Application (`azuread_application`)**  
- **Purpose**: Represents an application in Azure AD (metadata).  
- **Config**:  
  ```hcl
  resource "azuread_application" "app" {
    display_name = "MyApp"
    sign_in_audience = "AzureADMyOrg" # Single-tenant
    
    # Required for SP creation
    owners = [data.azuread_client_config.current.object_id]
    
    # Web app settings
    web {
      redirect_uris = ["https://myapp.com/auth"]
    }
    
    # API permissions (Microsoft Graph)
    required_resource_access {
      resource_app_id = "00000003-0000-0000-c000-000000000000" # Microsoft Graph
      resource_access {
        id   = "e1fe6dd8-ba31-4d61-89e7-88639da4683d" # User.Read
        type = "Scope"
      }
    }
  }
  ```
- **Critical Notes**:  
  - **Not the identity itself** – it's the *definition*. The actual identity is the **Service Principal**.  
  - **Owners**: Required to avoid "Insufficient privileges" errors.  

#### **Service Principal (`azuread_service_principal`)**  
- **Purpose**: The *instance* of the app used for authentication (like a user account for apps).  
- **Config**:  
  ```hcl
  resource "azuread_service_principal" "sp" {
    application_id = azuread_application.app.application_id
    owners         = [data.azuread_client_config.current.object_id]
    
    # Optional: Assign role to SP (e.g., for Terraform)
    depends_on = [azurerm_role_assignment.sp_role]
  }
  
  # Grant SP Contributor on Resource Group
  resource "azurerm_role_assignment" "sp_role" {
    scope                = azurerm_resource_group.rg.id
    role_definition_name = "Contributor"
    principal_id         = azuread_service_principal.sp.object_id
  }
  ```
- **Key Points**:  
  - **Created automatically** when you create an AD App *in the Azure portal*, but **not via Terraform**. You **must** create it explicitly.  
  - **`principal_id`**: Use this for RBAC assignments (not `application_id`).  
  - **Secrets/Certs**: Use `azuread_service_principal_password` or `azuread_service_principal_certificate` resources.  

> 🧠 **Why Two Resources?**  
> - **AD Application**: Template (like a class in OOP).  
> - **Service Principal**: Instance (like an object). One app can have many SPs (e.g., in different directories).  

---

### **11.8 Azure Blob Storage & CDN**  
**Static Content Delivery**  

#### **Blob Storage (in Storage Account)**  
- **Container (`azurerm_storage_container`)**:  
  ```hcl
  resource "azurerm_storage_container" "container" {
    name                  = "mycontainer"
    storage_account_name  = azurerm_storage_account.storage.name
    container_access_type = "private" # Or 'blob' (anonymous read)
  }
  ```
- **Upload Blob (`azurerm_storage_blob`)**:  
  ```hcl
  resource "azurerm_storage_blob" "blob" {
    name             = "index.html"
    storage_account_name = azurerm_storage_account.storage.name
    storage_container_name = azurerm_storage_container.container.name
    type             = "block"
    source           = "path/to/index.html"
  }
  ```

#### **Azure CDN (`azurerm_cdn_endpoint`)**  
- **Purpose**: Cache blobs globally for low latency.  
- **Config**:  
  ```hcl
  # CDN Profile (top-level)
  resource "azurerm_cdn_profile" "cdn" {
    name                = "prod-cdn"
    resource_group_name = azurerm_resource_group.rg.name
    location            = "Global" # Always "Global" for CDN
    sku                 = "Standard_Microsoft"
  }
  
  # CDN Endpoint (maps to origin)
  resource "azurerm_cdn_endpoint" "endpoint" {
    name                = "web-endpoint"
    profile_name        = azurerm_cdn_profile.cdn.name
    resource_group_name = azurerm_resource_group.rg.name
    is_http_allowed     = false
    is_https_allowed    = true
    
    # Origin (your blob storage)
    origin {
      name      = "blob-origin"
      host_name = "${azurerm_storage_account.storage.primary_blob_host}"
    }
  }
  
  # Optional: Custom domain + HTTPS
  resource "azurerm_cdn_endpoint_custom_domain" "domain" {
    name                = "www"
    cdn_endpoint_id     = azurerm_cdn_endpoint.endpoint.id
    hostname            = "www.example.com"
    
    # Managed certificate (requires DNS CNAME)
    managed_https {
      certificate_type = "cdn"
    }
  }
  ```
- **Critical Notes**:  
  - **Origin Path**: Use `origin_path = "/mycontainer"` to serve from specific container.  
  - **HTTPS**: **Always enforce HTTPS** (`is_http_allowed = false`).  
  - **Purge**: Use `azurerm_cdn_endpoint`'s `content_path` to purge cache (Terraform doesn't manage cache).  
  - **WAF**: Add `azurerm_cdn_endpoint` to a WAF policy via `azurerm_cdn_endpoint_security_policy`.  

> 🌍 **Best Practice**: Use **Private Endpoints** for blob storage + CDN to avoid public internet exposure.

---

### **11.9 Azure Functions & Logic Apps**  
**Serverless Automation**  

#### **Azure Functions (`azurerm_function_app`)**  
- **Minimal Config**:  
  ```hcl
  resource "azurerm_function_app" "func" {
    name                      = "prod-func"
    resource_group_name       = azurerm_resource_group.rg.name
    location                  = azurerm_resource_group.rg.location
    app_service_plan_id       = azurerm_app_service_plan.plan.id
    storage_account_name      = azurerm_storage_account.storage.name
    storage_account_access_key = azurerm_storage_account.storage.primary_access_key
    os_type                   = "linux"
    version                   = "~4"
    
    # App Settings (secrets via Key Vault!)
    app_settings = {
      "FUNCTIONS_WORKER_RUNTIME" = "python"
      "AzureWebJobsStorage"      = azurerm_storage_account.storage.primary_connection_string
    }
    
    # Managed Identity for secure access
    identity {
      type = "SystemAssigned"
    }
  }
  ```
- **Critical Components**:  
  - **App Service Plan**: Required (`azurerm_app_service_plan` with `sku_tier = "Dynamic"` for consumption plan).  
  - **Storage Account**: For triggers/state (use private endpoint!).  
  - **Identity**: Assign RBAC to Key Vault/Blob using `identity[0].principal_id`.  
- **Deployment**: Use `azurerm_function_app_slot` for staging, or `azurerm_resource_group_template_deployment` for ARM templates.  

#### **Logic Apps (`azurerm_logic_app_workflow`)**  
- **Purpose**: Visual workflow automation (e.g., "If email, then save to Blob").  
- **Config (ARM template required)**:  
  ```hcl
  resource "azurerm_logic_app_workflow" "logic" {
    name                = "prod-logicapp"
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    
    # Define workflow in JSON (simplified)
    definition = jsonencode({
      "$schema" : "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
      "contentVersion" : "1.0.0.0",
      "parameters" : {},
      "triggers" : {
        "Recurrence" : {
          "recurrence" : {
            "frequency" : "Day",
            "interval" : 1
          },
          "type" : "Recurrence"
        }
      },
      "actions" : {
        "HTTP" : {
          "inputs" : {
            "method" : "GET",
            "uri" : "https://api.example.com/data"
          },
          "type" : "Http"
        }
      }
    })
  }
  ```
- **Key Notes**:  
  - **Definition**: Must be valid **ARM JSON** (use Logic App Designer > "Code View" to generate).  
  - **Connections**: Managed separately (`azurerm_logic_app_trigger_http_request`, `azurerm_logic_app_action_http`).  
  - **Integration Account**: Required for B2B scenarios (separate resource).  

> ⚡ **Serverless Tip**: Use **Managed Identities** for both Functions/Logic Apps to access Azure services securely (no secrets!).

---

### **11.10 Using AzureRM Data Sources**  
**Import Existing Azure Resources into Terraform**  

#### **What are Data Sources?**  
- **Purpose**: Read **existing** Azure resources into Terraform state (e.g., pre-existing VNet, RG).  
- **Syntax**: `data "azurerm_RESOURCE" "NAME" { ... }`  
- **Why use them**:  
  - Reference resources **not managed by Terraform** (e.g., shared network).  
  - Avoid hardcoding IDs/locations.  
  - Enable modular Terraform (modules consume data sources).  

#### **Critical Examples**  
1. **Get Resource Group Location**:  
   ```hcl
   data "azurerm_resource_group" "shared_rg" {
     name = "shared-rg"
   }
   
   resource "azurerm_virtual_network" "vnet" {
     location = data.azurerm_resource_group.shared_rg.location # Dynamic!
   }
   ```

2. **Reference Shared VNet**:  
   ```hcl
   data "azurerm_virtual_network" "shared_vnet" {
     name                = "core-vnet"
     resource_group_name = "network-rg"
   }
   
   resource "azurerm_subnet" "app_subnet" {
     virtual_network_name = data.azurerm_virtual_network.shared_vnet.name
     # ... 
   }
   ```

3. **Get Client Config (Subscription/Tenant)**:  
   ```hcl
   data "azurerm_client_config" "current" {}
   
   output "subscription_id" {
     value = data.azurerm_client_config.current.subscription_id
   }
   ```

4. **Get Storage Account Keys (Securely)**:  
   ```hcl
   data "azurerm_storage_account" "storage" {
     name                = "prodstorage123"
     resource_group_name = "storage-rg"
   }
   
   resource "azurerm_function_app" "func" {
     app_settings = {
       "AzureWebJobsStorage" = data.azurerm_storage_account.storage.primary_connection_string
     }
   }
   ```
   > 🔒 **Security Note**: Never output storage keys! Terraform masks them by default.

#### **When NOT to Use Data Sources**  
- **Managing the resource**: If Terraform should *own* the resource, use a `resource`, not `data`.  
- **Frequent changes**: Data sources are read at **plan time** – if the source changes mid-plan, it may cause drift.  

> 💡 **Pro Tip**: Combine with **Terraform Modules**. Data sources let modules consume external resources without hardcoding.

---

## ✅ Key Takeaways for Your Reference  
1. **Provider Setup**: Always use `features {}` and pin versions.  
2. **Authentication**: SP for CI/CD, MI for Azure-hosted runners, CLI for local dev. **Never secrets in code!**  
3. **Networking**: Resource Group → VNet → Subnet. Subnets need NSGs/Route Tables.  
4. **Compute**: VMs need NICs + Disks. Use Availability Sets/Zones for HA.  
5. **Load Balancing**: LB (L4) for TCP, App Gateway (L7) for HTTP + WAF.  
6. **Data Services**: SQL Server needs firewall rules. Storage accounts require unique names.  
7. **Identity**: AD App + Service Principal = application identity. Assign RBAC via `principal_id`.  
8. **Blob + CDN**: CDN requires origin (blob URL). Always enforce HTTPS.  
9. **Serverless**: Functions need App Service Plan + Storage. Logic Apps require ARM JSON.  
10. **Data Sources**: For **reading** existing resources (not managing them).  
