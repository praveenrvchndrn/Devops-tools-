# Jenkins Troubleshooting Systematic Guide

## 🎯 Troubleshooting Workflow

```
1. Identify failure point in pipeline
2. Check console output logs
3. Verify Jenkins system health
4. Check plugin compatibility
5. Validate credentials/secrets
6. Test node connectivity
7. Fix and re-run
```

---

## 📋 Essential Diagnostic Commands

### Quick Health Check
```bash
# Check Jenkins service status
sudo systemctl status jenkins
sudo service jenkins status

# Check Jenkins logs
sudo tail -f /var/log/jenkins/jenkins.log
sudo journalctl -u jenkins -f

# Check disk space (common issue)
df -h /var/lib/jenkins

# Check Java process
ps aux | grep jenkins
jps -v

# Test Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ who-am-i

# Check available executors
curl -s http://localhost:8080/computer/api/json | jq '.computer[] | {displayName, offline}'
```

---

## 🔥 Common Issues & Troubleshooting

### 1. Pipeline Fails at Checkout Stage

**Symptoms:**
- "ERROR: Couldn't find any revision to build"
- "Permission denied (publickey)"
- "fatal: unable to access 'https://github.com/...': Could not resolve host"

**Diagnostic - Console Output:**
```groovy
// In Jenkins job, check "Console Output"
// Look for these patterns:

[Pipeline] checkout
ERROR: Error cloning remote repo 'origin'
hudson.plugins.git.GitException: Command "git fetch" returned status code 128
stderr: Permission denied (publickey)

// Or DNS issues:
fatal: unable to access 'https://github.com/user/repo.git/': 
Could not resolve host: github.com
```

**Diagnostic Commands:**
```bash
# On Jenkins master/agent node
# Test SSH key
ssh -T git@github.com

# Test HTTPS access
curl -I https://github.com

# Check Git installation
which git
git --version

# Check SSH config
cat ~/.ssh/config
ls -la ~/.ssh/

# Check known_hosts
cat ~/.ssh/known_hosts | grep github

# Test as jenkins user
sudo -u jenkins ssh -T git@github.com
```

**What to Check in Jenkins UI:**
1. **Manage Jenkins → Credentials**
   - Verify SSH key is added
   - Check key format (should be private key, not public)
   - Verify credential ID matches Jenkinsfile

2. **Job Configuration → Source Code Management**
   - Verify repository URL is correct
   - Check branch name (case sensitive)
   - Verify credentials selected

3. **Manage Jenkins → Script Console** (for debugging)
```groovy
// Test Git from Jenkins context
def sout = new StringBuilder()
def serr = new StringBuilder()
def proc = 'git ls-remote https://github.com/user/repo.git'.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitFor()
println "out: ${sout}"
println "err: ${serr}"
```

**Practice Scenario:**
```groovy
// Jenkinsfile with Git checkout
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/master']],
                    userRemoteConfigs: [[
                        url: 'git@github.com:myorg/myrepo.git',
                        credentialsId: 'github-ssh-key'  // Must exist in Jenkins
                    ]]
                ])
            }
        }
    }
}

// Common failures:
// 1. credentialsId doesn't exist → Add in Manage Jenkins → Credentials
// 2. SSH key not added to GitHub → Add deploy key or personal SSH key
// 3. Branch doesn't exist → Change to correct branch name
// 4. Network proxy blocks GitHub → Configure Git proxy
```

**Fix Examples:**
```bash
# Fix 1: Add SSH key to jenkins user
sudo su - jenkins
ssh-keygen -t rsa -b 4096 -C "jenkins@yourdomain.com"
cat ~/.ssh/id_rsa.pub
# Copy public key to GitHub → Settings → SSH Keys

# Fix 2: Add GitHub to known_hosts
sudo -u jenkins ssh-keyscan github.com >> /var/lib/jenkins/.ssh/known_hosts

# Fix 3: Configure Git for HTTPS with token
git config --global credential.helper store
echo "https://token:ghp_xxxxxxxxxxxx@github.com" > ~/.git-credentials

# Fix 4: Test from Jenkins
# Manage Jenkins → Script Console
println "git ls-remote https://github.com/user/repo.git".execute().text
```

---

### 2. Build Fails - "Tool Not Found"

**Symptoms:**
- "mvn: command not found"
- "npm: not found"
- "docker: command not found"
- "java.io.IOException: Cannot run program 'docker'"

**Diagnostic - Console Output:**
```
[Pipeline] sh
+ docker build -t myapp .
/var/lib/jenkins/workspace/myapp@tmp/durable-xxx/script.sh: line 1: 
docker: command not found
```

**Diagnostic Commands:**
```bash
# Check what tools are available on Jenkins node
# In Manage Jenkins → Script Console:
println "PATH: ${System.getenv('PATH')}"
println "which docker: ${'which docker'.execute().text}"
println "which mvn: ${'which mvn'.execute().text}"
println "which npm: ${'which npm'.execute().text}"

# Or via SSH to Jenkins agent
ssh jenkins-agent-01
which docker
which mvn
which node
which python3

# Check tool configuration
# Manage Jenkins → Global Tool Configuration
# Verify Maven/Node/JDK installations
```

**What to Check in Jenkins UI:**

1. **Manage Jenkins → Global Tool Configuration**
   - Check if tools are configured
   - Verify installation paths
   - Check automatic installer settings

2. **Pipeline → Tools Block**
```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.8.6'  // Must match name in Global Tool Config
        jdk 'JDK-11'
        nodejs 'NodeJS-18'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn --version'
                sh 'node --version'
            }
        }
    }
}
```

3. **Check Node/Agent Configuration**
   - Manage Jenkins → Manage Nodes → Agent → Configure
   - Check Environment Variables
   - Check Tools Location

**Practice Scenario:**
```groovy
// Pipeline using tools
pipeline {
    agent { label 'docker-agent' }
    
    tools {
        maven 'Maven-3.8.6'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
                sh 'docker build -t myapp .'
            }
        }
    }
}

// Failure 1: Maven not found
// Solution: Add Maven in Global Tool Configuration
// Manage Jenkins → Global Tool Configuration → Maven
// Name: Maven-3.8.6
// Install automatically from Apache

// Failure 2: Docker not found
// Solution: Install Docker on agent OR use docker:dind container
docker run -d --privileged --name jenkins-agent \
  -v jenkins_home:/var/jenkins_home \
  -e DOCKER_HOST=tcp://docker:2376 \
  jenkins/agent

// Or install Docker on host
sudo apt-get install docker.io
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

**Fix Patterns:**
```bash
# Fix 1: Add tool to PATH (on Jenkins agent)
sudo vi /etc/systemd/system/jenkins.service
# Add to Environment:
Environment="PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/go/bin"

sudo systemctl daemon-reload
sudo systemctl restart jenkins

# Fix 2: Install missing tool
# For Docker:
sudo apt-get update
sudo apt-get install docker.io
sudo usermod -aG docker jenkins

# For Maven:
sudo apt-get install maven
# Or download and extract to /opt/maven

# Fix 3: Use Docker agent with tools pre-installed
pipeline {
    agent {
        docker {
            image 'maven:3.8.6-jdk-11'
            args '-v $HOME/.m2:/root/.m2'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

---

### 3. Plugin Compatibility Issues

**Symptoms:**
- "Plugin X requires Jenkins Y or later"
- "java.lang.NoClassDefFoundError"
- "Plugin failed to load: ... incompatible"
- Jenkins UI becomes unresponsive after plugin update

**Diagnostic - Jenkins Logs:**
```bash
# Check Jenkins system log
# Manage Jenkins → System Log

# Or via filesystem:
sudo tail -f /var/log/jenkins/jenkins.log

# Look for patterns:
grep -i "plugin" /var/log/jenkins/jenkins.log | grep -i "error\|failed\|incompatible"

# Common log patterns:
SEVERE: Failed Loading plugin X vY.Z
java.lang.UnsupportedClassVersionError: Plugin compiled with Java 11 but running Java 8

WARNING: Plugin A vX depends on Plugin B vY but found vZ
```

**Diagnostic Commands:**
```bash
# List installed plugins via CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ list-plugins

# Check plugin versions via API
curl -s http://localhost:8080/pluginManager/api/json?depth=1 | \
  jq '.plugins[] | {shortName, version, hasUpdate, enabled}'

# Check Jenkins version
curl -s http://localhost:8080/api/json | jq '.version'

# Find plugin directory
ls -la /var/lib/jenkins/plugins/

# Check specific plugin manifest
unzip -p /var/lib/jenkins/plugins/git.jpi META-INF/MANIFEST.MF

# Check for .jpi.disabled or .hpi.disabled
ls -la /var/lib/jenkins/plugins/*.disabled
```

**What to Check in Jenkins UI:**

1. **Manage Jenkins → Manage Plugins → Installed**
   - Look for red warnings about dependency issues
   - Check for plugins with "Uninstall" in red

2. **Manage Jenkins → System Information**
   - Check Jenkins version
   - Check Java version
   - Check OS version

3. **Manage Jenkins → Plugin Manager → Available/Updates**
   - Check if updates available
   - Look for dependency chains

**Practice Scenario:**
```bash
# Scenario: Updated Pipeline plugin, now builds fail

# Error in console:
java.lang.NoSuchMethodError: 
org.jenkinsci.plugins.workflow.cps.CpsFlowExecution.loadProgramAsync

# Diagnostic:
# 1. Check Pipeline plugin version
curl -s http://localhost:8080/pluginManager/api/json?depth=1 | \
  jq '.plugins[] | select(.shortName=="workflow-aggregator")'

# Output shows: version 2.6, but requires Pipeline: Groovy v2.80

# 2. Check dependencies
cat /var/lib/jenkins/plugins/workflow-aggregator/META-INF/MANIFEST.MF | \
  grep "Plugin-Dependencies"

# Fix:
# Option 1: Update all dependencies via UI
# Manage Jenkins → Manage Plugins → Updates → Select All → Download and Install

# Option 2: Downgrade problematic plugin
cd /var/lib/jenkins/plugins/
rm workflow-aggregator.jpi
wget https://updates.jenkins.io/download/plugins/workflow-aggregator/2.5/workflow-aggregator.hpi
mv workflow-aggregator.hpi workflow-aggregator.jpi
sudo systemctl restart jenkins

# Option 3: Safe restart with plugin backup
cd /var/lib/jenkins/plugins/
cp -r . ~/jenkins-plugins-backup-$(date +%Y%m%d)
# Then update plugins
```

**Recovery from Bad Plugin:**
```bash
# If Jenkins won't start after plugin update:

# 1. Disable problematic plugin
cd /var/lib/jenkins/plugins/
mv problematic-plugin.jpi problematic-plugin.jpi.disabled

# 2. Restart Jenkins
sudo systemctl restart jenkins

# 3. Check if it starts
sudo systemctl status jenkins
curl -I http://localhost:8080

# 4. If starts, remove plugin via UI or CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ \
  disable-plugin problematic-plugin

# 5. Or remove completely
rm /var/lib/jenkins/plugins/problematic-plugin.jpi.disabled
rm -rf /var/lib/jenkins/plugins/problematic-plugin/
```

---

### 4. Out of Memory / Disk Space Issues

**Symptoms:**
- "java.lang.OutOfMemoryError: Java heap space"
- "java.lang.OutOfMemoryError: GC overhead limit exceeded"
- "No space left on device"
- Builds stuck in queue
- Jenkins becomes unresponsive

**Diagnostic - Logs:**
```bash
# Check for OOM in logs
sudo grep -i "OutOfMemoryError" /var/log/jenkins/jenkins.log

# Check disk space
df -h /var/lib/jenkins
du -sh /var/lib/jenkins/workspace/*
du -sh /var/lib/jenkins/builds/*

# Check memory usage
free -h
ps aux | grep jenkins | grep -v grep
jstat -gc $(pgrep -f jenkins) 1000 10

# Check Java heap settings
ps aux | grep jenkins | grep -oP '\-Xmx\S+'
```

**What to Check in Jenkins UI:**

1. **Manage Jenkins → System Information**
   - Check available disk space
   - Check memory usage
   - Check Java version and options

2. **Manage Jenkins → Manage Nodes and Clouds**
   - Check executor status
   - Check workspace disk usage per agent

3. **Build History**
   - Check for stuck builds
   - Check for builds consuming excessive resources

**Diagnostic Commands:**
```bash
# Find large workspaces
du -sh /var/lib/jenkins/workspace/* | sort -rh | head -20

# Find old builds
find /var/lib/jenkins/jobs/*/builds/* -type d -mtime +30

# Check Java heap usage
jmap -heap $(pgrep -f jenkins)

# Monitor in real-time
watch -n 5 'df -h /var/lib/jenkins && free -h'

# Check inode usage (can be issue even with space)
df -i /var/lib/jenkins
```

**Practice Scenario:**
```bash
# Scenario: Jenkins running out of disk space

# Diagnostic:
df -h /var/lib/jenkins
# Output: /dev/sda1  100G   98G  2.0G  98% /var/lib/jenkins

# Find culprits:
du -sh /var/lib/jenkins/* | sort -rh | head -10
# Output:
# 45G    /var/lib/jenkins/workspace
# 30G    /var/lib/jenkins/jobs
# 15G    /var/lib/jenkins/logs
# 5G     /var/lib/jenkins/plugins

# Check workspaces:
du -sh /var/lib/jenkins/workspace/* | sort -rh | head -10
# Output:
# 20G    /var/lib/jenkins/workspace/build-docker-images
# 15G    /var/lib/jenkins/workspace/integration-tests
# 10G    /var/lib/jenkins/workspace/build-artifacts

# Clean up:
# 1. Delete old workspaces
cd /var/lib/jenkins/workspace/
rm -rf build-docker-images@* # Remove parallel builds
rm -rf *@tmp # Remove temp directories

# 2. Clean old builds
cd /var/lib/jenkins/jobs/
find . -name "builds" -type d -exec du -sh {} \; | sort -rh | head -10

# 3. Configure build discarder in Jenkinsfile:
pipeline {
    options {
        buildDiscarder(logRotator(
            daysToKeepStr: '30',
            numToKeepStr: '10',
            artifactDaysToKeepStr: '7',
            artifactNumToKeepStr: '5'
        ))
    }
}

# 4. Add cleanup stage:
stage('Cleanup') {
    steps {
        cleanWs()  // Clean workspace after build
    }
}
```

**Memory Issues Fix:**
```bash
# Increase Java heap size
sudo vi /etc/default/jenkins

# Add/modify:
JAVA_ARGS="-Xmx4096m -Xms2048m -XX:MaxPermSize=512m"

# Or in systemd service:
sudo vi /etc/systemd/system/jenkins.service

# Modify ExecStart:
Environment="JAVA_OPTS=-Djava.awt.headless=true -Xmx4g -Xms2g"

sudo systemctl daemon-reload
sudo systemctl restart jenkins

# Verify:
ps aux | grep jenkins | grep Xmx
```

**Automated Cleanup Script:**
```bash
#!/bin/bash
# jenkins-cleanup.sh

JENKINS_HOME="/var/lib/jenkins"
DAYS_OLD=30

echo "=== Jenkins Cleanup Started ==="

# Clean old workspaces
find $JENKINS_HOME/workspace -maxdepth 1 -mtime +$DAYS_OLD -type d \
  -exec rm -rf {} \; 2>/dev/null

# Clean old builds
find $JENKINS_HOME/jobs/*/builds -maxdepth 1 -mtime +$DAYS_OLD -type d \
  -exec rm -rf {} \; 2>/dev/null

# Clean temp directories
find $JENKINS_HOME/workspace -name "*@tmp" -type d -exec rm -rf {} \; 2>/dev/null

# Clean Docker build cache (if using Docker)
docker system prune -af --volumes --filter "until=720h"

echo "=== Cleanup Complete ==="
df -h $JENKINS_HOME

# Add to cron:
# 0 2 * * 0 /usr/local/bin/jenkins-cleanup.sh > /var/log/jenkins-cleanup.log 2>&1
```

---

### 5. Credential & Secret Issues

**Symptoms:**
- "hudson.util.Secret$DecryptionException"
- "Credentials not found"
- "java.lang.NullPointerException" when accessing credentials
- AWS CLI commands fail with "Unable to locate credentials"

**Diagnostic - Console Output:**
```
[Pipeline] withCredentials
ERROR: Credentials 'aws-prod-credentials' not found
hudson.remoting.ProxyException: com.cloudbees.plugins.credentials.CredentialsUnavailableException
```

**Diagnostic Commands:**
```bash
# List all credentials (requires admin access)
# Manage Jenkins → Script Console:
import com.cloudbees.plugins.credentials.*
import com.cloudbees.plugins.credentials.domains.*

def creds = CredentialsProvider.lookupCredentials(
    StandardCredentials.class,
    Jenkins.instance,
    null,
    null
)

creds.each { c ->
    println "ID: ${c.id}"
    println "Description: ${c.description}"
    println "Class: ${c.class}"
    println "---"
}

# Check credential file (encrypted)
sudo ls -la /var/lib/jenkins/credentials.xml
sudo cat /var/lib/jenkins/credentials.xml

# Check secret key
sudo ls -la /var/lib/jenkins/secrets/

# Verify credentials in pipeline
# Add debug step:
```

```groovy
stage('Debug Credentials') {
    steps {
        script {
            try {
                withCredentials([
                    string(credentialsId: 'api-key', variable: 'KEY')
                ]) {
                    sh 'echo "Credential loaded successfully"'
                    sh 'echo ${KEY} | cut -c1-4'  // Show first 4 chars only
                }
            } catch (Exception e) {
                error "Failed to load credentials: ${e.message}"
            }
        }
    }
}
```

**What to Check in Jenkins UI:**

1. **Manage Jenkins → Manage Credentials**
   - Verify credential exists
   - Check credential ID matches Jenkinsfile
   - Check credential type (String/File/Username with password)
   - Check credential scope (System/Global)

2. **Job Configuration → Build Environment**
   - Verify "Use secret text(s) or file(s)" is configured correctly

3. **Pipeline Syntax Generator**
   - Use "withCredentials" snippet generator
   - Verify generated code

**Practice Scenario:**
```groovy
// Jenkinsfile with credential issues
pipeline {
    agent any
    
    environment {
        // WRONG: Trying to use credential directly
        API_KEY = credentials('api-key')  // This returns [username:password] format
    }
    
    stages {
        stage('Deploy') {
            steps {
                // Fails: API_KEY is not in expected format
                sh 'curl -H "Authorization: Bearer ${API_KEY}" api.example.com'
            }
        }
    }
}

// CORRECT way:
pipeline {
    agent any
    
    stages {
        stage('Deploy') {
            steps {
                withCredentials([
                    string(credentialsId: 'api-key', variable: 'API_KEY')
                ]) {
                    sh 'curl -H "Authorization: Bearer ${API_KEY}" api.example.com'
                }
            }
        }
    }
}

// For AWS credentials:
pipeline {
    agent any
    
    stages {
        stage('Deploy to AWS') {
            steps {
                withCredentials([
                    aws(credentialsId: 'aws-prod-credentials', 
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        aws s3 ls
                        aws ec2 describe-instances
                    '''
                }
            }
        }
    }
}
```

**Fix Patterns:**
```bash
# Fix 1: Add missing credential
# Via UI: Manage Jenkins → Manage Credentials → (select domain) → Add Credentials

# Via CLI:
java -jar jenkins-cli.jar -s http://localhost:8080/ \
  create-credentials-by-xml system::system::jenkins \
  < credential.xml

# Fix 2: Fix credential scope
# If credential is in "System" scope but pipeline needs "Global":
# Manage Jenkins → Manage Credentials → Update scope

# Fix 3: Re-encrypt credentials after master key change
# If you restored from backup and credentials don't work:
cd /var/lib/jenkins/secrets/
sudo cp master.key master.key.backup
# Restore original master.key
# Or regenerate all credentials

# Fix 4: Debug credential binding
# Add to Jenkinsfile:
stage('Debug') {
    steps {
        script {
            println "Available bindings:"
            binding.variables.each { k, v ->
                println "  ${k} = ${v?.class?.name}"
            }
        }
    }
}
```

---

### 6. Pipeline Syntax Errors

**Symptoms:**
- "WorkflowScript: ... unexpected token"
- "groovy.lang.MissingMethodException"
- "Expected a step" or "No such DSL method"

**Diagnostic - Console Output:**
```
WorkflowScript: 25: unexpected token: } @ line 25, column 9.
           }
           ^
```

**Diagnostic Approach:**

1. **Use Blue Ocean UI** - Better error visualization
2. **Pipeline Syntax Generator** - Generate correct syntax
3. **Replay Feature** - Edit and rerun without commit

**Common Syntax Issues:**

#### A. Missing 'steps' Block
```groovy
// WRONG:
pipeline {
    agent any
    stages {
        stage('Build') {
            sh 'mvn clean package'  // Missing steps block!
        }
    }
}

// CORRECT:
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

#### B. Incorrect Script Block Usage
```groovy
// WRONG: Using script block for everything
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    sh 'mvn clean package'  // Don't need script for simple sh
                }
            }
        }
    }
}

// CORRECT: Use script only when needed
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Deploy') {
            steps {
                script {
                    // Use script for complex Groovy logic
                    if (env.BRANCH_NAME == 'main') {
                        sh 'deploy-to-prod.sh'
                    }
                }
            }
        }
    }
}
```

#### C. String Interpolation Issues
```groovy
// WRONG: Using wrong quotes
stage('Build') {
    steps {
        sh 'echo ${BUILD_NUMBER}'  // Won't interpolate with single quotes!
    }
}

// CORRECT:
stage('Build') {
    steps {
        sh "echo ${BUILD_NUMBER}"  // Use double quotes for variable expansion
    }
}

// Or use triple quotes for multi-line:
stage('Build') {
    steps {
        sh """
            echo "Build number: ${BUILD_NUMBER}"
            echo "Branch: ${BRANCH_NAME}"
        """
    }
}
```

**Validation Tools:**
```bash
# Validate Jenkinsfile locally (requires jenkins-cli)
java -jar jenkins-cli.jar -s http://localhost:8080/ \
  declarative-linter < Jenkinsfile

# Or via API
curl -X POST \
  -F "jenkinsfile=<Jenkinsfile" \
  http://localhost:8080/pipeline-model-converter/validate

# Use Replay feature in Jenkins UI to test changes without commit
```

---

### 7. Agent/Node Connection Issues

**Symptoms:**
- "Agent went offline during the build"
- "ERROR: Connection terminated"
- "java.net.SocketException: Connection reset"
- Build stays in queue forever

**Diagnostic - Jenkins UI:**
```
Manage Jenkins → Manage Nodes and Clouds → [Node Name] → Log
```

**Common Log Patterns:**
```
java.net.ConnectException: Connection refused (Connection refused)
hudson.remoting.ChannelClosedException: Channel is already closed
SSHException: Connection refused
```

**Diagnostic Commands:**
```bash
# Check agent connectivity from master
telnet agent-01.example.com 22
ssh jenkins@agent-01.example.com

# Check from agent to master
curl -I http://jenkins-master:8080

# Check JNLP agent connection (for Windows/Docker agents)
java -jar agent.jar -jnlpUrl http://jenkins:8080/computer/agent-01/slave-agent.jnlp

# Check agent process on agent node
ps aux | grep java | grep agent

# Check agent logs on agent
tail -f /var/log/jenkins-agent/agent.log

# Check network
ping jenkins-master.example.com
traceroute jenkins-master.example.com
```

**What to Check:**

1. **Master Node Configuration**
   - Manage Jenkins → Configure System → Jenkins URL (must be accessible from agents)
   - Check if firewall allows agent connections

2. **Agent Node Configuration**
   - Manage Jenkins → Manage Nodes → [Node] → Configure
   - Verify hostname/IP
   - Check credentials
   - Verify remote root directory exists and is writable

3. **Agent Status**
   - Look for red X or disconnected status
   - Check "Log" link for error messages

**Fix Patterns:**
```bash
# Fix 1: Restart agent from Jenkins UI
# Manage Jenkins → Manage Nodes → [Node] → Disconnect → Launch agent

# Fix 2: Restart agent service on agent node
sudo systemctl restart jenkins-agent
# Or for JNLP agent:
sudo systemctl restart jenkins-agent-jnlp

# Fix 3: Re-register agent
# On agent node:
curl -O http://jenkins-master:8080/jnlpJars/agent.jar
java -jar agent.jar \
  -jnlpUrl http://jenkins-master:8080/computer/agent-01/slave-agent.jnlp \
  -secret <secret-from-UI> \
  -workDir "/var/lib/jenkins-agent"

# Fix 4: Fix SSH connectivity
# From master:
ssh-copy-id jenkins@agent-01
# Test:
ssh jenkins@agent-01 'echo "Connection OK"'

# Fix 5: Increase connection timeout
# Manage Jenkins → Configure System → Global properties
# Add environment variable:
JENKINS_OPTS="-Dhudson.remoting.Launcher.pingTimeoutSec=600"
```

---

### 8. Docker-in-Docker Pipeline Issues

**Symptoms:**
- "Cannot connect to the Docker daemon at unix:///var/run/docker.sock"
- "permission denied while trying to connect to the Docker daemon socket"
- "docker: command not found" inside Docker agent

**Diagnostic - Console Output:**
```
[Pipeline] docker
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. 
Is the docker daemon running?
```

**Diagnostic Commands:**
```bash
# On Jenkins host:
docker ps
groups jenkins
ls -la /var/run/docker.sock

# Test Docker access as jenkins user:
sudo -u jenkins docker ps

# Check Docker service:
sudo systemctl status docker
```

**Practice Scenario:**
```groovy
// Pipeline using Docker
pipeline {
    agent {
        docker {
            image 'maven:3.8.6-jdk-11'
            args '-v $HOME/.m2:/root/.m2'
        }
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
                
                // Build Docker image (Docker-in-Docker)
                sh 'docker build -t myapp .'  // Fails!
            }
        }
    }
}

// Error: Cannot connect to Docker daemon

// Fix Option 1: Mount Docker socket
pipeline {
    agent {
        docker {
            image 'maven:3.8.6-jdk-11'
            args '-v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.m2:/root/.m2'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'apt-get update && apt-get install -y docker.io'
                sh 'mvn clean package'
                sh 'docker build -t myapp .'  // Works now!
            }
        }
    }
}

// Fix Option 2: Use Docker-in-Docker (DinD)
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

// Fix Option 3: Use Kaniko (no Docker daemon needed)
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - /busybox/cat
    tty: true
'''
        }
    }
    stages {
        stage('Build Image') {
            steps {
                container('kaniko') {
                    sh '''
                        /kaniko/executor \
                          --context `pwd` \
                          --destination myregistry/myapp:${BUILD_NUMBER}
                    '''
                }
            }
        }
    }
}
```

---

## 🔍 Advanced Debugging Techniques

### Script Console Debugging
```groovy
// Manage Jenkins → Script Console

// 1. Check workspace contents
def job = Jenkins.instance.getItemByFullName('my-pipeline')
def build = job.lastBuild
println "Workspace: ${build.workspace}"
println "Files:"
build.workspace.list().each { file ->
    println "  - ${file.name}"
}

// 2. List running builds
Jenkins.instance.getAllItems(Job.class).each { job ->
    job.builds.each { build ->
        if (build.isBuilding()) {
            println "${job.name} #${build.number} - ${build.executor?.displayName}"
        }
    }
}

// 3. Clean up stuck builds
Jenkins.instance.getAllItems(Job.class).each { job ->
    job.builds.findAll { it.isBuilding() }.each { build ->
        println "Stopping ${job.name} #${build.number}"
        build.executor?.interrupt()
    }
}

// 4. Check plugin status
Jenkins.instance.pluginManager.plugins.findAll { 
    !it.enabled || it.hasUpdate()
}.each { plugin ->
    println "${plugin.shortName}: ${plugin.version} " +
            "(enabled: ${plugin.enabled}, update: ${plugin.hasUpdate()})"
}
```

### Pipeline Replay for Testing
```
1. Go to failed build
2. Click "Replay"
3. Edit Jenkinsfile directly in browser
4. Click "Run"
5. Test changes without committing
```

### Debug Logging in Pipeline
```groovy
pipeline {
    agent any
    
    options {
        timestamps()  // Add timestamps to console output
    }
    
    stages {
        stage('Debug') {
            steps {
                script {
                    println "=== Environment Variables ==="
                    sh 'env | sort'
                    
                    println "=== Build Info ==="
                    println "Build Number: ${env.BUILD_NUMBER}"
                    println "Job Name: ${env.JOB_NAME}"
                    println "Workspace: ${env.WORKSPACE}"
                    
                    println "=== System Info ==="
                    sh 'uname -a'
                    sh 'whoami'
                    sh 'pwd'
                    sh 'df -h'
                    sh 'free -h'
                }
            }
        }
    }
}
```

---

## 📊 Systematic Troubleshooting Checklist

When Jenkins build fails, follow this order:

```bash
# 1. Check console output
# (In Jenkins UI, click on build number → Console Output)

# 2. Identify failure stage
# Look for "FAILED" or red X in Blue Ocean UI

# 3. Check Jenkins system health
# Manage Jenkins → System Log
# Manage Jenkins → System Information

# 4. Check node status
# Manage Jenkins → Manage Nodes and Clouds

# 5. Check plugin health
# Manage Jenkins → Manage Plugins

# 6. Check credentials
# Manage Jenkins → Manage Credentials

# 7. Validate Jenkinsfile
java -jar jenkins-cli.jar declarative-linter < Jenkinsfile

# 8. Test on different agent
# Add: agent { label 'different-node' }

# 9. Check system logs
sudo tail -100 /var/log/jenkins/jenkins.log

# 10. Use replay to test fixes
# Build → Replay → Edit → Run
```

---

## 💡 Key Takeaways

1. **Console output first** - 90% of issues visible in build logs
2. **Check node health** - Agent disconnections are common
3. **Validate Jenkinsfile** - Use declarative-linter before commit
4. **Pin plugin versions** - Avoid surprise breakages
5. **Use withCredentials** - Never hardcode secrets
6. **Enable timestamps** - Makes debugging timeline clear
7. **Clean workspaces** - Prevent disk space issues
8. **Test with Replay** - No need to commit for testing
9. **Monitor disk/memory** - Resource exhaustion is common
10. **Use Blue Ocean** - Better visualization of failures

---

## 📚 Quick Reference Card

| Issue Type | First Check | Key Log Pattern |
|------------|-------------|-----------------|
| Git checkout fails | Console Output → checkout stage | Permission denied, Could not resolve |
| Tool not found | Global Tool Configuration | command not found |
| Plugin issue | System Log | Plugin failed to load |
| OOM/Disk full | System Information | OutOfMemoryError, No space left |
| Credential fails | Manage Credentials | Credentials not found |
| Syntax error | Blue Ocean UI | unexpected token @ line X |
| Agent offline | Manage Nodes | Connection refused, Channel closed |
| Docker issue | Console Output | Cannot connect to Docker daemon |

---

**Remember**: Jenkins errors are usually descriptive. Read the full console output. The error message tells you exactly what went wrong and often how to fix it.
