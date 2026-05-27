# Terraform: Complete Guide — Basics to Advanced

> Infrastructure as Code (IaC) with HashiCorp Terraform

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts](#3-core-concepts)
4. [HCL Syntax](#4-hcl-syntax)
5. [Providers](#5-providers)
6. [Resources](#6-resources)
7. [Variables](#7-variables)
8. [Outputs](#8-outputs)
9. [Local Values](#9-local-values)
10. [Data Sources](#10-data-sources)
11. [State Management](#11-state-management)
12. [Modules](#12-modules)
13. [Functions & Expressions](#13-functions--expressions)
14. [Meta-Arguments](#14-meta-arguments)
15. [Provisioners](#15-provisioners)
16. [Workspaces](#16-workspaces)
17. [Remote Backends](#17-remote-backends)
18. [Terraform Cloud & Enterprise](#18-terraform-cloud--enterprise)
19. [Import & Move](#19-import--move)
20. [Testing](#20-testing)
21. [Security Best Practices](#21-security-best-practices)
22. [CI/CD Integration](#22-cicd-integration)
23. [Advanced Patterns](#23-advanced-patterns)
24. [Troubleshooting](#24-troubleshooting)

---

## 1. Introduction

**Terraform** is an open-source Infrastructure as Code tool by HashiCorp that lets you define, provision, and manage cloud and on-premises infrastructure using a declarative configuration language called **HCL (HashiCorp Configuration Language)**.

### Why Terraform?

- **Multi-cloud**: Supports AWS, Azure, GCP, Kubernetes, and 3000+ providers
- **Declarative**: Describe *what* you want, not *how* to get there
- **Idempotent**: Running `apply` multiple times produces the same result
- **State tracking**: Knows the current state of your infrastructure
- **Plan before apply**: Preview changes before they happen
- **Version controlled**: Infrastructure config lives in Git like code

### Terraform Workflow

```
Write → Init → Plan → Apply → Destroy
```

```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Write  │───▶│   Init   │───▶│   Plan   │───▶│  Apply   │
│  .tf    │    │ download │    │ preview  │    │ execute  │
│  files  │    │providers │    │ changes  │    │ changes  │
└─────────┘    └──────────┘    └──────────┘    └──────────┘
```

---

## 2. Installation & Setup

### Install Terraform

**macOS (Homebrew):**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

**Ubuntu/Debian:**
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

**Windows (Chocolatey):**
```powershell
choco install terraform
```

**Verify Installation:**
```bash
terraform version
# Terraform v1.9.x
```

### Install tfenv (Version Manager)

```bash
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

tfenv install 1.9.0
tfenv use 1.9.0
```

### Project Structure (Best Practice)

```
my-infra/
├── main.tf           # Primary resources
├── variables.tf      # Input variable declarations
├── outputs.tf        # Output value declarations
├── providers.tf      # Provider configurations
├── terraform.tfvars  # Variable values (gitignore secrets!)
├── versions.tf       # Required versions
└── modules/
    ├── networking/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── compute/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## 3. Core Concepts

### Declarative vs Imperative

```hcl
# Declarative (Terraform): describe the desired end state
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
# Terraform figures out HOW to create/update/delete it
```

### The Terraform State

Terraform maintains a **state file** (`terraform.tfstate`) that maps your config to real-world infrastructure. It:
- Tracks resource IDs
- Detects configuration drift
- Enables planning (diff between desired & actual)

### Resource Lifecycle

```
Config written → terraform plan (shows diff) → terraform apply (executes)
                                                     │
                                         ┌───────────┴──────────┐
                                     Create new            Update/Delete
                                     resource              existing
```

---

## 4. HCL Syntax

HCL (HashiCorp Configuration Language) is human-readable and machine-friendly.

### Basic Block Structure

```hcl
<BLOCK_TYPE> "<BLOCK_LABEL>" "<BLOCK_LABEL>" {
  # Arguments
  <IDENTIFIER> = <EXPRESSION>
}
```

### Data Types

```hcl
# String
name = "my-server"

# Number
count = 3

# Boolean
enabled = true

# List (ordered, same type)
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

# Set (unordered, unique)
security_groups = toset(["sg-001", "sg-002"])

# Map (key-value pairs)
tags = {
  Environment = "production"
  Team        = "platform"
  CostCenter  = "12345"
}

# Object (structured, mixed types)
server_config = {
  name    = "web-01"
  cpu     = 4
  enabled = true
}

# Tuple (ordered, mixed types)
mixed = ["hello", 42, true]

# Null
value = null
```

### String Interpolation & Templates

```hcl
variable "env" {
  default = "prod"
}

# Interpolation
name = "my-app-${var.env}"

# Multi-line heredoc
user_data = <<-EOT
  #!/bin/bash
  echo "Hello from ${var.env}" > /tmp/hello.txt
  yum update -y
  yum install -y nginx
EOT

# Indented heredoc (strips leading whitespace)
description = <<-EOT
  This instance runs in ${var.env}.
  It was created by Terraform.
EOT
```

### Comments

```hcl
# Single-line comment

// Also a single-line comment

/*
  Multi-line
  comment block
*/
```

### Conditional Expressions

```hcl
# Ternary: condition ? true_val : false_val
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"

# In resource
count = var.create_instance ? 1 : 0
```

---

## 5. Providers

Providers are plugins that interact with APIs (AWS, Azure, GCP, etc.).

### Configuring a Provider

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # >= 5.0.0, < 6.0.0
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

# providers.tf
provider "aws" {
  region  = "us-east-1"
  profile = "my-aws-profile"  # uses ~/.aws/credentials

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
    }
  }
}
```

### Multiple Provider Instances (Alias)

```hcl
# Primary region
provider "aws" {
  region = "us-east-1"
}

# Secondary region with alias
provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
}

# Using aliased provider
resource "aws_s3_bucket" "eu_bucket" {
  provider = aws.eu
  bucket   = "my-eu-bucket"
}
```

### Common Providers

```hcl
# Azure
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

# Google Cloud
provider "google" {
  project = "my-gcp-project"
  region  = "us-central1"
}

# Kubernetes
provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}
```

---

## 6. Resources

Resources are the fundamental building blocks — they represent infrastructure objects.

### Basic Resource

```hcl
resource "<PROVIDER>_<TYPE>" "<NAME>" {
  # configuration arguments
}
```

### AWS EC2 Instance

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id  # reference another resource

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }

  tags = {
    Name = "web-server"
  }
}
```

### AWS S3 Bucket

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-unique-bucket-name-12345"
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

### Resource References

```hcl
# Reference syntax: <resource_type>.<resource_name>.<attribute>
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id  # references the VPC resource

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "app" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web.id]  # use the SG id
}
```

### Resource Dependencies

```hcl
# Implicit dependency (via reference) — Terraform auto-detects order
resource "aws_instance" "app" {
  subnet_id = aws_subnet.main.id  # Terraform knows to create subnet first
}

# Explicit dependency (use depends_on when no direct reference)
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  depends_on = [
    aws_iam_role_policy_attachment.app_policy,
    aws_s3_bucket.config
  ]
}
```

---

## 7. Variables

Variables make your configs reusable and configurable.

### Variable Declaration

```hcl
# variables.tf

# Simple string
variable "region" {
  type        = string
  description = "AWS region to deploy resources"
  default     = "us-east-1"
}

# Number with validation
variable "instance_count" {
  type        = number
  description = "Number of EC2 instances"
  default     = 2

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

# Boolean
variable "enable_monitoring" {
  type    = bool
  default = false
}

# List of strings
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Map
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "t3.micro"
    stg  = "t3.small"
    prod = "t3.large"
  }
}

# Object
variable "db_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    multi_az       = bool
  })
  default = {
    engine         = "postgres"
    engine_version = "15.3"
    instance_class = "db.t3.medium"
    multi_az       = false
  }
}

# Sensitive variable (value hidden in logs/output)
variable "db_password" {
  type      = string
  sensitive = true
}
```

### Using Variables

```hcl
# main.tf
resource "aws_instance" "app" {
  instance_type = var.instance_types[var.environment]
  count         = var.instance_count
}
```

### Providing Variable Values

**1. terraform.tfvars file:**
```hcl
# terraform.tfvars
region         = "us-west-2"
instance_count = 3
environment    = "prod"
db_password    = "super-secret-password"  # ⚠️ gitignore this file
```

**2. Named .tfvars files:**
```bash
terraform apply -var-file="prod.tfvars"
terraform apply -var-file="secrets.tfvars" -var-file="prod.tfvars"
```

**3. Command-line flags:**
```bash
terraform apply -var="instance_count=5" -var="region=eu-west-1"
```

**4. Environment variables (TF_VAR_ prefix):**
```bash
export TF_VAR_db_password="my-secret-pass"
export TF_VAR_region="us-east-1"
terraform apply
```

### Variable Precedence (lowest → highest)

```
defaults → terraform.tfvars → *.auto.tfvars → -var-file → -var → TF_VAR_*
```

---

## 8. Outputs

Outputs expose values from your configuration (useful for modules, CI/CD, and debugging).

```hcl
# outputs.tf

output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "load_balancer_dns" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.main.dns_name
}

output "db_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
  sensitive   = true  # hidden in CLI output, but stored in state
}

# Outputting a list
output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}

# Outputting a map
output "instance_info" {
  value = {
    id         = aws_instance.web.id
    public_ip  = aws_instance.web.public_ip
    private_ip = aws_instance.web.private_ip
  }
}
```

### Using Output Values

```bash
# Show all outputs after apply
terraform output

# Get specific output
terraform output load_balancer_dns

# Get output as JSON (useful in scripts)
terraform output -json instance_info

# Pipe to jq
terraform output -json | jq '.load_balancer_dns.value'
```

---

## 9. Local Values

Locals are computed values within a module — like named expressions to avoid repetition.

```hcl
locals {
  # Computed values
  environment = var.env == "prod" ? "production" : "non-production"
  name_prefix = "${var.project}-${var.env}"

  # Common tags applied everywhere
  common_tags = {
    Project     = var.project
    Environment = var.env
    ManagedBy   = "Terraform"
    Owner       = "platform-team"
  }

  # Derived from other locals
  full_name = "${local.name_prefix}-web-server"

  # Complex expression
  enabled_azs = [
    for az in var.availability_zones :
    az if !contains(var.disabled_azs, az)
  ]
}

# Use locals
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  tags          = merge(local.common_tags, { Name = local.full_name })
}
```

---

## 10. Data Sources

Data sources fetch information from existing infrastructure (read-only).

```hcl
# Fetch the latest Amazon Linux 2023 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Use the fetched AMI
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id  # uses fetched ID
  instance_type = "t3.micro"
}
```

### Common Data Sources

```hcl
# Fetch existing VPC by tag
data "aws_vpc" "main" {
  tags = {
    Name = "main-vpc"
  }
}

# Fetch all subnets in the VPC
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  tags = {
    Tier = "private"
  }
}

# Current AWS account ID and region
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

# Read an IAM policy document
data "aws_iam_policy_document" "s3_access" {
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:PutObject"]
    resources = ["${aws_s3_bucket.data.arn}/*"]
  }
}

resource "aws_iam_policy" "s3_access" {
  name   = "s3-access-policy"
  policy = data.aws_iam_policy_document.s3_access.json
}

# External data source (run a script, get JSON back)
data "external" "git_version" {
  program = ["bash", "${path.module}/scripts/get_version.sh"]
}

output "app_version" {
  value = data.external.git_version.result["version"]
}
```

---

## 11. State Management

### State Commands

```bash
# List all resources in state
terraform state list

# Show details of a specific resource
terraform state show aws_instance.web

# Remove a resource from state (does NOT destroy infra)
terraform state rm aws_instance.old_server

# Move resource to a new address (rename)
terraform state mv aws_instance.web aws_instance.web_server

# Pull current remote state to stdout
terraform state pull

# Push local state to remote backend
terraform state push terraform.tfstate
```

### State Locking

When using remote backends, state is automatically locked during operations to prevent concurrent changes.

```bash
# If a lock gets stuck, force unlock (use with caution!)
terraform force-unlock <LOCK_ID>
```

### Refresh State

```bash
# Sync state with actual infrastructure (read-only)
terraform refresh

# Or use -refresh-only flag (safer, shows what would change)
terraform plan -refresh-only
terraform apply -refresh-only
```

### Importing Existing Infrastructure

```bash
# Import existing resource into Terraform state
terraform import aws_instance.web i-0123456789abcdef0

# For resources with complex IDs
terraform import aws_s3_bucket.data my-existing-bucket
terraform import aws_iam_role_policy_attachment.admin arn:aws:iam::123456789012:role/admin/arn:aws:iam::aws:policy/AdministratorAccess
```

---

## 12. Modules

Modules are reusable, self-contained packages of Terraform configurations.

### Creating a Module

```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
```

```hcl
# modules/vpc/variables.tf
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks for public subnets"
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks for private subnets"
}

variable "environment" {
  type = string
}
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.environment}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.environment}-igw" }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  map_public_ip_on_launch = true
  tags = { Name = "${var.environment}-public-${count.index + 1}" }
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

### Calling a Module

```hcl
# main.tf (root module)

# Local module
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
  environment          = var.environment
}

# Use module outputs
resource "aws_instance" "app" {
  subnet_id = module.vpc.public_subnet_ids[0]
}

# Terraform Registry module
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-eks-cluster"
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
}

# GitHub source
module "vpc" {
  source = "github.com/my-org/terraform-modules//vpc?ref=v2.0.0"
}
```

### Module Best Practices

```hcl
# Use for_each to instantiate multiple copies
module "regional_vpc" {
  for_each = toset(["us-east-1", "eu-west-1", "ap-southeast-1"])

  source = "./modules/vpc"

  providers = {
    aws = aws.regional[each.key]
  }

  vpc_cidr    = var.vpc_cidrs[each.key]
  environment = var.environment
}

# Access per-region outputs
output "all_vpc_ids" {
  value = { for region, mod in module.regional_vpc : region => mod.vpc_id }
}
```

---

## 13. Functions & Expressions

### String Functions

```hcl
locals {
  upper   = upper("hello")                     # "HELLO"
  lower   = lower("WORLD")                     # "world"
  trimmed = trimspace("  hello  ")             # "hello"
  joined  = join(", ", ["a", "b", "c"])        # "a, b, c"
  split   = split(",", "a,b,c")               # ["a", "b", "c"]
  replaced = replace("hello world", "world", "terraform") # "hello terraform"
  formatted = format("instance-%03d", 7)       # "instance-007"
  starts    = startswith("hello", "hel")       # true
  ends      = endswith("hello", "llo")         # true
}
```

### Numeric Functions

```hcl
locals {
  max_val  = max(3, 7, 2)      # 7
  min_val  = min(3, 7, 2)      # 2
  abs_val  = abs(-5)           # 5
  ceil_val = ceil(1.2)         # 2
  floor_val = floor(1.9)       # 1
  pow_val  = pow(2, 8)         # 256
}
```

### Collection Functions

```hcl
locals {
  list = ["a", "b", "c", "d"]
  map  = { foo = "bar", baz = "qux" }

  length_val  = length(local.list)              # 4
  keys_val    = keys(local.map)                 # ["baz", "foo"]
  values_val  = values(local.map)               # ["qux", "bar"]
  contains_val = contains(local.list, "b")      # true
  flatten_val = flatten([["a", "b"], ["c"]])    # ["a", "b", "c"]
  distinct_val = distinct(["a", "b", "a", "c"]) # ["a", "b", "c"]
  compact_val = compact(["a", "", "b", null])   # ["a", "b"]
  merged_map  = merge({ a = 1 }, { b = 2 })    # { a = 1, b = 2 }
  reversed    = reverse(["a", "b", "c"])        # ["c", "b", "a"]
  sorted      = sort(["c", "a", "b"])           # ["a", "b", "c"]
  sliced      = slice(["a","b","c","d"], 1, 3)  # ["b", "c"]
  zipped      = zipmap(["a","b"], [1, 2])       # { a=1, b=2 }
  lookup_val  = lookup(local.map, "foo", "default") # "bar"
}
```

### Type Conversion Functions

```hcl
locals {
  to_str    = tostring(42)           # "42"
  to_num    = tonumber("42")         # 42
  to_bool   = tobool("true")         # true
  to_list   = tolist(["a", "b"])
  to_set    = toset(["a", "b", "a"]) # removes duplicates
  to_map    = tomap({ key = "value" })
}
```

### Filesystem & Encoding Functions

```hcl
locals {
  # Read a file
  script = file("${path.module}/scripts/init.sh")

  # File content as base64
  cert_b64 = filebase64("${path.module}/certs/server.crt")

  # Template file
  user_data = templatefile("${path.module}/templates/userdata.tpl", {
    hostname    = "web-01"
    environment = var.env
  })

  # JSON encode/decode
  config_json = jsonencode({
    port    = 8080
    workers = 4
  })

  # YAML encode
  config_yaml = yamlencode({ key = "value" })

  # Base64
  encoded = base64encode("hello world")
  decoded = base64decode(local.encoded)
}
```

### For Expressions

```hcl
# List comprehension
variable "names" {
  default = ["alice", "bob", "carol"]
}

locals {
  # Transform list
  upper_names = [for name in var.names : upper(name)]
  # ["ALICE", "BOB", "CAROL"]

  # Filter list
  long_names = [for name in var.names : name if length(name) > 3]
  # ["alice", "carol"]

  # Map to map
  name_lengths = { for name in var.names : name => length(name) }
  # { alice = 5, bob = 3, carol = 5 }

  # Map values
  instance_ids = { for k, v in aws_instance.servers : k => v.id }
}
```

### Splat Expressions

```hcl
# Get all IDs from a list of resources
resource "aws_instance" "servers" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

output "all_ids" {
  # Splat operator [*]
  value = aws_instance.servers[*].id
  # Equivalent to: [for s in aws_instance.servers : s.id]
}

output "all_private_ips" {
  value = aws_instance.servers[*].private_ip
}
```

### Dynamic Blocks

```hcl
variable "ingress_rules" {
  default = [
    { port = 80,  protocol = "tcp", cidr = "0.0.0.0/0" },
    { port = 443, protocol = "tcp", cidr = "0.0.0.0/0" },
    { port = 22,  protocol = "tcp", cidr = "10.0.0.0/8" },
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = var.vpc_id

  # Dynamic block replaces repeated static blocks
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = [ingress.value.cidr]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 14. Meta-Arguments

Meta-arguments change the behavior of resource blocks.

### count

```hcl
# Create multiple identical resources
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"  # web-0, web-1, web-2
  }
}

# Reference by index
output "first_instance_ip" {
  value = aws_instance.web[0].public_ip
}

output "all_instance_ips" {
  value = aws_instance.web[*].public_ip
}
```

### for_each

```hcl
# Better than count — uses a map or set (stable keys)
variable "servers" {
  default = {
    web  = { type = "t3.micro",  az = "us-east-1a" }
    app  = { type = "t3.small",  az = "us-east-1b" }
    db   = { type = "t3.medium", az = "us-east-1c" }
  }
}

resource "aws_instance" "servers" {
  for_each = var.servers

  ami               = "ami-0c55b159cbfafe1f0"
  instance_type     = each.value.type
  availability_zone = each.value.az

  tags = { Name = each.key }  # "web", "app", "db"
}

# Reference by key
output "db_ip" {
  value = aws_instance.servers["db"].private_ip
}

# Using a set of strings
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key  # each.value == each.key for sets
}
```

### lifecycle

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    # Don't destroy before creating replacement
    create_before_destroy = true

    # Never destroy this resource (useful for databases)
    prevent_destroy = true

    # Ignore changes to these attributes (e.g., changed outside Terraform)
    ignore_changes = [
      ami,
      user_data,
      tags["LastModified"],
    ]

    # Custom condition check (Terraform 1.2+)
    precondition {
      condition     = var.instance_type != "t2.nano"
      error_message = "t2.nano is not supported in production."
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
}
```

### provider (within resource)

```hcl
resource "aws_s3_bucket" "replica" {
  provider = aws.eu       # use aliased provider
  bucket   = "eu-replica"
}
```

---

## 15. Provisioners

Provisioners run scripts on resources after creation. **Use sparingly** — prefer cloud-init, user_data, or configuration management tools (Ansible, Chef) when possible.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  key_name      = aws_key_pair.deployer.key_name

  # Connection config (used by provisioners)
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/deployer.pem")
    host        = self.public_ip
  }

  # Copy a file to the instance
  provisioner "file" {
    source      = "scripts/setup.sh"
    destination = "/tmp/setup.sh"
  }

  # Run a command remotely (SSH)
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/setup.sh",
      "sudo /tmp/setup.sh",
      "sudo systemctl start nginx",
    ]
  }

  # Run a command locally (on Terraform operator's machine)
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }

  # Run on destroy
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance ${self.id} destroyed' >> audit.log"
  }
}
```

---

## 16. Workspaces

Workspaces allow multiple state files in the same backend — useful for environment separation.

```bash
# List workspaces
terraform workspace list
# * default
#   staging
#   production

# Create a new workspace
terraform workspace new staging
terraform workspace new production

# Switch workspace
terraform workspace select production

# Show current workspace
terraform workspace show

# Delete a workspace
terraform workspace delete staging
```

### Using Workspace in Config

```hcl
locals {
  instance_type = {
    default    = "t3.micro"
    staging    = "t3.small"
    production = "t3.large"
  }
}

resource "aws_instance" "app" {
  instance_type = lookup(local.instance_type, terraform.workspace, "t3.micro")

  tags = {
    Environment = terraform.workspace
  }
}

# Different bucket per workspace
resource "aws_s3_bucket" "state" {
  bucket = "my-app-${terraform.workspace}-state"
}
```

> ⚠️ **Note**: Workspaces share the same backend and provider config. For strong environment isolation, consider separate root modules + directories instead.

---

## 17. Remote Backends

Remote backends store state remotely and enable team collaboration.

### S3 Backend (AWS)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # for state locking
  }
}
```

**Create the S3 bucket and DynamoDB table first:**

```hcl
# bootstrap/main.tf (apply this once manually)

resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Azure Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateaccount123"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### GCS Backend (Google Cloud)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod/network"
  }
}
```

### Migrating Backend

```bash
# After changing backend config, init will prompt to migrate
terraform init -migrate-state
```

---

## 18. Terraform Cloud & Enterprise

```hcl
# Connect to Terraform Cloud
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      name = "my-app-prod"
      # Or use tags:
      # tags = ["app:my-app", "env:prod"]
    }
  }
}
```

```bash
# Login to Terraform Cloud
terraform login

# Then init as normal
terraform init
terraform plan
terraform apply
```

**Key Terraform Cloud features:**
- Remote state storage and locking
- Remote plan & apply (no local credentials needed)
- VCS integration (GitHub, GitLab, Bitbucket)
- Cost estimation
- Sentinel policy enforcement
- Audit logging
- Private module registry

---

## 19. Import & Move

### terraform import (Bring existing infra under Terraform management)

```bash
# Import an existing EC2 instance
terraform import aws_instance.web i-0123456789abcdef0

# Import S3 bucket
terraform import aws_s3_bucket.main my-existing-bucket

# Import RDS instance
terraform import aws_db_instance.prod mydb

# Import with module path
terraform import module.vpc.aws_vpc.main vpc-0abc123def456789
```

### Import Block (Terraform 1.5+ — declarative import)

```hcl
# main.tf
import {
  to = aws_instance.web
  id = "i-0123456789abcdef0"
}

resource "aws_instance" "web" {
  # Terraform will generate these values on import
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

```bash
# Generate config from existing resources (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf
```

### moved Block (Rename/refactor without destroy)

```hcl
# When you rename a resource, use moved block to prevent destroy/recreate
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}

# Moving into a module
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}

# Moving from count to for_each
moved {
  from = aws_instance.web[0]
  to   = aws_instance.web["us-east-1"]
}
```

---

## 20. Testing

### terraform validate

```bash
# Validate HCL syntax and internal consistency
terraform validate
```

### terraform fmt

```bash
# Format all .tf files to canonical style
terraform fmt

# Check without modifying (useful in CI)
terraform fmt -check -recursive

# Show diff
terraform fmt -diff
```

### Built-in Test Framework (Terraform 1.6+)

```
project/
├── main.tf
├── variables.tf
└── tests/
    ├── basic.tftest.hcl
    └── advanced.tftest.hcl
```

```hcl
# tests/basic.tftest.hcl
variables {
  environment    = "test"
  instance_count = 1
}

run "creates_instance" {
  command = plan

  assert {
    condition     = aws_instance.web.instance_type == "t3.micro"
    error_message = "Wrong instance type for test environment"
  }
}

run "check_tags" {
  command = apply

  assert {
    condition     = aws_instance.web.tags["Environment"] == "test"
    error_message = "Environment tag missing or incorrect"
  }
}
```

```bash
terraform test
```

### Terratest (Go-based Integration Testing)

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "vpc_cidr":    "10.0.0.0/16",
            "environment": "test",
        },
    }

    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    vpcID := terraform.Output(t, opts, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

### Checkov (Security Scanning)

```bash
pip install checkov
checkov -d . --framework terraform
checkov -d . --check CKV_AWS_18,CKV_AWS_19
```

### tflint (Linting)

```bash
tflint --init
tflint --recursive
```

---

## 21. Security Best Practices

### Never Commit Secrets

```bash
# .gitignore
*.tfvars          # variable files may contain secrets
*.tfvars.json
.terraform/       # cached providers (large)
.terraform.lock.hcl  # optionally commit this for reproducibility
terraform.tfstate     # NEVER commit state (contains secrets)
terraform.tfstate.backup
crash.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

### Use AWS Secrets Manager / Vault

```hcl
# Fetch secrets from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/db_password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}

# HashiCorp Vault
data "vault_generic_secret" "db" {
  path = "secret/myapp/database"
}

resource "aws_db_instance" "main" {
  username = data.vault_generic_secret.db.data["username"]
  password = data.vault_generic_secret.db.data["password"]
}
```

### Restrict Provider Permissions

```hcl
# Use least-privilege IAM roles
# Don't use admin credentials for Terraform — create a dedicated role
# with only the permissions needed for your Terraform configs
```

### Encrypt State

```hcl
terraform {
  backend "s3" {
    bucket  = "my-state-bucket"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true          # encrypt at rest
    # State is also encrypted in transit via HTTPS by default
  }
}
```

### Sentinel Policies (Terraform Enterprise/Cloud)

```hcl
# policies/restrict_instance_types.sentinel
import "tfplan/v2" as tfplan

allowed_types = ["t3.micro", "t3.small", "t3.medium"]

main = rule {
  all tfplan.resource_changes as _, changes {
    changes.type is "aws_instance" implies
      changes.change.after.instance_type in allowed_types
  }
}
```

---

## 22. CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  TF_VERSION: "1.9.0"
  AWS_REGION: "us-east-1"

jobs:
  terraform:
    name: Terraform
    runs-on: ubuntu-latest
    permissions:
      id-token: write     # for OIDC
      contents: read
      pull-requests: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GithubActionsRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        continue-on-error: true

      - name: Comment PR with Plan
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply

variables:
  TF_VERSION: "1.9.0"

.terraform_base:
  image: hashicorp/terraform:$TF_VERSION
  before_script:
    - terraform init

validate:
  extends: .terraform_base
  stage: validate
  script:
    - terraform fmt -check
    - terraform validate

plan:
  extends: .terraform_base
  stage: plan
  script:
    - terraform plan -out=plan.tfplan
  artifacts:
    paths: [plan.tfplan]

apply:
  extends: .terraform_base
  stage: apply
  script:
    - terraform apply plan.tfplan
  when: manual
  only: [main]
```

---

## 23. Advanced Patterns

### Terragrunt (DRY Terraform)

Terragrunt keeps your Terraform DRY by providing extra tooling for managing multiple modules.

```hcl
# terragrunt.hcl (root)
remote_state {
  backend = "s3"
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

inputs = {
  aws_region = "us-east-1"
}
```

```hcl
# live/prod/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "github.com/my-org/terraform-modules//vpc?ref=v2.0"
}

inputs = {
  vpc_cidr    = "10.0.0.0/16"
  environment = "prod"
}
```

### Terraform CDK (CDKTF)

Write Terraform in TypeScript, Python, Go, Java, or C#:

```typescript
// main.ts (TypeScript CDKTF)
import { App, TerraformStack } from "cdktf";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { Instance } from "@cdktf/provider-aws/lib/instance";

class MyStack extends TerraformStack {
  constructor(scope: App, id: string) {
    super(scope, id);

    new AwsProvider(this, "AWS", { region: "us-east-1" });

    new Instance(this, "webServer", {
      ami: "ami-0c55b159cbfafe1f0",
      instanceType: "t3.micro",
      tags: { Name: "web-server" },
    });
  }
}

const app = new App();
new MyStack(app, "my-stack");
app.synth();
```

### Cross-Module Data Sharing (Remote State)

```hcl
# networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

# application/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "prod/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.networking.outputs.vpc_id
}
```

### Null Resource & Triggers Pattern

```hcl
resource "null_resource" "db_migrate" {
  # Re-run when the Docker image changes
  triggers = {
    image_tag = var.app_version
  }

  provisioner "local-exec" {
    command = "kubectl exec deploy/app -- python manage.py migrate"
  }
}
```

### Time-based Resources

```hcl
resource "time_sleep" "wait_30_seconds" {
  depends_on      = [aws_instance.web]
  create_duration = "30s"
}

resource "null_resource" "health_check" {
  depends_on = [time_sleep.wait_30_seconds]

  provisioner "local-exec" {
    command = "curl -f http://${aws_instance.web.public_ip}/health"
  }
}
```

### Blue-Green Deployment Pattern

```hcl
variable "traffic_distribution" {
  description = "Blue/Green traffic split"
  type = object({
    blue  = number  # 0-100
    green = number  # 0-100
  })
  default = { blue = 100, green = 0 }
}

resource "aws_lb_listener_rule" "blue" {
  listener_arn = aws_lb_listener.main.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.blue.arn
  }

  condition {
    path_pattern { values = ["/*"] }
  }
}
```

---

## 24. Troubleshooting

### Enable Debug Logging

```bash
# Levels: TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG=DEBUG
export TF_LOG_PATH="./terraform.log"
terraform apply
```

### Common Issues & Fixes

**State lock stuck:**
```bash
terraform force-unlock <LOCK_ID>
```

**Provider version conflict:**
```bash
# Delete lock file and reinit
rm .terraform.lock.hcl
terraform init -upgrade
```

**Resource already exists:**
```bash
# Import the existing resource instead of recreating
terraform import aws_s3_bucket.main existing-bucket-name
```

**Cycle/dependency error:**
```bash
# Use depends_on carefully, or restructure resources
# Use terraform graph to visualize dependencies
terraform graph | dot -Tsvg > graph.svg
```

**Out-of-date state:**
```bash
terraform refresh
# or
terraform apply -refresh-only
```

**Targeted operations:**
```bash
# Only plan/apply a specific resource
terraform plan -target=aws_instance.web
terraform apply -target=module.vpc

# Destroy only a specific resource
terraform destroy -target=aws_instance.old_server
```

### Useful Commands Cheatsheet

```bash
# Initialize (download providers & modules)
terraform init

# Format code
terraform fmt -recursive

# Validate configuration
terraform validate

# Plan with saved output
terraform plan -out=tfplan

# Apply saved plan
terraform apply tfplan

# Apply with auto-approve (use in CI only)
terraform apply -auto-approve

# Destroy everything
terraform destroy

# Show current state
terraform show

# Show specific resource
terraform state show aws_instance.web

# Refresh state
terraform refresh

# View dependency graph
terraform graph | dot -Tsvg > graph.svg

# Get provider schemas
terraform providers schema -json | jq .

# Console (interactive expression testing)
terraform console
> length(["a", "b", "c"])
3
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
```

---

## Quick Reference Card

| Concept | Syntax / Command |
|---|---|
| Resource | `resource "type" "name" { ... }` |
| Variable | `var.variable_name` |
| Local | `local.local_name` |
| Data source | `data.type.name.attr` |
| Module output | `module.name.output` |
| Count index | `count.index` |
| For_each key | `each.key` / `each.value` |
| Init | `terraform init` |
| Plan | `terraform plan -out=tfplan` |
| Apply | `terraform apply tfplan` |
| Destroy | `terraform destroy` |
| State list | `terraform state list` |
| Format | `terraform fmt -recursive` |
| Validate | `terraform validate` |
| Console | `terraform console` |
| Debug | `TF_LOG=DEBUG terraform apply` |

---

*Last updated: 2025 | Terraform v1.9+ | HashiCorp Terraform Documentation: https://developer.hashicorp.com/terraform/docs*
