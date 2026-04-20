# Terraform Troubleshooting Systematic Guide

## 🎯 Troubleshooting Workflow

```
1. Identify the error message
2. Check Terraform state
3. Validate configuration syntax
4. Verify provider credentials
5. Inspect state file integrity
6. Debug module issues
7. Fix and re-apply
```

---

## 📋 Essential Diagnostic Commands

### Quick Health Check
```bash
# Validate syntax
terraform validate

# Check formatting
terraform fmt -check -recursive

# View current state
terraform show

# List resources in state
terraform state list

# Check Terraform version
terraform version

# Initialize and check plugins
terraform init -upgrade
```

---

## 🔥 Common Issues & Troubleshooting

### 1. Terraform Init Failures

**Symptoms:**
- "Error installing provider"
- "Failed to query available provider packages"
- "Could not retrieve the list of available versions"

**Diagnostic Commands:**
```bash
# Check with debug logging
TF_LOG=DEBUG terraform init 2>&1 | tee init-debug.log

# Check provider configuration
cat .terraform.lock.hcl

# Verify network connectivity to registry
curl -I https://registry.terraform.io/v1/providers/hashicorp/aws

# Check for cached providers
ls -la .terraform/providers/

# Clear cache and retry
rm -rf .terraform
rm .terraform.lock.hcl
terraform init
```

**What to Check in Logs:**
```bash
# Look for these patterns
grep -i "error\|failed\|timeout" init-debug.log
grep "certificate" init-debug.log
grep "authentication" init-debug.log
grep "network" init-debug.log
```

**Common Log Patterns:**
- `Error: Failed to install provider` → Version constraint issue
- `Error: Failed to query available provider packages` → Network/proxy issue
- `x509: certificate signed by unknown authority` → SSL/TLS issue
- `timeout` → Firewall/connectivity issue

**Practice Scenario:**
```bash
# Create config with wrong provider version
cat > main.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "99.99.99"  # Non-existent version
    }
  }
}
EOF

# Try to initialize
terraform init
# Error: Failed to query available provider packages

# Fix: Use valid version or range
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

---

### 2. Syntax & Validation Errors

**Symptoms:**
- "Error: Invalid resource type"
- "Error: Unsupported argument"
- "Error: Missing required argument"

**Diagnostic Commands:**
```bash
# Validate configuration
terraform validate

# Check formatting issues
terraform fmt -check -recursive

# Validate specific file
terraform validate -json | jq '.diagnostics[]'

# Show detailed error with line numbers
terraform plan 2>&1 | grep -A 5 "Error:"
```

**What to Check in Logs:**
```bash
# Look for line numbers
grep "on main.tf line" error.log

# Find argument issues
grep "argument" error.log

# Find attribute issues
grep "attribute" error.log
```

**Practice Scenario:**
```bash
# Create config with syntax error
cat > main.tf << 'EOF'
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Web Server
    # Missing closing quote ^
  }
}
EOF

# Validate to find error
terraform validate
# Error on main.tf line 7: Unterminated template string

# Fix the syntax
terraform fmt main.tf
terraform validate
# Success! Configuration is valid
```

---

### 3. State File Issues

**Symptoms:**
- "Error acquiring the state lock"
- "Error loading state: state snapshot was created by Terraform v..."
- "Resource not found in state"

**Diagnostic Commands:**
```bash
# Check state file location
terraform state list

# Show specific resource state
terraform state show aws_instance.web

# Verify state file integrity
terraform state pull | jq '.version'

# Check for state lock
aws dynamodb get-item \
  --table-name terraform-lock-table \
  --key '{"LockID": {"S": "your-bucket/terraform.tfstate-md5"}}'

# Force unlock (DANGEROUS - use only if sure)
terraform force-unlock <LOCK_ID>

# View state file version
terraform state pull | jq '.terraform_version'
```

**What to Check in Logs:**
```bash
# State lock issues
grep -i "lock\|LockID" terraform.log

# State version mismatch
grep "version" terraform.log

# State corruption
terraform state pull | jq '.' > /dev/null
echo $?  # 0 = valid JSON, non-zero = corrupted
```

**Common State Issues:**

#### A. State Lock Error
```bash
# Error message:
# Error: Error acquiring the state lock
# Lock Info:
#   ID:        abc123...
#   Path:      s3-bucket/terraform.tfstate
#   Operation: OperationTypePlan
#   Who:       user@hostname
#   Created:   2024-01-20 10:30:00

# Diagnostic:
# 1. Check if another process is running
ps aux | grep terraform

# 2. Check lock in DynamoDB
aws dynamodb scan --table-name terraform-lock-table

# 3. If stale lock (previous run crashed):
terraform force-unlock <LOCK_ID>
```

#### B. State Drift Detection
```bash
# Compare state with actual infrastructure
terraform plan -detailed-exitcode

# Exit codes:
# 0 = no changes
# 1 = error
# 2 = changes detected (drift!)

# List resources with drift
terraform plan -no-color | grep -E "will be (created|destroyed|updated)"

# Show specific resource drift
terraform state show aws_instance.web
terraform plan -target=aws_instance.web
```

#### C. Resource Removed from State
```bash
# Error: Resource 'aws_instance.web' not found in state

# Check if resource exists in state
terraform state list | grep aws_instance

# If missing, import it back
terraform import aws_instance.web i-1234567890abcdef

# Verify import
terraform state show aws_instance.web
terraform plan  # Should show no changes
```

**Practice Scenario:**
```bash
# Setup: Create resource with state lock
cat > backend.tf << 'EOF'
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "test/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
EOF

# Simulate stale lock
# In terminal 1: Start a plan (Ctrl+C to interrupt)
terraform plan

# In terminal 2: Try another operation
terraform plan
# Error: Error acquiring the state lock

# Diagnose lock
aws dynamodb scan --table-name terraform-locks

# Force unlock
terraform force-unlock <ID_from_error_message>

# Verify
terraform plan  # Should work now
```

---

### 4. Module Issues

**Symptoms:**
- "Module not found"
- "Error loading modules"
- "Invalid module source"

**Diagnostic Commands:**
```bash
# Initialize and download modules
terraform init

# Show module tree
terraform get

# Validate including modules
terraform validate

# Show module sources
grep -r "source.*=" ./*.tf

# Check downloaded modules
ls -la .terraform/modules/

# Show specific module
terraform state list | grep module.

# Debug module path
terraform console
> module.vpc
```

**What to Check in Logs:**
```bash
# Module source issues
grep "Module not found" terraform.log
grep "source" terraform.log

# Module version issues
grep "version constraint" terraform.log

# Module initialization
terraform init 2>&1 | grep -A 5 "module\."
```

**Common Module Issues:**

#### A. Module Source Path Wrong
```bash
# Wrong path
module "vpc" {
  source = "./modules/vpc"  # But directory doesn't exist
}

# Diagnostic
ls -la ./modules/vpc
# No such file or directory

terraform init
# Error: Module not found

# Fix: Correct the path
module "vpc" {
  source = "./network/vpc"
}
```

#### B. Module Version Conflict
```bash
# Using registry module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "1.0.0"  # Old version with breaking changes
}

# Diagnostic
terraform init
# May download but...

terraform plan
# Error: Unsupported argument "enable_nat_gateway"

# Fix: Check module documentation for correct version
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  # Use correct arguments for this version
  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

#### C. Module Output Not Available
```bash
# In root module
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnet_ids[0]
  # Error: Unsupported attribute
}

# Diagnostic: Check module outputs
terraform console
> module.vpc

# Or check module source
cat .terraform/modules/vpc/outputs.tf

# Issue: Output name is different
# outputs.tf has: private_subnets (not private_subnet_ids)

# Fix
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnets[0]
}
```

**Practice Scenario:**
```bash
# Create module structure
mkdir -p modules/ec2

cat > modules/ec2/main.tf << 'EOF'
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = var.tags
}
EOF

cat > modules/ec2/variables.tf << 'EOF'
variable "ami_id" {
  type = string
}

variable "instance_type" {
  type = string
}

variable "tags" {
  type = map(string)
  default = {}
}
EOF

cat > modules/ec2/outputs.tf << 'EOF'
output "instance_id" {
  value = aws_instance.this.id
}

output "public_ip" {
  value = aws_instance.this.public_ip
}
EOF

# Use module incorrectly
cat > main.tf << 'EOF'
module "web_server" {
  source = "./modules/ec2"
  
  ami_id = "ami-12345678"
  # Missing required variable instance_type
}
EOF

# Try to plan
terraform init
terraform plan
# Error: Missing required argument "instance_type"

# Fix
cat > main.tf << 'EOF'
module "web_server" {
  source = "./modules/ec2"
  
  ami_id        = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Web Server"
    Env  = "production"
  }
}

output "web_server_ip" {
  value = module.web_server.public_ip
}
EOF

terraform plan  # Should work now
```

---

### 5. Provider Authentication Issues

**Symptoms:**
- "Error: error configuring Terraform AWS Provider"
- "NoCredentialProviders: no valid providers in chain"
- "AuthFailure: AWS was not able to validate the provided access credentials"

**Diagnostic Commands:**
```bash
# Check AWS credentials
aws sts get-caller-identity

# Check credential chain
export AWS_PROFILE=default
aws configure list

# Test with debug
TF_LOG=DEBUG terraform plan 2>&1 | grep -i "credential\|auth"

# Verify provider configuration
terraform console
> provider::aws::region

# Check environment variables
env | grep AWS
```

**What to Check in Logs:**
```bash
# Authentication failures
grep -i "authfailure\|unauthorized\|forbidden" terraform.log

# Credential issues
grep -i "credential\|access key" terraform.log

# Region issues
grep -i "region" terraform.log
```

**Practice Scenario:**
```bash
# Simulate missing credentials
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN

# Try to plan
terraform plan
# Error: NoCredentialProviders

# Diagnostic
aws sts get-caller-identity
# Unable to locate credentials

# Fix: Set credentials
export AWS_PROFILE=your-profile
# OR
aws configure

# Verify
aws sts get-caller-identity
terraform plan  # Should work now
```

---

### 6. Resource Dependencies & Cycles

**Symptoms:**
- "Error: Cycle detected in graph"
- "depends_on must be a list"
- "Resource depends on itself"

**Diagnostic Commands:**
```bash
# Generate dependency graph
terraform graph | dot -Tsvg > graph.svg

# Or use online viewer
terraform graph > graph.dot
# Upload to http://www.webgraphviz.com/

# Show resource dependencies
terraform state list
terraform state show aws_instance.web

# Check plan with detailed output
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan | jq '.configuration.root_module.resources[]'
```

**What to Check in Logs:**
```bash
# Cycle errors
grep -i "cycle" terraform.log

# Dependency issues
grep "depends_on\|dependency" terraform.log

# Graph errors
terraform graph 2>&1 | grep -i error
```

**Practice Scenario:**
```bash
# Create circular dependency
cat > main.tf << 'EOF'
resource "aws_security_group" "app" {
  name = "app-sg"
  
  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.lb.id]
  }
}

resource "aws_security_group" "lb" {
  name = "lb-sg"
  
  egress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # Circular!
  }
}
EOF

# Try to apply
terraform init
terraform plan
# Error: Cycle detected in graph

# Fix: Use security_group_rules
cat > main.tf << 'EOF'
resource "aws_security_group" "app" {
  name = "app-sg"
}

resource "aws_security_group" "lb" {
  name = "lb-sg"
}

resource "aws_security_group_rule" "app_from_lb" {
  type                     = "ingress"
  from_port                = 80
  to_port                  = 80
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.lb.id
  security_group_id        = aws_security_group.app.id
}

resource "aws_security_group_rule" "lb_to_app" {
  type                     = "egress"
  from_port                = 80
  to_port                  = 80
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.app.id
  security_group_id        = aws_security_group.lb.id
}
EOF

terraform plan  # No cycle!
```

---

### 7. State File Corruption & Recovery

**Symptoms:**
- "Error loading state: unexpected EOF"
- "Error parsing state file"
- State file shows all zeros or is empty

**Diagnostic Commands:**
```bash
# Check state file integrity
terraform state pull | jq '.'

# Check state file size
ls -lh terraform.tfstate

# Validate JSON structure
cat terraform.tfstate | jq empty
echo $?  # 0 = valid, non-zero = corrupted

# Check backup
ls -lh terraform.tfstate.backup

# Pull from remote backend
terraform state pull > current-state.json

# Compare with backup
diff terraform.tfstate terraform.tfstate.backup
```

**Recovery Steps:**

```bash
# Step 1: Backup current (even if corrupted)
cp terraform.tfstate terraform.tfstate.corrupted
cp terraform.tfstate.backup terraform.tfstate.backup.safe

# Step 2: Try to recover from backup
cp terraform.tfstate.backup terraform.tfstate

# Step 3: Validate recovered state
terraform state pull | jq '.'

# Step 4: Refresh to sync with actual infrastructure
terraform refresh

# Step 5: Verify
terraform plan
# Should show no changes if recovery successful

# If backup also corrupted, rebuild state:
# 1. Remove state file
rm terraform.tfstate

# 2. Re-import all resources
terraform import aws_instance.web i-1234567890
terraform import aws_security_group.web sg-12345678
# ... repeat for all resources

# 3. Verify
terraform plan  # Should show no changes
```

**Prevention:**

```bash
# Use remote backend with versioning
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    
    # Enable versioning on the S3 bucket
  }
}

# Regular backups
# Add to CI/CD pipeline:
aws s3 cp s3://my-terraform-state/prod/terraform.tfstate \
  ./backups/terraform-$(date +%Y%m%d-%H%M%S).tfstate

# Or use Terraform Cloud with built-in versioning
```

---

### 8. Terraform State Move/Refactoring

**Symptoms:**
- "Resource will be destroyed and recreated" (when you just renamed it)
- "Resource not found in state"
- Need to reorganize code without destroying infrastructure

**Diagnostic Commands:**
```bash
# List current state
terraform state list

# Show resource details
terraform state show aws_instance.web

# Preview move (doesn't actually move)
terraform plan

# Check what would change
terraform state mv -dry-run aws_instance.old aws_instance.new
```

**Common Refactoring Scenarios:**

#### A. Rename Resource (No Module)
```bash
# Before: main.tf
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# Want to rename to: web_server

# WRONG WAY (destroys and recreates):
# Just change code and apply

# RIGHT WAY:
# 1. Move in state first
terraform state mv aws_instance.web aws_instance.web_server

# 2. Update code
resource "aws_instance" "web_server" {  # Renamed
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# 3. Verify no changes
terraform plan
# No changes. Infrastructure is up-to-date.
```

#### B. Move Resource to Module
```bash
# Before: Resource in root
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# After: Resource in module
module "web_server" {
  source = "./modules/ec2"
  
  ami_id        = "ami-12345678"
  instance_type = "t2.micro"
}

# Steps:
# 1. Create module structure
mkdir -p modules/ec2
# ... create module files

# 2. Move state
terraform state mv aws_instance.web module.web_server.aws_instance.this

# 3. Update root config
# Remove old resource, add module block

# 4. Verify
terraform plan  # No changes
```

#### C. Split Monolithic State
```bash
# Scenario: One state file for entire infrastructure
# Want to split into: networking, compute, database

# Step 1: Create new backend configs
# networking/backend.tf
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Step 2: Pull current state
terraform state pull > full-state.json

# Step 3: Move resources to new workspace
# In networking directory:
terraform init
terraform state push ../full-state.json
terraform state rm aws_instance.web  # Remove non-networking resources
terraform state rm aws_db_instance.main
# Keep only VPC, subnets, etc.

# Step 4: Repeat for other components

# Step 5: Verify each component
cd networking && terraform plan  # Only networking resources
cd ../compute && terraform plan   # Only compute resources
```

**Practice Scenario:**
```bash
# Create initial infrastructure
cat > main.tf << 'EOF'
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id
}
EOF

terraform init
terraform apply -auto-approve

# Now refactor to modules
mkdir -p modules/{vpc,ec2}

# Create VPC module
cat > modules/vpc/main.tf << 'EOF'
resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = var.public_subnet_cidr
}
EOF

# ... (create variables and outputs)

# Move state to module
terraform state mv aws_vpc.main module.vpc.aws_vpc.main
terraform state mv aws_subnet.public module.vpc.aws_subnet.public
terraform state mv aws_instance.web module.ec2.aws_instance.web

# Update main.tf
cat > main.tf << 'EOF'
module "vpc" {
  source             = "./modules/vpc"
  cidr_block         = "10.0.0.0/16"
  public_subnet_cidr = "10.0.1.0/24"
}

module "ec2" {
  source        = "./modules/ec2"
  ami_id        = "ami-12345678"
  instance_type = "t2.micro"
  subnet_id     = module.vpc.public_subnet_id
}
EOF

# Verify no changes
terraform plan
# No changes. Infrastructure is up-to-date.
```

---

## 🔍 Advanced Debugging Techniques

### Enable Debug Logging
```bash
# Comprehensive debug logging
export TF_LOG=DEBUG
export TF_LOG_PATH=./terraform-debug.log

terraform plan

# Specific component logging
export TF_LOG_CORE=DEBUG      # Terraform core
export TF_LOG_PROVIDER=DEBUG  # Provider plugin

# Disable logging
unset TF_LOG
unset TF_LOG_PATH
```

### Analyze Debug Logs
```bash
# Find errors
grep -i "error\|fail" terraform-debug.log

# Find provider API calls
grep "aws_" terraform-debug.log

# Find state operations
grep "state" terraform-debug.log

# Find authentication issues
grep -i "credential\|auth" terraform-debug.log

# Find timing issues
grep "elapsed" terraform-debug.log
```

### Interactive Debugging with Console
```bash
# Start Terraform console
terraform console

# Test expressions
> var.region
> module.vpc.vpc_id
> aws_instance.web.private_ip

# Test functions
> cidrsubnet("10.0.0.0/16", 8, 1)
> lookup(var.tags, "Environment", "dev")

# Debug complex expressions
> [for s in var.subnets : s.cidr if s.public]
```

### Graph Visualization
```bash
# Generate dependency graph
terraform graph > graph.dot

# Convert to image (requires graphviz)
terraform graph | dot -Tpng > graph.png

# View online
terraform graph | pbcopy  # Copy to clipboard
# Paste at http://www.webgraphviz.com/

# Filter graph to specific resource
terraform graph | grep -A 5 "aws_instance.web"
```

---

## 📊 Systematic Troubleshooting Checklist

When Terraform fails, follow this order:

```bash
# 1. Check syntax
terraform validate

# 2. Check formatting
terraform fmt -check -recursive

# 3. Check initialization
terraform init -upgrade

# 4. Check state
terraform state list
terraform show

# 5. Enable debug logging
export TF_LOG=DEBUG
export TF_LOG_PATH=./debug.log

# 6. Try the operation
terraform plan  # or apply

# 7. Analyze logs
grep -i "error" debug.log
tail -100 debug.log

# 8. Check specific resource
terraform state show <resource>

# 9. Check provider credentials
aws sts get-caller-identity  # For AWS

# 10. Check dependencies
terraform graph | grep <resource>
```

---

## 🎓 Real-World Production Scenarios

### Scenario 1: Accidental Resource Destruction

**Context:** Developer updated EC2 instance type, but Terraform wants to destroy/recreate production database.

```bash
# The problem:
resource "aws_db_instance" "main" {
  instance_class    = "db.t3.large"  # Changed from db.t3.medium
  # ...
}

terraform plan
# Plan: 0 to add, 0 to change, 1 to destroy, 1 to create
# aws_db_instance.main must be replaced

# Diagnostic:
# Check which attributes force replacement
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan | jq '.resource_changes[] | select(.address=="aws_db_instance.main")'

# Shows: "before_sensitive": false, "after_unknown": true
# Attribute "instance_class" forces replacement

# Solution: Use apply_method = "immediate" and it won't recreate
resource "aws_db_instance" "main" {
  instance_class    = "db.t3.large"
  apply_immediately = true  # Allows in-place modification
  # ...
}

terraform plan
# Now shows: 0 to destroy, 1 to change (in-place update)
```

### Scenario 2: State Lock During CI/CD Pipeline

**Context:** Multiple CI/CD pipelines trying to run Terraform simultaneously.

```bash
# Pipeline 1: Running terraform apply
# Pipeline 2: Starts, gets lock error

# Error in Pipeline 2:
# Error: Error acquiring the state lock
# Lock Info:
#   ID:        abc123-def456
#   Who:       jenkins@ci-worker-01
#   Created:   2024-01-20 14:30:00

# Diagnostic:
# Check DynamoDB lock table
aws dynamodb scan \
  --table-name terraform-locks \
  --filter-expression "LockID = :lock" \
  --expression-attribute-values '{":lock":{"S":"terraform-state-prod"}}'

# Check if Pipeline 1 is still running
# If yes: Wait for completion
# If no (crashed): Force unlock

terraform force-unlock abc123-def456

# Prevention: Add lock timeouts and retries to pipeline
# In Jenkinsfile:
retry(3) {
  sh 'terraform apply -auto-approve -lock-timeout=10m'
}
```

### Scenario 3: Module Version Breaking Change

**Context:** Upgrading VPC module version breaks production deployment.

```bash
# Before: Using old version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.0.0"
  
  enable_nat_gateway = true
  single_nat_gateway = true
}

# After: Upgraded to 5.0
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  enable_nat_gateway = true
  single_nat_gateway = true
}

terraform init -upgrade
terraform plan
# Error: Unsupported argument
# The argument "single_nat_gateway" is not expected here

# Diagnostic:
# Check module documentation
curl -s https://registry.terraform.io/v1/modules/terraform-aws-modules/vpc/aws/5.0.0 | jq

# Or check locally
cat .terraform/modules/vpc/variables.tf | grep -A 5 "nat_gateway"

# Solution: Update to new argument names
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  enable_nat_gateway    = true
  enable_single_nat_gateway = true  # New name in v5
}
```

### Scenario 4: Drift Detection in Production

**Context:** Manual changes made to infrastructure outside Terraform.

```bash
# Daily drift detection job runs
terraform plan -detailed-exitcode
# Exit code: 2 (changes detected)

# Diagnostic: See what changed
terraform plan -no-color > drift-report.txt

cat drift-report.txt
# Shows:
# ~ aws_security_group.web
#   ~ ingress {
#       + cidr_blocks = ["0.0.0.0/0"]
#       - cidr_blocks = ["10.0.0.0/8"]
#     }

# Someone opened port to internet manually!

# Options:
# 1. Revert manual change
terraform apply -auto-approve

# 2. Accept manual change as new baseline
terraform refresh
# Update code to match
resource "aws_security_group" "web" {
  ingress {
    cidr_blocks = ["0.0.0.0/0"]  # Updated to match reality
  }
}

# 3. Import if resource removed from state
terraform import aws_security_group.web sg-12345678
```

### Scenario 5: Import Existing Infrastructure

**Context:** Team built infrastructure manually, now want to manage with Terraform.

```bash
# Step 1: Write Terraform config matching existing resources
resource "aws_instance" "existing_web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  # Add all attributes
}

# Step 2: Import existing resource
terraform import aws_instance.existing_web i-0123456789abcdef0

# Step 3: Check for differences
terraform plan
# Shows all the differences between your config and actual

# Step 4: Adjust config to match exactly
# Update your resource definition until:
terraform plan
# Shows: No changes. Infrastructure is up-to-date.

# Common attributes to check:
# - tags
# - security_groups
# - subnet_id
# - iam_instance_profile
# - user_data

# Use terraform show to see imported state
terraform state show aws_instance.existing_web
# Copy values into your config
```

---

## 💡 Key Takeaways

1. **Always validate first** - `terraform validate` catches syntax errors
2. **Use debug logging** - `TF_LOG=DEBUG` reveals provider API calls
3. **Check state health** - `terraform state list` and `terraform show`
4. **Lock management** - Force unlock only if certain previous run crashed
5. **Module versioning** - Pin versions to avoid breaking changes
6. **State file backups** - Use remote backend with versioning
7. **Graph visualization** - Use `terraform graph` for dependency issues
8. **Import carefully** - Match config exactly before importing
9. **Drift detection** - Regular `terraform plan` in CI/CD
10. **Test refactoring** - Use `terraform state mv` for safe renames

---

## 📚 Quick Reference Card

| Issue Type | First Command | Key Log Pattern |
|------------|---------------|-----------------|
| Init fails | `terraform init -upgrade` | error installing provider |
| Syntax error | `terraform validate` | on line X |
| State locked | Check DynamoDB/state backend | Error acquiring lock |
| Module issue | `terraform get` | Module not found |
| Auth failure | `aws sts get-caller-identity` | NoCredentialProviders |
| Cycle detected | `terraform graph` | Cycle detected |
| State corrupt | `terraform state pull` | unexpected EOF |
| Drift | `terraform plan -detailed-exitcode` | Exit code 2 |

---

**Remember**: Terraform errors are usually clear - read the full error message. The line number and resource address tell you exactly where to look.
