# 🚀 **Terraform: From Beginner to Expert – Full Course Table of Contents**

---

## **Module 1: Introduction to Infrastructure as Code (IaC)**
- 1.1 What is Infrastructure as Code (IaC)?
- 1.2 Benefits of IaC (Consistency, Versioning, Automation)
- 1.3 Declarative vs Imperative Models
- 1.4 Overview of IaC Tools (Terraform vs Ansible vs CloudFormation vs Pulumi)
- 1.5 Why Terraform? (Multi-cloud, Declarative, State Management)

---

## **Module 2: Terraform Fundamentals**
- 2.1 What is Terraform?
- 2.2 Terraform Architecture Overview
- 2.3 Terraform Workflow (Write → Plan → Apply → Destroy)
- 2.4 Installing Terraform (CLI)
- 2.5 Verifying Installation & Terraform Version
- 2.6 Basic Terraform Commands (`init`, `validate`, `plan`, `apply`, `destroy`)
- 2.7 Writing Your First Terraform Configuration (Hello World with AWS S3)

---

## **Module 3: Terraform Configuration Language (HCL)**
- 3.1 Introduction to HashiCorp Configuration Language (HCL)
- 3.2 Syntax Basics: Blocks, Arguments, Expressions
- 3.3 Terraform File Structure (.tf files)
- 3.4 Comments, Whitespace, and Formatting
- 3.5 Expressions: Literals, Variables, Functions
- 3.6 Interpolation Syntax `${...}` and `...` (v0.12+)
- 3.7 Conditional Expressions (Ternary)
- 3.8 Collections: Lists, Maps, Sets
- 3.9 Loops and Iteration (`for_each`, `count`)
- 3.10 Dynamic Blocks

---

## **Module 4: Core Terraform Concepts**
- 4.1 Providers
  - What is a Provider?
  - Installing and Configuring Providers
  - Multiple Providers & Aliases
  - Provider Version Constraints
- 4.2 Resources
  - Defining Resources
  - Resource Dependencies (Implicit & Explicit)
  - Lifecycle Rules (`create_before_destroy`, `prevent_destroy`, `ignore_changes`)
- 4.3 Data Sources
  - Reading External Data
  - Using Data Sources with Resources
- 4.4 Outputs
  - Defining Outputs
  - Output Sensitivity (`sensitive = true`)
  - Output Formatting
- 4.5 Variables
  - Input Variables (Types, Descriptions, Defaults)
  - Variable Validation Rules
  - Variable Files (`terraform.tfvars`, `*.auto.tfvars`)
  - Variable Precedence (CLI, File, Environment)
- 4.6 Locals
  - Using Locals for Reusable Values
  - Difference Between Locals and Variables

---

## **Module 5: Managing Terraform State**
- 5.1 What is Terraform State?
- 5.2 Purpose of State (Mapping, Metadata, Performance)
- 5.3 `terraform.tfstate` File (Local Backend)
- 5.4 State Locking and Concurrency
- 5.5 Remote State (Backend Configuration)
  - AWS S3 + DynamoDB
  - Azure Storage
  - Google Cloud Storage
  - Terraform Cloud
- 5.6 Migrating State to Remote Backend
- 5.7 Importing Existing Resources into State
- 5.8 State Commands (`terraform state list`, `show`, `mv`, `rm`, `import`)
- 5.9 State Isolation Strategies (Workspaces vs Directories)
- 5.10 Tainting Resources (`terraform taint` / `terraform untaint`)

---

## **Module 6: Terraform Modules**
- 6.1 What are Modules?
- 6.2 Module Structure and Layout
- 6.3 Creating a Local Module
- 6.4 Calling Modules (Source, Version, Inputs)
- 6.5 Module Inputs and Outputs
- 6.6 Public vs Private Modules
- 6.7 Module Composition and Reusability
- 6.8 Using Remote Modules (GitHub, Terraform Registry)
- 6.9 Publishing Modules to Terraform Registry
- 6.10 Module Versioning (SemVer)
- 6.11 Module Best Practices (Input validation, documentation, examples)

---

## **Module 7: Terraform Workspaces**
- 7.1 What are Workspaces?
- 7.2 When to Use Workspaces
- 7.3 Managing Workspaces (`new`, `list`, `select`, `delete`)
- 7.4 Workspaces vs Directory-per-Environment Pattern
- 7.5 Limitations of Workspaces

---

## **Module 8: Managing Multiple Environments**
- 8.1 Environment Patterns: Dev, Staging, Prod
- 8.2 Strategy 1: Directory-per-Environment
- 8.3 Strategy 2: Workspaces (with caveats)
- 8.4 Strategy 3: Terragrunt (Bonus)
- 8.5 Using `tfvars` per Environment
- 8.6 Environment-Specific Variables and Overrides
- 8.7 Module Reuse Across Environments

---

## **Module 9: Terraform Cloud & Terraform Enterprise**
- 9.1 Introduction to Terraform Cloud (Free & Paid Tiers)
- 9.2 Setting Up a Workspace
- 9.3 VCS Integration (GitHub, GitLab, Bitbucket)
- 9.4 Remote State & Plan/Apply in Cloud
- 9.5 Team & Policy Management
- 9.6 Sentinel Policy-as-Code (Enterprise)
- 9.7 Private Module Registry
- 9.8 Running Terraform in CI/CD via Cloud
- 9.9 Agent Pools (Enterprise)

---

## **Module 10: Terraform with AWS (Deep Dive)**
- 10.1 AWS Provider Configuration
- 10.2 Authentication (IAM Roles, AWS CLI, Credentials File)
- 10.3 Creating VPC, Subnets, Route Tables
- 10.4 EC2 Instances & Key Pairs
- 10.5 Security Groups & Network ACLs
- 10.6 Auto Scaling Groups & Launch Templates
- 10.7 ELB/ALB Configuration
- 10.8 RDS (PostgreSQL, MySQL)
- 10.9 S3 Buckets & Versioning
- 10.10 IAM Roles, Policies, and Users
- 10.11 Lambda Functions & API Gateway
- 10.12 Route 53 & DNS Management
- 10.13 CloudFront & S3 Static Website
- 10.14 Using AWS Data Sources (e.g., `aws_ami`, `aws_availability_zones`)

---

## **Module 11: Terraform with Azure (Deep Dive)**
- 11.1 Azure Provider Setup
- 11.2 Authentication (Service Principal, CLI, Managed Identity)
- 11.3 Resource Groups, VNets, Subnets
- 11.4 Virtual Machines & Availability Sets
- 11.5 Azure Load Balancer & Application Gateway
- 11.6 Azure SQL & Storage Accounts
- 11.7 Azure AD Applications & Service Principals
- 11.8 Azure Blob Storage & CDN
- 11.9 Azure Functions & Logic Apps
- 11.10 Using AzureRM Data Sources

---

## **Module 12: Terraform with Google Cloud (Deep Dive)**
- 12.1 GCP Provider Configuration
- 12.2 Authentication (Service Account Key, Workload Identity)
- 12.3 VPC, Subnetworks, Firewalls
- 12.4 Compute Engine (VMs)
- 12.5 Cloud SQL & Cloud Storage
- 12.6 Kubernetes Engine (GKE)
- 12.7 Cloud Functions & Cloud Run
- 12.8 Using Google Data Sources

---

## **Module 13: Advanced Terraform Patterns**
- 13.1 Dynamic Blocks for Complex Nested Arguments
- 13.2 `for_each` vs `count` (When to Use Which)
- 13.3 Conditional Resource Creation
- 13.4 Meta-Arguments (`depends_on`, `lifecycle`, `providers`)
- 13.5 Handling Large Configurations (Module Decomposition)
- 13.6 Cross-Module Dependencies
- 13.7 Using `null_resource` and `local-exec` / `remote-exec` (Use Cases & Warnings)
- 13.8 Provisioners (Best Practices & Anti-Patterns)
- 13.9 Terraform Expressions & Functions (Advanced)
- 13.10 Working with JSON & YAML (Using `jsonencode`, `yamlencode`)

---

## **Module 14: Terraform Security Best Practices**
- 14.1 Securing Terraform State (Encryption at Rest, Access Control)
- 14.2 Managing Secrets
  - Avoiding Hardcoded Secrets
  - Using Vault with Terraform
  - AWS Secrets Manager / SSM Parameter Store
  - Azure Key Vault
  - Google Secret Manager
- 14.3 Principle of Least Privilege (IAM Roles)
- 14.4 Audit Logging & Drift Detection
- 14.5 Terraform Sentinel Policies (Compliance & Governance)
- 14.6 Static Code Analysis (Checkov, tfsec, tflint)
- 14.7 Preventing Accidental Destruction (`prevent_destroy`)
- 14.8 Environment Isolation & Network Security

---

## **Module 15: Testing Terraform Code**
- 15.1 Why Test Infrastructure Code?
- 15.2 Unit Testing with `tflint` and `tfsec`
- 15.3 Integration Testing with Terratest (Go-based)
- 15.4 Testing Modules in Isolation
- 15.5 Mocking Cloud APIs (Optional)
- 15.6 Testing Outputs and Resource Attributes
- 15.7 End-to-End Testing in CI/CD

---

## **Module 16: CI/CD with Terraform**
- 16.1 CI/CD Pipeline Overview
- 16.2 GitHub Actions for Terraform
- 16.3 GitLab CI/CD with Terraform
- 16.4 Jenkins Pipeline for Terraform
- 16.5 Pull Request Workflow with `plan`
- 16.6 Automated `apply` with Approval Gates
- 16.7 Environment Promotion (Dev → Staging → Prod)
- 16.8 Using Atlantis for Automated Terraform in Git
- 16.9 Integrating with Argo CD / Flux (GitOps)

---

## **Module 17: Terraform and Kubernetes**
- 17.1 Kubernetes Provider Configuration
- 17.2 Deploying Resources (Pods, Services, Deployments)
- 17.3 Managing Namespaces, Secrets, ConfigMaps
- 17.4 Helm Provider: Installing Helm Charts
- 17.5 Managing CRDs and Operators
- 17.6 Limitations of Terraform for Kubernetes (vs Helm/Kustomize)

---

## **Module 18: Terraform Debugging & Troubleshooting**
- 18.1 Reading Terraform Plan Output
- 18.2 Debugging State Drift
- 18.3 Using `TF_LOG` for Debug Logs
- 18.4 Handling State Conflicts & Locking Issues
- 18.5 Fixing `terraform apply` Failures
- 18.6 Recovering from Corrupted State
- 18.7 Using `refresh_only` and `replace_triggered_by`
- 18.8 Common Errors & Solutions

---

## **Module 19: Real-World Project: Multi-Tier Application**
- 19.1 Project Scope: VPC, Web, App, DB Tier
- 19.2 Module Design (Network, Compute, Database)
- 19.3 Environment Management (Dev/Staging/Prod)
- 19.4 CI/CD Pipeline Setup
- 19.5 Security Hardening (Security Groups, IAM)
- 19.6 Monitoring & Alerting (Optional: Integrate with CloudWatch)
- 19.7 Documentation & READMEs

---

## **Module 20: Terraform Best Practices & Anti-Patterns**
- 20.1 Directory Structure Best Practices
- 20.2 Naming Conventions
- 20.3 Version Pinning (Providers, Modules)
- 20.4 Avoiding Large Monolithic Configurations
- 20.5 Using `for_each` Over `count` When Possible
- 20.6 Minimizing Use of Provisioners
- 20.7 Immutable Infrastructure Principles
- 20.8 Handling Destructive Changes Safely
- 20.9 Documentation in Code (Comments, READMEs)

---

## **Module 21: Advanced Topics & Ecosystem Tools**
- 21.1 Terragrunt (Simplify Terraform at Scale)
- 21.2 Atlantis (Automated Terraform via Git)
- 21.3 OpenTofu (Open-Source Fork of Terraform)
- 21.4 Terraformer (Import Existing Infrastructure)
- 21.5 Crossplane (Kubernetes-native IaC)
- 21.6 Packer + Terraform (Golden Images)
- 21.7 Ansible + Terraform (Hybrid Approaches)

---

## **Module 22: Terraform Upgrades & Migration**
- 22.1 Terraform Version Management (tfenv)
- 22.2 Upgrading Terraform Versions
- 22.3 Upgrading Provider Versions
- 22.4 Migrating from `terraform 0.11` to `0.12+`
- 22.5 Migrating from `0.14` to `1.x`
- 22.6 Breaking Changes & How to Handle Them

---

## **Module 23: Community, Resources & Certification**
- 23.1 Terraform Documentation & HashiCorp Learn
- 23.2 Terraform Registry (Public & Private)
- 23.3 GitHub Repositories & Open Source Modules
- 23.4 Terraform Certification (HashiCorp Certified: Terraform Associate)
  - Exam Topics
  - Study Guide
  - Practice Questions
- 23.5 Joining the Terraform Community (Forums, Slack, Reddit)

---

## **Module 24: Final Capstone Project**
- 24.1 Design a Scalable, Secure, Multi-Cloud Architecture
- 24.2 Implement CI/CD with GitHub Actions & Terraform Cloud
- 24.3 Enforce Policy with Sentinel or Open Policy Agent
- 24.4 Write Comprehensive Tests
- 24.5 Document Architecture & Decisions (ADR)
- 24.6 Present & Review

---
