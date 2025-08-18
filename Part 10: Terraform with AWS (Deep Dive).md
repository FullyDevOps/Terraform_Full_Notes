### **10.1 AWS Provider Configuration**  
**What it is:** The foundational block telling Terraform *how* to connect to AWS.  
**Why it matters:** Without this, Terraform can't interact with AWS.  
**Key Configuration Parameters:**  
```hcl
provider "aws" {
  region = "us-east-1" # Mandatory: AWS region (e.g., us-west-2, eu-central-1)
  profile = "dev-profile" # Optional: AWS CLI profile name (uses ~/.aws/credentials)
  shared_credentials_file = "/path/to/creds" # Custom credentials file path
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "TerraformSession"
    external_id  = "OPTIONAL_EXTERNAL_ID" # For cross-account security
  }
  default_tags { # Auto-apply tags to ALL AWS resources in this config
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}
```
**Critical Notes:**  
- **`region` is REQUIRED** (unless set via `AWS_REGION` env var).  
- **Avoid hardcoding credentials** – use `profile` or IAM roles (see 10.2).  
- **`default_tags`** prevents manual tagging errors (inherited by resources supporting tags).  
- **Provider Aliasing:** Use `alias = "east"` to manage multiple regions/accounts in one config:  
  ```hcl
  provider "aws" { alias = "east"; region = "us-east-1" }
  resource "aws_s3_bucket" "east_bucket" { provider = aws.east; ... }
  ```

---

### **10.2 Authentication (IAM Roles, AWS CLI, Credentials File)**  
**How Terraform Authenticates with AWS:**  
Terraform uses the **AWS SDK for Go** – same auth methods as AWS CLI/SDKs.  

#### **Authentication Methods (Ordered by Security Best Practice):**  
1. **IAM Roles (EC2 Instance Profile / ECS Task Role)**  
   - *Best for:* CI/CD pipelines (CodeBuild, Jenkins on EC2), serverless deployments.  
   - *How:* Attach IAM role to EC2/ECS. Terraform auto-discovers credentials via metadata service (`169.254.169.254`).  
   - *No config needed in Terraform!*  

2. **AWS CLI Profile (`~/.aws/credentials`)**  
   - *Best for:* Local development.  
   - *How:*  
     ```bash
     aws configure --profile dev-profile
     # Enter AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, region
     ```  
   - *Terraform Usage:* `profile = "dev-profile"` in provider block.  

3. **Environment Variables**  
   ```bash
   export AWS_ACCESS_KEY_ID="AKIA..."
   export AWS_SECRET_ACCESS_KEY="..."
   export AWS_SESSION_TOKEN="..." # If using MFA/STS
   ```  
   - *Terraform Usage:* No provider config needed (auto-detected).  

4. **Static Credentials in Provider Block (AVOID)**  
   ```hcl
   provider "aws" {
     access_key = "AKIA..." # ❌ NEVER HARD-CODE IN CODE!
     secret_key = "..."
   }
   ```  
   - **Security Risk:** Credentials leak into state file, version control, logs.  
   - **Only use for testing** with `TF_VAR_` env vars (still risky).  

#### **Critical Security Practices:**  
- **Least Privilege:** IAM policies for Terraform should *only* have permissions for resources it manages.  
- **MFA Enforcement:** Require MFA for `sts:AssumeRole` in IAM policies.  
- **Rotate Credentials:** Use IAM roles (ephemeral tokens) instead of long-term keys.  
- **Never commit `~/.aws/credentials`** to Git! Use `.gitignore`.  

---

### **10.3 Creating VPC, Subnets, Route Tables**  
**Core Components of AWS Networking:**  
#### **VPC (Virtual Private Cloud)**  
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "main-vpc" }
  enable_dns_support = true   # Required for Route 53 private zones
  enable_dns_hostnames = true # Required for EC2 private DNS names
  instance_tenancy = "default" # or "dedicated"
}
```
- **`cidr_block`**: Must not overlap with other networks (e.g., `10.0.0.0/16`, `172.16.0.0/12`).  
- **NAT Gateway?** Requires `enable_dns_hostnames = true`.  

#### **Subnets**  
```hcl
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true # For public subnets (EC2 gets public IP)
  tags = { Tier = "public" }
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
  tags = { Tier = "private" }
}
```
- **Placement:** Spread subnets across AZs for HA (`us-east-1a`, `us-east-1b`, etc.).  
- **Public vs Private:**  
  - *Public:* `map_public_ip_on_launch = true` + Route to Internet Gateway (IGW).  
  - *Private:* No public IP + Route to NAT Gateway.  

#### **Route Tables**  
```hcl
# Internet Gateway (for public subnets)
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
}

# Associate public subnet with public route table
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```
- **Private Route Table Example:**  
  ```hcl
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.ng.id # Requires NAT Gateway in public subnet
  }
  ```
- **Default Route Table:** Automatically created but *not manageable* via Terraform. Always create custom route tables.  

**Gotchas:**  
- Subnets must be associated with a route table (default or custom).  
- Deleting a VPC requires deleting *all* dependent resources first (Terraform handles this via dependencies).  
- Use `aws_vpc_endpoint` for private S3/DynamoDB access (no NAT needed).  

---

### **10.4 EC2 Instances & Key Pairs**  
#### **EC2 Instance**  
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c7217cdde317cfec" # Ubuntu 22.04 LTS (us-east-1)
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id] # NOT security_groups (name-based)

  # IAM Role for EC2 (see 10.10)
  iam_instance_profile = aws_iam_instance_profile.web.name

  # User Data (run on first boot)
  user_data = <<-EOF
              #!/bin/bash
              echo "Hello from Terraform!" > /home/ubuntu/greeting.txt
              EOF

  tags = { Role = "web-server" }
}
```
- **`ami`**: Use `data "aws_ami"` (see 10.14) to avoid hardcoding.  
- **`vpc_security_group_ids`**: Use *security group IDs*, not names (critical!).  
- **`user_data`**: Must be valid shell script. Use `file("init.sh")` for complex scripts.  

#### **Key Pairs**  
**Option 1: Import Existing Key (Recommended)**  
```hcl
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("~/.ssh/id_ed25519.pub") # Path to public key
}
```
- **Never store private keys in Terraform state!** Only public keys are saved.  
- Attach to EC2: `key_name = aws_key_pair.deployer.key_name`  

**Option 2: Generate New Key (Use with Caution)**  
```hcl
resource "tls_private_key" "this" {
  algorithm = "ED25519"
}

resource "aws_key_pair" "generated" {
  key_name   = "generated-key"
  public_key = tls_private_key.this.public_key_openssh
}

# Save private key to file (DO NOT commit to Git!)
resource "local_file" "private_key" {
  content  = tls_private_key.this.private_key_pem
  filename = "generated-key.pem"
  file_permission = "0600"
}
```
- **Security Risk:** Private key leaks into Terraform state unless `sensitive = true` (still visible in plan/apply).  
- **Best Practice:** Use AWS Systems Manager Session Manager instead of SSH keys.  

---

### **10.5 Security Groups & Network ACLs**  
#### **Security Groups (Stateful Firewall)**  
```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP/HTTPS from anywhere"
  vpc_id      = aws_vpc.main.id

  # Ingress (Incoming)
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Egress (Outgoing) - Default: ALLOW ALL
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1" # ALL
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Project = "web" }
}
```
- **Stateful:** Return traffic auto-allowed (e.g., if you allow port 80 ingress, responses on ephemeral ports are allowed).  
- **Order Agnostic:** Rules are evaluated together (no priority numbers).  
- **Use `source_security_group_id` for VPC peering/internal traffic:**  
  ```hcl
  ingress {
    from_port                = 3306
    to_port                  = 3306
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.db.id
  }
  ```

#### **Network ACLs (Stateless Firewall)**  
```hcl
resource "aws_network_acl" "private" {
  vpc_id = aws_vpc.main.id
  subnet_ids = [aws_subnet.private.id]

  # Ingress Rules
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  # Explicit deny rule (required - NACLs default to DENY)
  ingress {
    rule_no    = 200
    protocol   = "-1"
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
}
```
- **Stateless:** Must define rules for *both* request and response (e.g., allow outbound ephemeral ports).  
- **Rule Number Priority:** Lower numbers = higher priority (e.g., rule 100 processed before 200).  
- **Default NACL:** Auto-created with ALLOW ALL rules (but not manageable via Terraform).  
- **When to Use:** Rarely needed! Security Groups are sufficient for 95% of use cases. NACLs are for subnet-level blanket rules (e.g., blocking malicious IPs).  

**Critical Differences:**  
| Feature          | Security Group                     | Network ACL               |
|------------------|------------------------------------|---------------------------|
| **State**        | Stateful                           | Stateless                 |
| **Scope**        | Instance-level                     | Subnet-level              |
| **Rules**        | Allow rules only                   | Allow + Deny rules        |
| **Rule Order**   | Evaluated as a whole (no priority) | Explicit rule numbers     |
| **Default**      | Deny all                           | Deny all (explicit rules) |

---

### **10.6 Auto Scaling Groups & Launch Templates**  
#### **Launch Template (Modern Replacement for Launch Config)**  
```hcl
resource "aws_launch_template" "web" {
  name_prefix   = "web-lt-"
  image_id      = "ami-0c7217cdde317cfec" # Ubuntu
  instance_type = "t3.micro"

  # IAM Role (see 10.10)
  iam_instance_profile {
    name = aws_iam_instance_profile.web.name
  }

  # User Data (cloud-init)
  user_data = base64encode(<<-EOF
              #cloud-config
              runcmd:
                - [ sh, -c, "echo 'Hello from LT' > /home/ubuntu/greeting.txt" ]
              EOF
  )

  # Block Devices (EBS Volumes)
  block_device_mappings {
    device_name = "/dev/sda1"
    ebs {
      volume_size = 20
      volume_type = "gp3"
    }
  }
}
```
- **Immutable:** Updates create new template versions (no in-place edits).  
- **`user_data` must be base64 encoded** (unlike EC2 instances).  
- **Use `name_prefix`** for Terraform-managed naming (prevents conflicts).  

#### **Auto Scaling Group (ASG)**  
```hcl
resource "aws_autoscaling_group" "web" {
  name = "web-asg"
  min_size = 2
  max_size = 5
  desired_capacity = 2
  vpc_zone_identifier = [aws_subnet.private.id, aws_subnet.public.id] # Subnets for instances

  # Link to Launch Template
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest" # Use latest template version
  }

  # Scaling Policies
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 70.0
  }

  tag {
    key                 = "Name"
    value               = "web-server"
    propagate_at_launch = true # Tags EC2 instances
  }
}
```
- **`vpc_zone_identifier`**: Must specify at least 2 subnets for HA.  
- **Scaling Policies:** Target tracking (CPU), step scaling (custom metrics), or scheduled actions.  
- **Instance Refresh:** Replace instances with new LT version:  
  ```hcl
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }
  ```

**Why Launch Templates > Launch Configurations:**  
- Supports Instance Metadata V2, Elastic GPUs, multiple instance types.  
- Versioning enables safe rollbacks.  
- Required for mixed instance policies (e.g., spot + on-demand).  

---

### **10.7 ELB/ALB Configuration**  
**ALB (Application Load Balancer) - Most Common for Web Apps**  
```hcl
# 1. Security Group for ALB
resource "aws_security_group" "alb" {
  vpc_id = aws_vpc.main.id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# 2. ALB
resource "aws_lb" "web" {
  name               = "web-alb"
  internal           = false # false = internet-facing
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [aws_subnet.public.id, aws_subnet.public2.id] # Must be in different AZs
}

# 3. Target Group (Backend Instances)
resource "aws_lb_target_group" "web" {
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  target_type = "instance" # or "ip" (for Lambda) or "alb" (for ALB-to-ALB)
  health_check {
    path = "/health"
  }
}

# 4. Listener (Frontend)
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web.arn
  port              = "80"
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# 5. Attach ASG to Target Group
resource "aws_autoscaling_attachment" "asg_attachment" {
  autoscaling_group_name = aws_autoscaling_group.web.id
  lb_target_group_arn    = aws_lb_target_group.web.arn
}
```
- **ALB vs NLB:** Use ALB for HTTP/HTTPS (Layer 7), NLB for TCP/UDP (Layer 4) or static IPs.  
- **HTTPS Setup:**  
  ```hcl
  resource "aws_lb_listener" "https" {
    certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/..." # From ACM
    ssl_policy      = "ELBSecurityPolicy-TLS13-1-2-2021-06"
    protocol        = "HTTPS"
    ...
  }
  ```
- **Sticky Sessions:** Enable in `aws_lb_target_group` with `stickiness_enabled = true`.  
- **WAF Integration:** Attach Web ACL via `aws_wafv2_web_acl_association`.  

**Critical Gotchas:**  
- ALB **must** have at least 2 subnets in different AZs.  
- Health checks must pass before instances receive traffic.  
- Always use ALB in front of ASG (never expose EC2 directly).  

---

### **10.8 RDS (PostgreSQL, MySQL)**  
**Managed Relational Database Service**  
```hcl
# 1. DB Subnet Group (Must span private subnets)
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet"
  subnet_ids = [aws_subnet.private.id, aws_subnet.private2.id]
}

# 2. Security Group for RDS
resource "aws_security_group" "db" {
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id] # Only allow web SG
  }
}

# 3. RDS Instance (PostgreSQL)
resource "aws_db_instance" "postgres" {
  identifier           = "main-postgres"
  engine               = "postgres"
  engine_version       = "14.7"
  instance_class       = "db.t4g.micro"
  allocated_storage    = 20
  storage_type         = "gp3"
  db_subnet_group_name = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  username             = "admin"
  password             = var.db_password # Use SSM Parameter Store or Vault!
  publicly_accessible  = false # Must be false for multi-AZ
  skip_final_snapshot  = true # ❌ NEVER IN PROD! Set to false + final_snapshot_identifier
  multi_az             = true # For HA/failover
  backup_retention_period = 7 # Days
  maintenance_window   = "sun:05:00-sun:06:00"
  apply_immediately    = true # Caution: Causes downtime if false

  # Enhanced Monitoring (CloudWatch)
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
}
```
- **MySQL Example:** `engine = "mysql"`, `engine_version = "8.0.32"`.  
- **Storage Types:** `gp3` (default, cost-effective), `io1` (high IOPS).  
- **Never set `skip_final_snapshot = true` in production!** Always take a final snapshot.  
- **Parameter Groups:** Customize DB settings:  
  ```hcl
  parameter_group_name = "custom-postgres14"
  ```
- **Secrets Management:** **NEVER hardcode passwords.** Use:  
  ```hcl
  password = data.aws_ssm_parameter.db_password.value
  ```

**Critical Considerations:**  
- **Multi-AZ:** Essential for production (synchronous standby replica).  
- **Backup Window:** Set outside peak hours.  
- **Failover:** Takes 60-120 seconds (test your app's resilience!).  
- **Aurora:** Use `aws_rds_cluster` + `aws_rds_cluster_instance` for MySQL/PostgreSQL Aurora.  

---

### **10.9 S3 Buckets & Versioning**  
**Simple Storage Service (Object Storage)**  
```hcl
resource "aws_s3_bucket" "app_data" {
  bucket = "my-unique-bucket-name-12345" # Must be GLOBALLY unique!
  acl    = "private" # Legacy, use bucket policy instead

  # Block ALL public access (required for most compliance standards)
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true

  # Versioning - CRITICAL for accidental deletion recovery
  versioning {
    enabled = true
  }

  # Server-Side Encryption (SSE-S3)
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }

  # Lifecycle Rules (Archive/Expire objects)
  lifecycle_rule {
    id      = "archive-old-logs"
    prefix  = "logs/"
    enabled = true
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    expiration {
      days = 365
    }
  }
}
```
- **Bucket Naming Rules:**  
  - Lowercase letters, numbers, hyphens.  
  - Must be DNS-compliant (e.g., `my.bucket.name` invalid).  
  - **Globally unique** across *all* AWS accounts.  
- **`acl = "private"` is legacy.** Use bucket policies for permissions (see below).  
- **Versioning:** Once enabled, **cannot be disabled** (only suspended). Essential for ransomware protection.  
- **Encryption:** Always enable SSE-S3 (`AES256`) or SSE-KMS.  

#### **Bucket Policy (Replace ACLs)**  
```hcl
resource "aws_s3_bucket_policy" "allow_access" {
  bucket = aws_s3_bucket.app_data.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action   = "s3:GetObject"
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.app_data.arn}/*"
        Principal = {
          AWS = "arn:aws:iam::123456789012:root" # Account ID
        }
      }
    ]
  })
}
```
- **Deny Public Access:** Always set `block_public_* = true` + policy denying public access.  
- **Static Website Hosting:**  
  ```hcl
  website {
    index_document = "index.html"
    error_document = "error.html"
  }
  ```

---

### **10.10 IAM Roles, Policies, and Users**  
**Identity and Access Management (Least Privilege Principle)**  

#### **IAM Policy (JSON Document)**  
```hcl
data "aws_iam_policy_document" "s3_access" {
  statement {
    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]
    resources = [
      aws_s3_bucket.app_data.arn,
      "${aws_s3_bucket.app_data.arn}/*"
    ]
  }
}

resource "aws_iam_policy" "s3_access" {
  name        = "S3AccessPolicy"
  description = "Allows read access to app data bucket"
  policy      = data.aws_iam_policy_document.s3_access.json
}
```
- **`aws_iam_policy_document`** generates valid JSON (avoids syntax errors).  
- **Always scope resources** (`arn:aws:s3:::my-bucket/*`), never use `*`.  

#### **IAM Role (For EC2/Lambda)**  
```hcl
resource "aws_iam_role" "ec2_role" {
  name = "EC2Role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = { Service = "ec2.amazonaws.com" }
      }
    ]
  })
}

# Attach policy to role
resource "aws_iam_role_policy_attachment" "s3_attach" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = aws_iam_policy.s3_access.arn
}

# Instance Profile (Required for EC2)
resource "aws_iam_instance_profile" "web" {
  name = "WebInstanceProfile"
  role = aws_iam_role.ec2_role.name
}
```
- **`assume_role_policy`**: Defines *who* can assume the role (EC2, Lambda, etc.).  
- **Instance Profile:** Special container for EC2 roles (must attach to EC2 instance).  

#### **IAM User (Avoid for Automation!)**  
```hcl
resource "aws_iam_user" "deployer" {
  name = "deployer"
}

resource "aws_iam_access_key" "deployer_key" {
  user = aws_iam_user.deployer.name
}

# Output ONLY the ID (secret key is sensitive!)
output "deployer_access_key" {
  value     = aws_iam_access_key.deployer_key.id
  sensitive = true
}
```
- **Never use IAM users for applications/services.** Use IAM roles.  
- **Rotate keys frequently** (use `aws_iam_access_key` resource).  

**Golden Rules:**  
1. **Roles > Users** for services (EC2, Lambda).  
2. **Policies via `aws_iam_policy_document`** (not raw JSON strings).  
3. **Least privilege:** Start with no permissions, add only what's needed.  
4. **Enable IAM Access Analyzer** to detect unintended resource access.  

---

### **10.11 Lambda Functions & API Gateway**  
#### **Lambda Function**  
```hcl
# 1. IAM Role for Lambda
resource "aws_iam_role" "lambda" {
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })

  # Managed policy for basic logging
  managed_policy_arns = ["arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"]
}

# 2. Lambda Function
resource "aws_lambda_function" "hello" {
  function_name = "hello-world"
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  filename      = "lambda.zip" # Path to .zip file

  # Environment variables
  environment {
    variables = {
      TABLE_NAME = aws_dynamodb_table.users.name
    }
  }

  # Allow API Gateway to invoke
  depends_on = [aws_lambda_permission.apigw]
}

# 3. Grant API Gateway Permission to Invoke
resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.hello.arn
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.main.execution_arn}/*/*/*"
}
```
- **Packaging:** Use `archive_file` to zip code:  
  ```hcl
  data "archive_file" "lambda_zip" {
    type        = "zip"
    source_file = "lambda.js"
    output_path = "lambda.zip"
  }
  filename = data.archive_file.lambda_zip.output_path
  ```
- **Concurrency:** Set `reserved_concurrent_executions` to avoid throttling.  

#### **API Gateway (REST API)**  
```hcl
# 1. REST API
resource "aws_api_gateway_rest_api" "main" {
  name = "MyAPI"
}

# 2. Resource & Method
resource "aws_api_gateway_resource" "users" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  parent_id   = aws_api_gateway_rest_api.main.root_resource_id
  path_part   = "users"
}

resource "aws_api_gateway_method" "get_users" {
  rest_api_id   = aws_api_gateway_rest_api.main.id
  resource_id   = aws_api_gateway_resource.users.id
  http_method   = "GET"
  authorization = "NONE"
  api_key_required = false
}

# 3. Integration with Lambda
resource "aws_api_gateway_integration" "users_lambda" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  resource_id = aws_api_gateway_resource.users.id
  http_method = aws_api_gateway_method.get_users.http_method
  integration_http_method = "POST"
  type                      = "AWS_PROXY" # Simplifies mapping
  uri                       = aws_lambda_function.hello.invoke_arn
}

# 4. Deploy API
resource "aws_api_gateway_deployment" "deploy" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  stage_name  = "prod"
  depends_on = [
    aws_api_gateway_method.get_users,
    aws_api_gateway_integration.users_lambda
  ]
}
```
- **`AWS_PROXY` Integration:** Avoids manual request/response mapping (recommended).  
- **Deployment Trigger:** Changes to `aws_api_gateway_deployment` require new `stage_name` or `triggers`.  
- **CORS:** Configure via `aws_api_gateway_method_response` + `aws_api_gateway_method_integration`.  

**Critical Notes:**  
- **Lambda Timeout:** Default is 3 seconds – increase for slow operations.  
- **API Gateway Caching:** Enable for high-traffic endpoints (reduces Lambda invocations).  
- **Custom Domains:** Use `aws_api_gateway_domain_name` + Route 53 (see 10.12).  

---

### **10.12 Route 53 & DNS Management**  
**AWS DNS Service**  

#### **Public Hosted Zone**  
```hcl
resource "aws_route53_zone" "main" {
  name = "example.com"
}
```

#### **Record Sets**  
```hcl
# A Record for ALB
resource "aws_route53_record" "web" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "web.example.com"
  type    = "A"
  alias {
    name                   = aws_lb.web.dns_name
    zone_id                = aws_lb.web.zone_id
    evaluate_target_health = true
  }
}

# CNAME for S3 Static Site
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "CNAME"
  ttl     = 300
  records = [aws_s3_bucket.website.website_endpoint]
}

# TXT Record for Verification
resource "aws_route53_record" "spf" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "TXT"
  ttl     = 300
  records = ["\"v=spf1 include:_spf.google.com ~all\""]
}
```
- **ALB Alias:** Use `alias` block (not `records`) for ALB/CloudFront – avoids DNS lookup latency.  
- **S3 Static Site:** Requires bucket name = record name (e.g., `www.example.com` bucket).  
- **TTL:** Set low TTL (e.g., 300) for records that change frequently.  

#### **Private Hosted Zone (VPC Internal DNS)**  
```hcl
resource "aws_route53_zone" "internal" {
  name = "internal.example.com"
  vpc {
    vpc_id = aws_vpc.main.id
  }
}

resource "aws_route53_record" "db" {
  zone_id = aws_route53_zone.internal.zone_id
  name    = "db.internal.example.com"
  type    = "CNAME"
  ttl     = 300
  records = [aws_db_instance.postgres.address]
}
```
- **Use Case:** Internal service discovery (e.g., `db.internal.example.com` resolves to RDS).  
- **VPC Association:** Must specify VPC(s) in `aws_route53_zone`.  

**Gotchas:**  
- **SOA/NS Records:** Auto-created – don’t manage them.  
- **Delegation Sets:** For multi-account DNS (advanced).  
- **Health Checks:** Integrate with `aws_route53_health_check` for failover routing.  

---

### **10.13 CloudFront & S3 Static Website**  
**Content Delivery Network + Static Site**  

#### **S3 Bucket for Static Site**  
```hcl
resource "aws_s3_bucket" "website" {
  bucket = "www.example.com"
  acl    = "public-read" # Required for static site

  website {
    index_document = "index.html"
    error_document = "error.html"
  }

  # Block public access EXCEPT for website
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# Bucket policy to allow public read
resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "s3:GetObject"
      Effect    = "Allow"
      Resource  = "${aws_s3_bucket.website.arn}/*"
      Principal = "*"
    }]
  })
}
```

#### **CloudFront Distribution**  
```hcl
resource "aws_cloudfront_distribution" "www" {
  origin {
    domain_name = aws_s3_bucket.website.website_endpoint
    origin_id   = "S3-www.example.com"

    # Required for S3 website endpoint
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "http-only" # S3 website uses HTTP
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  # Viewer Protocol Policy (HTTPS enforcement)
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-www.example.com"
    viewer_protocol_policy = "redirect-to-https" # Forces HTTPS
    compress         = true

    # Cache TTLs
    min_ttl    = 0
    default_ttl = 86400
    max_ttl     = 31536000

    # Security headers (via Lambda@Edge)
    # lambda_function_association { ... }
  }

  # SSL Certificate (from ACM)
  viewer_certificate {
    acm_certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/..."
    ssl_support_method  = "sni-only"
  }

  # Custom Domain
  aliases = ["www.example.com"]
}
```
- **S3 Origin:** Must use `website_endpoint` (not `bucket_regional_domain_name`).  
- **Origin Protocol:** `http-only` for S3 website endpoints (they don’t support HTTPS).  
- **HTTPS Enforcement:** `viewer_protocol_policy = "redirect-to-https"` is critical.  
- **ACM Certificate:** Must be in `us-east-1` for CloudFront.  

#### **Route 53 Integration**  
```hcl
resource "aws_route53_record" "cf" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "A"
  alias {
    name                   = aws_cloudfront_distribution.www.domain_name
    zone_id                = aws_cloudfront_distribution.www.hosted_zone_id
    evaluate_target_health = false
  }
}
```
- **CloudFront Zone ID:** Always `Z2FDTNDATAQYW2` (hardcoded in AWS).  

**Performance Tips:**  
- Enable **Compression** (`compress = true`).  
- Set long `max_ttl` for static assets (use cache-busting filenames).  
- Use **Lambda@Edge** for security headers (CSP, HSTS) or redirects.  

---

### **10.14 Using AWS Data Sources (e.g., aws_ami, aws_availability_zones)**  
**Dynamic Data from AWS API (Avoid Hardcoding!)**  

#### **Why Data Sources?**  
- Get latest AMI IDs without hardcoding.  
- Discover AZs in a region (for HA deployments).  
- Reference resources created *outside* Terraform.  

#### **Key Data Sources**  
```hcl
# 1. Latest Ubuntu AMI (us-east-1)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Usage: ami = data.aws_ami.ubuntu.id

# 2. Availability Zones in Region
data "aws_availability_zones" "available" {
  state = "available" # Only return AZs available for your account
}

# Usage: 
#   availability_zone = data.aws_availability_zones.available.names[0]
#   subnet_ids = data.aws_availability_zones.available.names

# 3. Current AWS Account/Region
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Usage: 
#   account_id = data.aws_caller_identity.current.account_id
#   region     = data.aws_region.current.name

# 4. SSM Parameter (e.g., DB password)
data "aws_ssm_parameter" "db_password" {
  name = "/prod/db/password"
  with_decryption = true
}

# Usage: password = data.aws_ssm_parameter.db_password.value
```

**Critical Best Practices:**  
- **Always use `most_recent = true` for AMIs** (with strict filters).  
- **Filter aggressively** to avoid unexpected matches (e.g., by `name`, `virtualization-type`).  
- **SSM Parameters > Secrets Manager** for Terraform (simpler permissions).  
- **Data sources run during `terraform plan`** – ensure IAM permissions for `data` blocks.  

**Why Avoid Hardcoding?**  
- AMI IDs change frequently (breaks deployments).  
- AZ names vary by account (e.g., `us-east-1a` vs `us-east-1b` per account).  
- Secrets in state files = security risk.  

---

### **Final Pro Tips for Production**  
1. **State Management:**  
   - **ALWAYS** use remote state (S3 + DynamoDB locking).  
   - Enable state versioning in S3 bucket.  
2. **Security:**  
   - Scan Terraform configs with **Checkov** or **tfsec**.  
   - Restrict IAM permissions for Terraform role to *only* necessary actions.  
3. **Modularize:** Break configs into modules (`vpc/`, `eks/`, `rds/`).  
4. **Variables & Outputs:**  
   - Use `sensitive = true` for secrets.  
   - Validate inputs with `validation` blocks.  
5. **Zero-Downtime Deployments:**  
   - Use `create_before_destroy` lifecycle for critical resources.  
   - Test ASG instance refreshes before production.  
