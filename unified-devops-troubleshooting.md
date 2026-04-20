# Unified DevOps Troubleshooting Framework
## Docker + Terraform + Jenkins Integration

This guide provides a **systematic approach** to troubleshoot issues across the entire DevOps pipeline, from infrastructure provisioning (Terraform) to CI/CD (Jenkins) to container runtime (Docker).

---

## 🎯 Universal Troubleshooting Workflow

```
1. IDENTIFY   → What failed? Where in the pipeline?
2. ISOLATE    → Which tool/component?
3. INSPECT    → Check logs, state, configuration
4. DIAGNOSE   → Root cause analysis
5. FIX        → Apply solution
6. VERIFY     → Confirm resolution
7. DOCUMENT   → Update runbook
```

---

## 📊 Cross-Tool Issue Matrix

| Symptom | Docker | Terraform | Jenkins | Common Root Cause |
|---------|--------|-----------|---------|-------------------|
| **Permission Denied** | Volume mounts | State file access | Workspace/credentials | File permissions, user mismatch |
| **Network Timeout** | Container connectivity | Provider API | Git checkout | Firewall, proxy, DNS |
| **Resource Exhaustion** | OOMKilled | Large plan/apply | Build fails | Insufficient memory/disk |
| **Auth Failure** | Registry login | Provider credentials | Git/AWS credentials | Expired/wrong credentials |
| **File Not Found** | Missing volume | Module not found | Tool not installed | Path misconfiguration |
| **Syntax Error** | Dockerfile/compose | .tf configuration | Jenkinsfile | Invalid configuration |
| **State Corruption** | Container state | tfstate file | Build metadata | Aborted operations |
| **Version Conflict** | Image tags | Provider versions | Plugin versions | Incompatible versions |

---

## 🔥 End-to-End Pipeline Scenarios

### Scenario 1: Complete CI/CD Pipeline Failure

**Context:** Jenkins pipeline provisions infrastructure with Terraform, then deploys Docker containers.

**Pipeline Flow:**
```
Jenkins → Terraform (create AWS infra) → Docker (deploy app) → Tests
```

**Failure Point:** Each stage can fail, and errors cascade.

---

#### Stage 1: Jenkins Checkout Fails

**Error:**
```
[Pipeline] checkout
ERROR: Error cloning remote repo 'origin'
Permission denied (publickey)
```

**Diagnostic Process:**

```bash
# 1. Check from Jenkins perspective
# Manage Jenkins → Script Console:
println "ssh -T git@github.com".execute().text

# 2. Check from Jenkins agent
ssh jenkins-agent-01
sudo -u jenkins ssh -T git@github.com

# 3. Verify SSH key
sudo -u jenkins cat ~/.ssh/id_rsa.pub
# Compare with GitHub deploy key

# 4. Test manually
sudo -u jenkins git clone git@github.com:org/repo.git /tmp/test
```

**Cross-Tool Check:**
```bash
# Check Docker if Jenkins is running in container
docker exec jenkins-master ssh -T git@github.com

# Check Terraform if SSH key created by Terraform
terraform state show aws_instance.jenkins_master | grep key_name
```

**Fix:**
```bash
# Add SSH key to Jenkins
sudo su - jenkins
ssh-keygen -t rsa -b 4096
cat ~/.ssh/id_rsa.pub
# Add to GitHub → Settings → Deploy Keys

# Or mount SSH key if Jenkins in Docker
docker run -d \
  -v /home/user/.ssh:/var/jenkins_home/.ssh:ro \
  jenkins/jenkins:lts
```

---

#### Stage 2: Terraform Init Fails

**Error:**
```
[Pipeline] sh 'terraform init'
Error: Failed to query available provider packages
```

**Diagnostic Process:**

```bash
# 1. Check from Jenkins workspace
# In Jenkins Console Output, find workspace path
cd /var/lib/jenkins/workspace/deploy-pipeline

# 2. Check network from Jenkins agent
curl -I https://registry.terraform.io

# 3. Check Terraform version
terraform version

# 4. Check provider configuration
cat main.tf | grep required_providers

# 5. Enable debug logging
export TF_LOG=DEBUG
terraform init 2>&1 | tee init-debug.log
```

**Cross-Tool Check:**
```bash
# If Jenkins in Docker, check Docker network
docker exec jenkins-agent curl -I https://registry.terraform.io

# Check if proxy needed
docker inspect jenkins-agent --format='{{.Config.Env}}' | grep -i proxy

# Check if Terraform installed correctly
docker exec jenkins-agent which terraform
docker exec jenkins-agent terraform version
```

**Fix:**
```bash
# Option 1: Add proxy to Jenkins agent
# In Jenkinsfile:
pipeline {
    agent any
    
    environment {
        HTTP_PROXY = 'http://proxy.company.com:8080'
        HTTPS_PROXY = 'http://proxy.company.com:8080'
    }
    
    stages {
        stage('Terraform Init') {
            steps {
                sh 'terraform init'
            }
        }
    }
}

# Option 2: Use Docker agent with Terraform pre-installed
pipeline {
    agent {
        docker {
            image 'hashicorp/terraform:1.5'
            args '-v $HOME/.terraform.d:/root/.terraform.d'
        }
    }
    stages {
        stage('Terraform Init') {
            steps {
                sh 'terraform init'
            }
        }
    }
}
```

---

#### Stage 3: Terraform Apply - State Lock Error

**Error:**
```
[Pipeline] sh 'terraform apply -auto-approve'
Error: Error acquiring the state lock
Lock Info:
  ID: abc-123-def
  Who: jenkins@ci-worker-02
  Created: 2024-01-20 10:30:00
```

**Diagnostic Process:**

```bash
# 1. Check if other Jenkins jobs running Terraform
# In Jenkins → Build Queue
# Look for running jobs that might hold lock

# 2. Check DynamoDB lock table (for S3 backend)
aws dynamodb scan --table-name terraform-locks

# 3. Check if previous build was aborted
# Jenkins UI → Job → Last Build → Console Output
# Look for "Aborted by user" or build timeout

# 4. Check lock timestamp
# If > 1 hour old and no job running → stale lock

# 5. Verify which workspace holds the lock
echo "Lock Path: s3-bucket/path/terraform.tfstate"
# Cross-reference with Jenkins workspaces
```

**Cross-Tool Check:**
```bash
# Check if Docker container holding lock
docker ps | grep terraform
# If found, check its process:
docker top <container-id>

# Check Jenkins builds
curl -s http://localhost:8080/api/json | \
  jq '.jobs[] | select(.name | contains("terraform"))'
```

**Fix:**
```bash
# Option 1: Wait for legitimate lock to release
# Monitor until other job completes

# Option 2: Force unlock if stale (after verification)
# In Jenkins job or manually:
terraform force-unlock abc-123-def

# Option 3: Prevent future locks with Jenkins pipeline
pipeline {
    agent any
    
    options {
        lock(resource: 'terraform-state-prod', skipIfLocked: true)
    }
    
    stages {
        stage('Terraform Apply') {
            steps {
                sh '''
                    terraform init
                    terraform apply -auto-approve -lock-timeout=10m
                '''
            }
        }
    }
}

# Option 4: Use separate workspaces per environment
pipeline {
    stages {
        stage('Terraform') {
            steps {
                sh '''
                    terraform workspace select ${ENVIRONMENT} || terraform workspace new ${ENVIRONMENT}
                    terraform init
                    terraform apply -auto-approve
                '''
            }
        }
    }
}
```

---

#### Stage 4: Docker Build Fails in Jenkins

**Error:**
```
[Pipeline] sh 'docker build -t myapp .'
Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

**Diagnostic Process:**

```bash
# 1. Check Docker daemon on Jenkins agent
ssh jenkins-agent-01
sudo systemctl status docker
docker ps

# 2. Check Jenkins user permissions
sudo groups jenkins
# Should include 'docker'

# 3. Test Docker access as Jenkins user
sudo -u jenkins docker ps

# 4. Check from Jenkins context
# In Jenkinsfile, add debug stage:
stage('Debug') {
    steps {
        sh 'whoami'
        sh 'groups'
        sh 'ls -la /var/run/docker.sock'
        sh 'docker version || echo "Docker not accessible"'
    }
}
```

**Cross-Tool Integration:**

```bash
# Scenario: Terraform created EC2 instance for Jenkins agent
# Docker not installed by default

# Check Terraform state
terraform state show aws_instance.jenkins_agent

# Verify user_data script
terraform state show aws_instance.jenkins_agent | grep user_data

# Expected user_data:
#!/bin/bash
apt-get update
apt-get install -y docker.io
usermod -aG docker jenkins
systemctl enable docker
systemctl start docker
```

**Fix:**

```bash
# Option 1: Add jenkins to docker group on agent
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins-agent

# Option 2: Update Terraform to install Docker
# In Terraform configuration:
resource "aws_instance" "jenkins_agent" {
  ami           = "ami-12345678"
  instance_type = "t3.medium"
  
  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y docker.io
    usermod -aG docker ubuntu
    systemctl enable docker
    systemctl start docker
    
    # Install Jenkins agent
    wget -O agent.jar ${var.jenkins_url}/jnlpJars/agent.jar
    # ... agent setup
  EOF
  
  tags = {
    Name = "Jenkins-Agent"
  }
}

# Option 3: Use Docker-in-Docker in Jenkinsfile
pipeline {
    agent {
        docker {
            image 'docker:dind'
            args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t myapp .'
            }
        }
    }
}

# Option 4: Build Docker image with Terraform-created infrastructure
# After Terraform creates ECR:
pipeline {
    stages {
        stage('Get ECR Login') {
            steps {
                script {
                    env.ECR_LOGIN = sh(
                        script: 'aws ecr get-login-password --region us-east-1',
                        returnStdout: true
                    ).trim()
                }
            }
        }
        stage('Build & Push') {
            steps {
                sh '''
                    echo ${ECR_LOGIN} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker build -t ${ECR_REGISTRY}/myapp:${BUILD_NUMBER} .
                    docker push ${ECR_REGISTRY}/myapp:${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

---

#### Stage 5: Docker Container Deployment Fails

**Error:**
```
[Pipeline] sh 'docker run -d myapp'
docker: Error response from daemon: 
Conflict. The container name "/myapp" is already in use
```

**Diagnostic Process:**

```bash
# 1. Check existing containers
docker ps -a | grep myapp

# 2. Check if from previous failed deployment
docker logs myapp

# 3. Check Jenkins build history
# See if previous build left container running

# 4. Check container state
docker inspect myapp --format='{{.State.Status}}'

# 5. Check if port is in use
docker port myapp
sudo netstat -tulpn | grep <port>
```

**Cross-Tool Analysis:**

```bash
# Terraform may have created the initial container
terraform state list | grep docker_container

# Check if container is managed by Terraform
docker inspect myapp --format='{{.Config.Labels}}'
# Look for: terraform.io/managed=true

# If managed by Terraform, don't remove manually!
terraform state show docker_container.myapp
```

**Fix:**

```bash
# Option 1: Clean up in Jenkins pipeline (scripted)
pipeline {
    stages {
        stage('Deploy') {
            steps {
                sh '''
                    # Stop and remove old container if exists
                    docker stop myapp || true
                    docker rm myapp || true
                    
                    # Deploy new version
                    docker run -d --name myapp \
                      -p 8080:8080 \
                      myapp:${BUILD_NUMBER}
                '''
            }
        }
    }
}

# Option 2: Use unique names per build
pipeline {
    stages {
        stage('Deploy') {
            steps {
                sh '''
                    docker run -d --name myapp-${BUILD_NUMBER} \
                      -p $((8080 + BUILD_NUMBER)):8080 \
                      myapp:${BUILD_NUMBER}
                    
                    # Clean up old builds (keep last 3)
                    docker ps -a --format "{{.Names}}" | \
                      grep "myapp-" | \
                      sort -r | \
                      tail -n +4 | \
                      xargs -r docker rm -f
                '''
            }
        }
    }
}

# Option 3: Use Docker Compose with recreate
pipeline {
    stages {
        stage('Deploy') {
            steps {
                sh '''
                    # Update image tag in compose file
                    sed -i "s/myapp:.*/myapp:${BUILD_NUMBER}/" docker-compose.yml
                    
                    # Recreate containers
                    docker-compose up -d --force-recreate
                '''
            }
        }
    }
}

# Option 4: Let Terraform manage Docker containers
# Update Terraform with new image tag:
pipeline {
    stages {
        stage('Update Infrastructure') {
            steps {
                sh '''
                    cd terraform/
                    
                    # Update variable
                    cat > terraform.tfvars <<EOF
app_image = "myapp:${BUILD_NUMBER}"
EOF
                    
                    # Apply Terraform
                    terraform init
                    terraform apply -auto-approve
                '''
            }
        }
    }
}

# In Terraform:
resource "docker_container" "myapp" {
  name  = "myapp"
  image = var.app_image
  
  ports {
    internal = 8080
    external = 8080
  }
  
  # Force replacement on image change
  lifecycle {
    replace_triggered_by = [
      docker_image.myapp.id
    ]
  }
}
```

---

### Scenario 2: State File Corruption Across Tools

**Context:** Terraform state corrupted, affecting both Jenkins deployments and running Docker containers.

**Problem Chain:**
```
Terraform state corrupt → Can't manage infrastructure → Jenkins can't deploy → Docker containers orphaned
```

**Detection:**

```bash
# 1. Terraform shows errors
terraform plan
# Error: Error loading state: unexpected EOF

# 2. Jenkins builds fail
[Pipeline] sh 'terraform plan'
# Error: Error loading state

# 3. Docker containers running but not in Terraform state
docker ps
# Shows containers that Terraform should manage

terraform state list | grep docker_container
# Returns nothing or incomplete list
```

**Diagnostic Process:**

```bash
# 1. Verify Terraform state file integrity
terraform state pull | jq '.'
echo $?  # Non-zero = corrupted

# 2. Check backup state file
ls -la terraform.tfstate.backup
cat terraform.tfstate.backup | jq '.'

# 3. Check S3 versioning (if using S3 backend)
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/terraform.tfstate

# 4. Identify missing resources
# List actual Docker containers
docker ps --format "{{.Names}}"

# Compare with Terraform state
terraform state list

# Find orphaned containers
comm -23 \
  <(docker ps --format "{{.Names}}" | sort) \
  <(terraform state list | grep docker_container | cut -d. -f2 | sort)
```

**Recovery Strategy:**

```bash
# Step 1: Restore from backup
# Option A: Local backup
cp terraform.tfstate.backup terraform.tfstate

# Option B: S3 version
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/terraform.tfstate \
  --query 'Versions[0].VersionId' --output text

aws s3api get-object \
  --bucket my-terraform-state \
  --key prod/terraform.tfstate \
  --version-id <VERSION_ID> \
  terraform.tfstate.restored

mv terraform.tfstate.restored terraform.tfstate

# Step 2: Validate restored state
terraform state pull | jq '.version'

# Step 3: Refresh state with actual infrastructure
terraform refresh

# Step 4: Import missing Docker containers
# For each orphaned container:
docker ps --format "{{.Names}}" | while read container; do
    if ! terraform state list | grep -q "docker_container.${container}"; then
        echo "Importing ${container}"
        docker inspect ${container} --format='{{.Id}}' | \
          xargs terraform import docker_container.${container}
    fi
done

# Step 5: Verify in Jenkins
# Update Jenkinsfile to check state health:
pipeline {
    stages {
        stage('Verify Terraform State') {
            steps {
                sh '''
                    # Check state integrity
                    terraform state pull | jq empty
                    if [ $? -ne 0 ]; then
                        echo "ERROR: Terraform state is corrupted!"
                        exit 1
                    fi
                    
                    # Check for drift
                    terraform plan -detailed-exitcode
                    EXIT_CODE=$?
                    
                    if [ $EXIT_CODE -eq 1 ]; then
                        echo "ERROR: Terraform plan failed"
                        exit 1
                    elif [ $EXIT_CODE -eq 2 ]; then
                        echo "WARNING: Infrastructure drift detected"
                        terraform plan -no-color > drift-report.txt
                        # Alert team
                    fi
                '''
            }
        }
    }
}
```

---

### Scenario 3: Cross-Tool Authentication Cascade Failure

**Context:** AWS credentials expired, affecting Terraform, Jenkins AWS CLI calls, and Docker ECR pushes.

**Failure Cascade:**
```
AWS credentials expire 
  ↓
Terraform can't access S3 backend (state)
  ↓
Jenkins can't run Terraform
  ↓
Jenkins can't push to ECR
  ↓
Docker containers can't be deployed
```

**Detection:**

```bash
# 1. Terraform shows auth error
terraform init
# Error: NoCredentialProviders: no valid providers in chain

# 2. Jenkins build fails
[Pipeline] sh 'terraform init'
# Error: error configuring Terraform AWS Provider

# 3. Docker ECR push fails
[Pipeline] sh 'docker push <ecr-repo>'
# Error: no basic auth credentials
```

**Unified Diagnostic:**

```bash
# 1. Test AWS credentials directly
aws sts get-caller-identity
# Error: The security token included in the request is expired

# 2. Check credential source
echo $AWS_PROFILE
echo $AWS_ACCESS_KEY_ID
cat ~/.aws/credentials

# 3. Check in Jenkins
# Manage Jenkins → Script Console:
println System.getenv('AWS_ACCESS_KEY_ID')
println System.getenv('AWS_SECRET_ACCESS_KEY')

# 4. Check in Docker context
docker run --rm \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  amazon/aws-cli sts get-caller-identity

# 5. Check Terraform provider config
grep -A 10 "provider \"aws\"" main.tf
```

**Unified Fix:**

```bash
# Fix 1: Rotate credentials in all places

# A. Update local credentials
aws configure
# Enter new access key and secret key

# B. Update Jenkins credentials
# Manage Jenkins → Manage Credentials → Update 'aws-prod-credentials'

# C. Test Terraform
terraform init
terraform plan

# D. Test Docker ECR login
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ecr-repo>

# Fix 2: Use IAM roles instead (better practice)

# A. For Terraform (on EC2)
# Attach IAM role to Jenkins master/agents
resource "aws_iam_role" "jenkins" {
  name = "jenkins-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_instance_profile" "jenkins" {
  name = "jenkins-profile"
  role = aws_iam_role.jenkins.name
}

resource "aws_instance" "jenkins" {
  iam_instance_profile = aws_iam_instance_profile.jenkins.name
  # ...
}

# B. Update Terraform provider (remove hardcoded creds)
provider "aws" {
  region = "us-east-1"
  # No access_key or secret_key - will use IAM role
}

# C. Update Jenkinsfile (remove credential references)
pipeline {
    stages {
        stage('Terraform') {
            steps {
                sh '''
                    # Will use IAM role automatically
                    terraform init
                    terraform apply -auto-approve
                '''
            }
        }
        stage('Docker Push') {
            steps {
                sh '''
                    # Will use IAM role automatically
                    aws ecr get-login-password | \
                      docker login --username AWS --password-stdin ${ECR_REPO}
                    docker push ${ECR_REPO}/myapp:${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

---

## 🛠️ Unified Diagnostic Commands

### One-Command Health Check Across All Tools

```bash
#!/bin/bash
# devops-health-check.sh

echo "=== DevOps Stack Health Check ==="
echo ""

# Docker
echo "1. Docker Status:"
docker version &>/dev/null && echo "  ✓ Docker daemon running" || echo "  ✗ Docker daemon not accessible"
docker ps -q &>/dev/null && echo "  ✓ Containers accessible" || echo "  ✗ Cannot list containers"
df -h / | awk 'NR==2 {print "  Disk: " $5 " used"}'
echo ""

# Terraform
echo "2. Terraform Status:"
terraform version &>/dev/null && echo "  ✓ Terraform installed" || echo "  ✗ Terraform not found"
if [ -f terraform.tfstate ]; then
    terraform state pull | jq empty &>/dev/null && echo "  ✓ State file valid" || echo "  ✗ State file corrupted"
    echo "  Resources: $(terraform state list 2>/dev/null | wc -l)"
fi
echo ""

# Jenkins
echo "3. Jenkins Status:"
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 2>/dev/null | \
  grep -q "200\|403" && echo "  ✓ Jenkins responding" || echo "  ✗ Jenkins not accessible"
systemctl is-active jenkins &>/dev/null && echo "  ✓ Jenkins service running" || echo "  ✗ Jenkins service not running"
echo ""

# AWS Credentials
echo "4. AWS Credentials:"
aws sts get-caller-identity &>/dev/null && echo "  ✓ AWS credentials valid" || echo "  ✗ AWS credentials invalid"
echo ""

# Network
echo "5. Network Connectivity:"
ping -c 1 8.8.8.8 &>/dev/null && echo "  ✓ Internet reachable" || echo "  ✗ No internet"
curl -s -o /dev/null https://github.com && echo "  ✓ GitHub reachable" || echo "  ✗ GitHub unreachable"
curl -s -o /dev/null https://registry.terraform.io && echo "  ✓ Terraform registry reachable" || echo "  ✗ Registry unreachable"
echo ""

echo "=== Health Check Complete ==="
```

---

## 📚 Cross-Tool Quick Reference

| When This Happens | Check Docker | Check Terraform | Check Jenkins |
|-------------------|--------------|-----------------|---------------|
| **Build fails** | `docker logs <id>` | `terraform plan` | Console Output |
| **Can't connect** | `docker inspect <id>` | `terraform refresh` | Manage Nodes |
| **State issues** | `docker ps -a` | `terraform state list` | Build History |
| **Auth fails** | `docker login` | `aws sts get-caller-identity` | Manage Credentials |
| **Resource exhaustion** | `docker stats` | `terraform plan` output size | System Information |
| **Version conflicts** | Image tags | Provider versions | Plugin versions |

---

## 💡 Best Practices for Integrated Troubleshooting

1. **Enable Centralized Logging**
   ```bash
   # Send all logs to same place
   # Docker → CloudWatch/Splunk
   # Terraform → S3/logging
   # Jenkins → Splunk/ELK
   ```

2. **Consistent Tagging/Naming**
   ```hcl
   # Terraform
   tags = {
     ManagedBy = "terraform"
     Jenkins_Job = var.jenkins_job_name
     Build_Number = var.jenkins_build_number
   }
   ```

3. **Unified Monitoring**
   ```bash
   # Monitor all three stacks together
   # Grafana dashboard showing:
   # - Docker container health
   # - Terraform drift detection
   # - Jenkins build success rate
   ```

4. **State Management Hierarchy**
   ```
   Source of Truth: Terraform State
   ↓
   Deployed By: Jenkins Pipeline
   ↓
   Running As: Docker Containers
   ```

5. **Automated Health Checks**
   ```groovy
   // In Jenkins pipeline
   stage('Health Check') {
       steps {
           sh './devops-health-check.sh'
       }
   }
   ```

---

**Remember**: Most issues span multiple tools. When troubleshooting, think about the entire pipeline, not just individual components. The error you see in Docker might have originated in Terraform, triggered by Jenkins!
