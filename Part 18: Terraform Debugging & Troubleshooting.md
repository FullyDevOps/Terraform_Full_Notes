### **18.1 Reading Terraform Plan Output**  
*Why it matters:* The `terraform plan` output is your **primary safety net** before changes hit production. Misinterpreting it causes catastrophic errors.

#### **Key Components & How to Read Them:**
1. **Operation Symbols (Critical!):**  
   - `+` = **Resource to be CREATED** (e.g., `+ aws_s3_bucket.example`)  
     *Risk:* Accidental creation of expensive resources (e.g., RDS instances).  
   - `-` = **Resource to be DESTROYED** (e.g., `- aws_s3_bucket.old`)  
     *Risk:* **DATA LOSS!** Always verify if destruction is intentional.  
   - `~` = **Resource to be MODIFIED** (e.g., `~ aws_instance.web`)  
     *Risk:* Modifications may cause downtime (e.g., changing an EC2 instance type).  
   - `<=` = **Data source read** (e.g., `<= data.aws_ami.ubuntu`)  
     *Note:* Harmless but indicates external data dependencies.  
   - `<=` + `-` = **Resource to be REPLACED** (e.g., `-/+ aws_db_instance.prod`)  
     *Critical:* Replacement destroys old resource *before* creating new one → **downtime/data loss**.

2. **Attribute-Level Changes:**  
   ```diff
   ~ tags = {
       - "Environment" = "dev"
       + "Environment" = "prod"
     }
   ```  
   - `-` = Value to be **removed**  
   - `+` = Value to be **added**  
   *Watch for:* Accidental removal of critical tags (e.g., `CostCenter`).

3. **"Tainted" Resources:**  
   ```diff
   # aws_s3_bucket.corrupted: tainted
   -/+ resource "aws_s3_bucket" "corrupted" {
   ```  
   *Meaning:* Resource was manually modified outside Terraform → **Force recreation on next apply**.

4. **"No Changes" Output:**  
   ```diff
   Your configuration has been successfully validated!
   No changes. Infrastructure is up-to-date.
   ```  
   *But:* Always check if this is *expected*. If you made config changes but see this, **state drift may exist** (see 18.2).

#### **Pro Tips:**  
- **Color Matters:** Red = destructive, Yellow = modification, Green = creation.  
- **Always run `terraform plan -out=tfplan`** → Review offline → Apply with `terraform apply tfplan`.  
- **Use `terraform plan -detailed-exitcode`** in CI/CD: Exit code `2` = changes planned (safe to apply), `1` = error.  
- **Danger Zone:** If you see `-/+` on a stateful resource (e.g., database), **STOP and investigate** why replacement is needed.

---

### **18.2 Debugging State Drift**  
*What is Drift?* When real infrastructure **diverges** from Terraform's state (e.g., manual AWS console changes).

#### **How to Detect Drift:**  
1. **`terraform plan` (Primary Method):**  
   ```bash
   terraform plan
   ```  
   - Outputs changes like `~ aws_s3_bucket.example: tags have changed` → **Drift detected!**  
   - *Why it works:* Terraform implicitly runs `refresh` (state sync) before planning.

2. **Explicit Drift Detection (Terraform 1.5+):**  
   ```bash
   terraform plan -refresh-only -out=drift.tfplan
   terraform show drift.tfplan
   ```  
   - `-refresh-only` **only** checks drift without planning changes → Zero risk.  
   - `terraform show` displays *exactly* what drifted.

#### **Why Drift is Dangerous:**  
- Terraform **assumes state is truth** → Applying changes may overwrite manual fixes or cause conflicts.  
- Example: Manual security group rule added → Next `apply` deletes it → **Security breach**.

#### **Fixing Drift:**  
| Scenario                          | Solution                                                                 |
|-----------------------------------|--------------------------------------------------------------------------|
| **Intentional manual change**     | 1. `terraform apply -refresh-only` to sync state with reality.<br>2. Update config to match reality (prevents future drift). |
| **Accidental manual change**      | 1. Revert change in cloud console.<br>2. `terraform apply -refresh-only` to reset state. |
| **Drift in immutable resource**   | **Replace resource:** `terraform apply -replace="aws_s3_bucket.broken"` |

#### **Preventing Drift:**  
- **IAM Policies:** Restrict cloud console access (e.g., AWS IAM deny `s3:*` except via Terraform role).  
- **Drift Monitoring:** Use tools like [AWS Config](https://aws.amazon.com/config/) or [Datadog Terraform Drift Detection](https://www.datadoghq.com/blog/terraform-drift-detection/).  
- **Policy as Code:** Enforce with [Open Policy Agent (OPA)](https://www.openpolicyagent.org/) or [Sentinel](https://www.hashicorp.com/sentinel).

---

### **18.3 Using TF_LOG for Debug Logs**  
*When to use:* When Terraform fails with vague errors (e.g., `Error: failed to create EC2 instance`).

#### **Step-by-Step Debugging:**  
1. **Set Log Level (Environment Variables):**  
   ```bash
   # Linux/macOS
   export TF_LOG=DEBUG  # Most useful level
   export TF_LOG_PATH="terraform-debug.log"  # Save to file

   # Windows (PowerShell)
   $env:TF_LOG="DEBUG"
   $env:TF_LOG_PATH="terraform-debug.log"
   ```

2. **Log Levels Explained:**  
   | Level    | When to Use                                  | Output Volume |  
   |----------|---------------------------------------------|---------------|  
   | `TRACE`  | **Provider/plugin internals** (e.g., HTTP requests) | Extreme (1000s of lines) |  
   | `DEBUG`  | **Most common**: Resource operations, state sync | High (100s of lines) |  
   | `INFO`   | Basic workflow (init/plan/apply steps)       | Medium        |  
   | `WARN`   | Only warnings (default for non-interactive)  | Low           |  
   | `ERROR`  | Only errors (useless for debugging)          | Minimal       |  

3. **Critical Log Sections to Check:**  
   - **Provider Calls:**  
     ```log
     [DEBUG] plugin.terraform-provider-aws_v4.70.0: 2023-10-05T12:00:00.000Z [DEBUG] [aws-sdk-go] > PUT /v1/resources/... HTTP/1.1
     ```  
     *Look for:* HTTP 4xx/5xx errors, missing permissions.  
   - **State Operations:**  
     ```log
     [DEBUG] state: state read complete: 12 resources
     [DEBUG] state: state write complete
     ```  
     *Look for:* Failures during state lock/unlock.  
   - **Resource Lifecycle:**  
     ```log
     [DEBUG] aws_instance.web: applying the planned Create change
     ```  
     *Look for:* Stuck operations (e.g., hanging at "Creating...").

4. **After Debugging:**  
   ```bash
   unset TF_LOG  # Linux/macOS
   unset TF_LOG_PATH
   ```

#### **Pro Tips:**  
- **Never commit logs** containing `TF_LOG=TRACE` → **Exposes secrets** (API keys, passwords).  
- For **provider-specific issues**, set `TF_LOG=DEBUG` + `TF_LOG_PATH` and search for `[DEBUG] plugin.terraform-provider-<name>`.  
- **Windows Users:** Use PowerShell (CMD has encoding issues with debug logs).

---

### **18.4 Handling State Conflicts & Locking Issues**  
*Root Cause:* Multiple Terraform processes trying to modify state **simultaneously**.

#### **How State Locking Works:**  
- **Backend Requirement:** Only works with remote backends (S3, Terraform Cloud, etc.). Local state **has no locking**.  
- **Lock File:** When `terraform apply` starts, it creates a lock entry (e.g., DynamoDB item in AWS).  
- **Conflict Example:**  
  ```bash
  Error: Error acquiring state lock
  
  Lock ID: my-lock-id
  Path: example/terraform.tfstate
  Error: ConditionalCheckFailedException: The conditional request failed
  ```  
  *Translation:* Another process holds the lock.

#### **Resolving Lock Conflicts:**  
1. **Wait:** If the other process is legitimate (e.g., teammate's apply), wait for completion.  
2. **Break Lock (Only if safe!):**  
   ```bash
   terraform force-unlock LOCK_ID
   ```  
   - **LOCK_ID** = ID from the error message (e.g., `my-lock-id`).  
   - **DANGER:** Only do this if you're **100% sure** the lock is stale (e.g., process crashed).  
3. **Manual Backend Unlock:**  
   - **S3 + DynamoDB:** Delete the lock item in DynamoDB table.  
   - **Terraform Cloud:** Use UI → Workspace → State → Unlock.  
   - **Never delete S3 state file directly!**  

#### **Preventing Lock Conflicts:**  
- **Short-Lived Applies:** Break large deploys into smaller modules.  
- **CI/CD Queues:** Use tools like [Atlantis](https://www.runatlantis.io/) to serialize `apply` jobs.  
- **State Isolation:** Use separate workspaces for environments (dev/staging/prod).  

#### **Critical Warning:**  
> ⚠️ **`terraform force-unlock` is a last resort!** Breaking a live lock corrupts state. Always verify:  
> - No other `apply` is running (check CI/CD pipelines, teammate screens).  
> - The lock is older than your timeout (e.g., > 1 hour).

---

### **18.5 Fixing `terraform apply` Failures**  
*Why it happens:* Resource creation/modification fails (e.g., AWS quota exceeded, invalid config).

#### **Step-by-Step Recovery:**  
1. **Identify Failure Point:**  
   ```bash
   terraform apply
   # Fails at:
   # aws_s3_bucket.example: Creating...
   # Error: BucketAlreadyExists
   ```  
   *Meaning:* Bucket name `example` is globally taken on AWS.

2. **Check Partial State:**  
   - Terraform **saves state up to the failure point**.  
   - Run `terraform state list` → Resources *before* the failed one exist.  
   - *Critical:* **Do not rerun blindly!** Fix the error first.

3. **Common Fixes:**  
   | Error Type                  | Solution                                                                 |
   |-----------------------------|--------------------------------------------------------------------------|
   | **Quota Exceeded**          | Request limit increase (e.g., AWS Service Quotas) → `terraform apply` again. |
   | **Invalid Config**          | Fix HCL (e.g., wrong AMI ID) → `terraform apply` (no need for `-refresh-only`). |
   | **Dependency Failure**      | Manually create dependency (e.g., VPC) → `terraform import` → `apply`.    |
   | **Transient Cloud Error**   | Wait 5 mins → `terraform apply` (Terraform is idempotent).               |

4. **After Fixing:**  
   ```bash
   terraform apply  # Resumes from failure point
   ```

#### **Advanced Scenarios:**  
- **"Resource Already Exists" (But Not in State):**  
  - **Cause:** Manual creation outside Terraform.  
  - **Fix:** `terraform import aws_s3_bucket.example existing-bucket-name` → Then `apply`.  
- **"Timeout Waiting for State":**  
  - **Cause:** Resource created but Terraform can't verify (e.g., slow AWS API).  
  - **Fix:** Check cloud console → If resource exists, `terraform apply` will recover state.

#### **Golden Rule:**  
> ✅ **Never delete `.terraform` or state files** after a failure. Terraform can recover.  
> ❌ **Never run `terraform destroy`** to "fix" an apply failure → You'll lose partial resources.

---

### **18.6 Recovering from Corrupted State**  
*Corruption Signs:* `terraform plan` fails with `Error refreshing state: ...`, state file has invalid JSON.

#### **Recovery Strategies (Ordered by Safety):**  
1. **Use Backup File (Safest):**  
   - Terraform auto-saves state to `terraform.tfstate.backup` before operations.  
   - **Recover:**  
     ```bash
     cp terraform.tfstate.backup terraform.tfstate  # Local state
     # OR for remote state:
     terraform state pull > corrupt.tfstate
     cp corrupt.tfstate terraform.tfstate  # Then push
     ```

2. **Salvage from `.terraform/` Directory:**  
   - If state was partially written, check:  
     ```bash
     cat .terraform/terraform.tfstate  # May contain valid state
     ```

3. **Manual State Repair (Advanced):**  
   - **Step 1:** Export corrupt state:  
     ```bash
     terraform state pull > corrupt.tfstate
     ```  
   - **Step 2:** Fix JSON manually (e.g., remove invalid characters).  
   - **Step 3:** Validate:  
     ```bash
     terraform validate corrupt.tfstate  # Checks JSON structure
     ```  
   - **Step 4:** Push fixed state:  
     ```bash
     terraform state push corrupt.tfstate
     ```  
   *Warning:* Only do this if you understand state file structure.

4. **Rebuild State from Scratch (Last Resort):**  
   - **Step 1:** `terraform init` (to load providers).  
   - **Step 2:** For **every resource**, run:  
     ```bash
     terraform import aws_s3_bucket.example existing-bucket-name
     ```  
   - **Step 3:** Verify with `terraform plan` → Should show "No changes".

#### **Preventing Corruption:**  
- **Always use remote state** (S3, TFC) → Built-in versioning and locking.  
- **Never edit `terraform.tfstate` directly** → Use `terraform state` commands.  
- **Enable state versioning** (e.g., S3 versioning) → Roll back to last good state.

---

### **18.7 Using `refresh_only` and `replace_triggered_by`**  
*Specialized tools for edge cases.*

#### **`refresh_only` (Drift Correction):**  
- **Purpose:** Sync state with reality **without modifying infrastructure**.  
- **Use Case:** After manual fix in cloud console, update state to match.  
- **Command:**  
  ```bash
  terraform apply -refresh-only -auto-approve
  ```  
- **Critical Notes:**  
  - Does **NOT** change config → Only updates state.  
  - **Never use `-replace` with `-refresh-only`** (it will destroy resources!).  
  - Best paired with `-target` for specific resources:  
    ```bash
    terraform apply -refresh-only -target=aws_s3_bucket.drifted
    ```

#### **`replace_triggered_by` (Controlled Recreation):**  
- **Purpose:** Force recreation of a resource **only when a specific attribute changes** (avoids full replacement cycles).  
- **Use Case:** Immutable infrastructure (e.g., recreate EC2 instance when AMI changes).  
- **Example:**  
  ```hcl
  resource "aws_instance" "web" {
    ami           = "ami-123456"  # Change this to trigger replacement
    instance_type = "t3.micro"
    
    lifecycle {
      replace_triggered_by = [ami]  # Only replace when AMI changes
    }
  }
  ```  
- **Why it beats `terraform taint`:**  
  - Automatic (no manual `taint` step).  
  - Granular (only triggers on specific changes).  
- **Critical Limitation:**  
  - Only works for **attributes that support in-place updates** (e.g., `ami` on EC2). Won't work for `instance_type`.

#### **When to Use These:**  
| Scenario                                      | Tool                     |  
|-----------------------------------------------|--------------------------|  
| Manual cloud fix needs state sync             | `refresh_only`           |  
| Avoid full replacement cycle for AMI updates  | `replace_triggered_by`   |  
| Fixing drift without config changes           | `refresh_only`           |  
| Replacing resource due to hidden dependency   | `replace_triggered_by`   |  

---

### **18.8 Common Errors & Solutions**  
*Real-world issues with exact fixes.*

#### **1. "Provider registry.terraform.io not found"**  
- **Cause:** Missing `terraform init` or corrupted plugin cache.  
- **Fix:**  
  ```bash
  rm -rf .terraform  # Delete plugin cache
  terraform init     # Re-download providers
  ```

#### **2. "Circular Dependency"**  
- **Error:** `Error: Cycle: aws_security_group.web, aws_security_group.db`  
- **Cause:** Group A references Group B, which references Group A.  
- **Fix:**  
  - Break cycle with [security group rules](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule) instead of groups:  
    ```hcl
    resource "aws_security_group_rule" "db_ingress" {
      type        = "ingress"
      cidr_blocks = [aws_security_group.web.id]  # Reference ID, not the group itself
    }
    ```

#### **3. "Timeout waiting for state"**  
- **Cause:** Cloud API slow/unresponsive (common with Azure/AWS GovCloud).  
- **Fix:**  
  ```hcl
  # In provider config
  provider "aws" {
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    max_retries                 = 10  # Default is 2 → Increase for slow regions
  }
  ```

#### **4. "Module not found"**  
- **Error:** `Error: Failed to read module directory`  
- **Cause:**  
  - Forgot `terraform init` after adding module.  
  - Module source uses `../` but path is wrong.  
- **Fix:**  
  ```bash
  terraform init -upgrade  # Re-download modules
  ```

#### **5. "Inconsistent Configuration"**  
- **Error:** `The configuration is invalid: ...`  
- **Cause:** Syntax error (e.g., missing `}`) or invalid attribute.  
- **Fix:**  
  ```bash
  terraform validate  # Shows exact line number of error
  ```

#### **6. "State is corrupt"**  
- **Error:** `State file could not be parsed as JSON`  
- **Fix:** Use **Section 18.6** (Recovering from Corrupted State).

#### **7. "No valid credential sources"**  
- **Cause:** AWS/Azure/GCP credentials not configured.  
- **Fix:**  
  - AWS: `aws configure` or set `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`.  
  - Azure: `az login` or set `ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, etc.  
  - GCP: `gcloud auth application-default login`.

#### **8. "Resource already exists" (But Not in State)**  
- **Cause:** Resource created manually outside Terraform.  
- **Fix:** `terraform import aws_s3_bucket.example existing-bucket-name`.

---

### **Critical Best Practices Summary**  
1. **Always run `terraform plan`** before `apply` → Never skip.  
2. **Use remote state** with versioning (S3 + DynamoDB) → Avoids 90% of state issues.  
3. **Never edit state files manually** → Use `terraform state` commands.  
4. **Treat `terraform force-unlock` like a chainsaw** → Only when absolutely necessary.  
5. **Enable `TF_LOG=DEBUG`** for all non-trivial applies in CI/CD → Logs are gold.  
6. **Isolate environments** with workspaces → Prevents prod/dev conflicts.  
7. **Test drift detection** weekly → Catch issues before they escalate.  

> 💡 **Pro Tip:** Bookmark [Terraform State Commands Cheatsheet](https://developer.hashicorp.com/terraform/cli/commands/state) – You'll need it during outages.
