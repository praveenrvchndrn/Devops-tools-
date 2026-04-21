# EKS (Amazon Elastic Kubernetes Service) Troubleshooting Guide

## 🎯 Troubleshooting Workflow

```
1. Identify failure point (pod, node, service, ingress)
2. Check cluster health
3. Review pod/container logs
4. Verify resource availability
5. Check networking/DNS
6. Validate IAM permissions
7. Fix and verify
```

---

## 📋 Essential Diagnostic Commands

### Quick Health Check
```bash
# Check cluster status
aws eks describe-cluster --name my-cluster --query 'cluster.status'

# Check nodes
kubectl get nodes
kubectl get nodes -o wide

# Check pods across all namespaces
kubectl get pods -A

# Check pod status
kubectl get pods -n <namespace>

# Check services
kubectl get svc -A

# Check events (recent issues)
kubectl get events -A --sort-by='.lastTimestamp'

# Check cluster version
kubectl version --short

# Check cluster info
kubectl cluster-info
```

---

## 🔥 Common Issues & Troubleshooting

### 1. Pod Stuck in Pending State

**Symptoms:**
- Pod shows STATUS: Pending
- Pod never becomes Running
- Application not accessible

**Diagnostic Commands:**
```bash
# Check pod status
kubectl get pods -n production
# Shows: myapp-7d6c9f5b-xyz    0/1     Pending   0          5m

# Describe pod for detailed info
kubectl describe pod myapp-7d6c9f5b-xyz -n production

# Check pod events
kubectl get events -n production --field-selector involvedObject.name=myapp-7d6c9f5b-xyz

# Check node resources
kubectl top nodes
kubectl describe nodes

# Check if any taints preventing scheduling
kubectl describe nodes | grep -i taint
```

**What to Check in Describe Output:**
```yaml
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes available: insufficient memory
  Warning  FailedScheduling  0/3 nodes available: 1 node had taint {key: value}, that pod didn't tolerate
  Warning  FailedScheduling  0/3 nodes available: pod has unbound immediate PersistentVolumeClaims
```

**Common Reasons & Fixes:**

#### A. Insufficient Resources
```bash
# Diagnosis
kubectl describe pod myapp-7d6c9f5b-xyz -n production
# Events: "0/3 nodes are available: insufficient cpu"

# Check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"

# Fix 1: Reduce pod resource requests
kubectl edit deployment myapp -n production
# Reduce:
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          requests:
            memory: "256Mi"  # Was 2Gi
            cpu: "100m"      # Was 1000m

# Fix 2: Add more nodes
# Check current node group
aws eks describe-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup

# Update desired size
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --scaling-config desiredSize=5

# Verify new nodes
kubectl get nodes
```

#### B. Unbound PVC (PersistentVolumeClaim)
```bash
# Diagnosis
kubectl describe pod myapp-7d6c9f5b-xyz -n production
# Events: "pod has unbound immediate PersistentVolumeClaims"

# Check PVC status
kubectl get pvc -n production
# Shows: STATUS = Pending

# Describe PVC
kubectl describe pvc data-pvc -n production

# Common issues:
# 1. No StorageClass available
kubectl get storageclass

# Fix: Create StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
EOF

# 2. Insufficient EBS volume quota
aws service-quotas get-service-quota \
  --service-code ebs \
  --quota-code L-D18FCD1D

# Request quota increase if needed

# Delete and recreate pod to retry
kubectl delete pod myapp-7d6c9f5b-xyz -n production
```

#### C. Node Selector Not Matching
```bash
# Diagnosis
kubectl describe pod myapp-7d6c9f5b-xyz -n production
# Events: "0/3 nodes are available: 3 node(s) didn't match node selector"

# Check pod node selector
kubectl get pod myapp-7d6c9f5b-xyz -n production -o yaml | grep -A 5 nodeSelector

# Check node labels
kubectl get nodes --show-labels

# Fix: Add label to node
kubectl label nodes <node-name> disktype=ssd

# Or remove node selector from deployment
kubectl edit deployment myapp -n production
# Remove nodeSelector section
```

---

### 2. Pod CrashLoopBackOff

**Symptoms:**
- Pod STATUS: CrashLoopBackOff
- Pod restarts repeatedly
- RESTARTS count increasing

**Diagnostic Commands:**
```bash
# Check pod status
kubectl get pods -n production
# Shows: myapp-7d6c9f5b-xyz  0/1  CrashLoopBackOff  5  10m

# Check pod logs
kubectl logs myapp-7d6c9f5b-xyz -n production

# Check previous container logs
kubectl logs myapp-7d6c9f5b-xyz -n production --previous

# Describe pod for events
kubectl describe pod myapp-7d6c9f5b-xyz -n production

# Check all containers in pod
kubectl logs myapp-7d6c9f5b-xyz -n production --all-containers=true

# Follow logs in real-time
kubectl logs -f myapp-7d6c9f5b-xyz -n production
```

**What to Check in Logs:**
```bash
# Application errors
kubectl logs myapp-7d6c9f5b-xyz -n production | grep -i "error\|exception\|fatal"

# Exit codes
kubectl describe pod myapp-7d6c9f5b-xyz -n production | grep "Exit Code"
```

**Common Exit Codes:**
- **0**: Normal exit (but restarting = liveness probe failing)
- **1**: Application error
- **137**: OOMKilled (out of memory)
- **139**: Segmentation fault
- **143**: SIGTERM (graceful shutdown)

**Practice Scenario:**
```bash
# Create pod that crashes
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo Starting && exit 1"]
EOF

# Watch it crash
kubectl get pods crash-test --watch

# Check logs
kubectl logs crash-test

# Check describe
kubectl describe pod crash-test

# Clean up
kubectl delete pod crash-test
```

**Real-World Fixes:**

```bash
# Scenario 1: Application failing to connect to database

# Check logs
kubectl logs myapp-7d6c9f5b-xyz -n production
# Output: "Error: Unable to connect to database"

# Check database service
kubectl get svc -n production | grep postgres

# Test DNS resolution from pod
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup postgres-service

# Check database credentials
kubectl get secret db-credentials -n production -o yaml

# Fix: Update database connection string
kubectl edit deployment myapp -n production
# Update env vars:
- name: DATABASE_URL
  value: "postgresql://postgres-service:5432/mydb"

# Scenario 2: Container memory limit too low (OOMKilled)

# Check exit code
kubectl describe pod myapp-7d6c9f5b-xyz -n production
# Shows: Exit Code: 137 (OOMKilled)

# Check memory limit
kubectl get pod myapp-7d6c9f5b-xyz -n production -o yaml | grep -A 3 resources

# Increase memory limit
kubectl edit deployment myapp -n production
spec:
  template:
    spec:
      containers:
      - name: myapp
        resources:
          limits:
            memory: "1Gi"  # Was 256Mi
          requests:
            memory: "512Mi"

# Verify
kubectl rollout status deployment/myapp -n production

# Scenario 3: Liveness probe failing

# Check describe
kubectl describe pod myapp-7d6c9f5b-xyz -n production
# Events: "Liveness probe failed: HTTP probe failed with statuscode: 500"

# Check probe configuration
kubectl get deployment myapp -n production -o yaml | grep -A 10 livenessProbe

# Fix: Adjust probe settings
kubectl edit deployment myapp -n production
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 60  # Was 10 - app needs more time
  periodSeconds: 10
  failureThreshold: 5      # Was 3 - allow more retries
```

---

### 3. Service Not Accessible / Connection Refused

**Symptoms:**
- Cannot access application via service
- "Connection refused" or timeout
- LoadBalancer shows pending

**Diagnostic Commands:**
```bash
# Check service
kubectl get svc -n production
kubectl describe svc myapp-service -n production

# Check endpoints (are pods registered?)
kubectl get endpoints myapp-service -n production

# Check pod labels and service selector match
kubectl get pods -n production --show-labels
kubectl get svc myapp-service -n production -o yaml | grep -A 3 selector

# Test service from within cluster
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- bash
# Inside debug pod:
curl myapp-service.production.svc.cluster.local:8080

# Check LoadBalancer status (if type=LoadBalancer)
kubectl describe svc myapp-service -n production | grep "LoadBalancer Ingress"

# Check security groups (AWS)
aws ec2 describe-security-groups \
  --filters "Name=tag:kubernetes.io/cluster/my-cluster,Values=owned"
```

**Common Issues & Fixes:**

#### A. Service Selector Not Matching Pods
```bash
# Check service selector
kubectl get svc myapp-service -n production -o yaml
# Shows: selector: app=myapp

# Check pod labels
kubectl get pods -n production --show-labels
# Shows: app=my-app (note the dash!)

# Fix: Update service selector
kubectl edit svc myapp-service -n production
selector:
  app: my-app  # Match pod labels exactly

# Verify endpoints created
kubectl get endpoints myapp-service -n production
# Should show pod IPs
```

#### B. LoadBalancer Stuck in Pending
```bash
# Check service
kubectl get svc myapp-service -n production
# Shows: <pending> in EXTERNAL-IP

# Check events
kubectl describe svc myapp-service -n production

# Common causes:
# 1. AWS Load Balancer Controller not installed
kubectl get deployment -n kube-system aws-load-balancer-controller

# Install if missing:
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster

# 2. IAM permissions missing
# Check node instance profile has LoadBalancer permissions

# 3. Subnet tagging missing
# Public subnets need:
aws ec2 create-tags \
  --resources subnet-xxx \
  --tags Key=kubernetes.io/role/elb,Value=1
```

#### C. Security Group Blocking Traffic
```bash
# Find security group of nodes
aws ec2 describe-instances \
  --filters "Name=tag:kubernetes.io/cluster/my-cluster,Values=owned" \
  --query 'Reservations[].Instances[].SecurityGroups[].GroupId'

# Check security group rules
aws ec2 describe-security-groups --group-ids sg-xxx

# Add rule to allow traffic on NodePort
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 30000-32767 \
  --cidr 0.0.0.0/0
```

---

### 4. DNS Resolution Failures

**Symptoms:**
- "nslookup: can't resolve 'service-name'"
- Pods can't communicate with services
- "dial tcp: lookup service-name: no such host"

**Diagnostic Commands:**
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS from pod
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Check DNS service
kubectl get svc -n kube-system kube-dns

# Verify DNS config in pod
kubectl run -it --rm debug --image=busybox --restart=Never -- cat /etc/resolv.conf
```

**What Should You See:**
```bash
# Healthy CoreDNS:
kubectl get pods -n kube-system -l k8s-app=kube-dns
# All pods Running

# Pod's /etc/resolv.conf:
nameserver 10.100.0.10  # ClusterIP of kube-dns service
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**Common Fixes:**

```bash
# Scenario 1: CoreDNS pods not running

# Check status
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Shows: CrashLoopBackOff

# Check logs
kubectl logs -n kube-system -l k8s-app=kube-dns
# Shows: "plugin/loop: Loop detected"

# Fix: Update CoreDNS ConfigMap
kubectl edit configmap coredns -n kube-system
# Remove 'loop' if causing issues
# Or add: forward . /etc/resolv.conf

# Restart CoreDNS
kubectl rollout restart -n kube-system deployment/coredns

# Scenario 2: Incorrect DNS service ClusterIP

# Check kube-dns service
kubectl get svc -n kube-system kube-dns
# ClusterIP should match pods' /etc/resolv.conf

# If mismatch, pods need to be recreated
kubectl delete pod <pod-name> -n production

# Scenario 3: Network policy blocking DNS

# Check for network policies
kubectl get networkpolicies -A

# Allow DNS egress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
EOF
```

---

### 5. Node NotReady / Node Pressure

**Symptoms:**
- Node STATUS: NotReady
- Node showing MemoryPressure, DiskPressure, or PIDPressure
- Pods evicted from nodes

**Diagnostic Commands:**
```bash
# Check node status
kubectl get nodes

# Describe problematic node
kubectl describe node <node-name>

# Check node conditions
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[?(@.type=="Ready")].status,REASON:.status.conditions[?(@.type=="Ready")].reason

# Check node resource usage
kubectl top nodes

# SSH to node (if possible)
# For EKS with managed nodes:
aws ssm start-session --target <instance-id>

# Check kubelet logs
journalctl -u kubelet -f

# Check system resources on node
df -h
free -h
top
```

**Common Issues & Fixes:**

#### A. Disk Pressure
```bash
# Diagnosis
kubectl describe node <node-name>
# Shows: Conditions: DiskPressure = True

# SSH to node
aws ssm start-session --target <instance-id>

# Check disk usage
df -h
# Shows: /dev/xvda1  100% full

# Find what's using space
du -sh /var/lib/* | sort -rh | head -10

# Clean Docker images/containers
docker system prune -af

# Clean old logs
find /var/log -type f -name "*.log" -mtime +7 -delete

# Or cordon and drain node, then replace
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Delete node group and create new one
aws eks delete-nodegroup --cluster-name my-cluster --nodegroup-name old-group
```

#### B. Memory Pressure
```bash
# Check node
kubectl describe node <node-name>
# Conditions: MemoryPressure = True

# See what's consuming memory
kubectl top pods -A --sort-by=memory

# Identify resource hogs
kubectl describe node <node-name> | grep -A 10 "Allocated resources"

# Fix: Add more nodes or increase node size
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --scaling-config minSize=3,maxSize=10,desiredSize=5
```

#### C. Node Not Ready (kubelet issues)
```bash
# SSH to node
aws ssm start-session --target <instance-id>

# Check kubelet status
sudo systemctl status kubelet

# Check kubelet logs
sudo journalctl -u kubelet -n 100

# Common issues:
# 1. Kubelet crashed
sudo systemctl restart kubelet

# 2. Container runtime issues
sudo systemctl status docker
sudo systemctl restart docker

# 3. Node can't reach API server
# Check security groups allow connection to EKS control plane
```

---

### 6. IAM Permissions / IRSA Issues

**Symptoms:**
- Pods can't access AWS services
- "Access Denied" errors in pod logs
- "User: is not authorized to perform: s3:GetObject"

**Diagnostic Commands:**
```bash
# Check if IRSA (IAM Roles for Service Accounts) is configured
kubectl get sa -n production myapp-sa -o yaml

# Check for eks.amazonaws.com/role-arn annotation
kubectl describe sa myapp-sa -n production

# Check pod's service account
kubectl get pod myapp-7d6c9f5b-xyz -n production -o yaml | grep serviceAccount

# Verify IAM role exists
aws iam get-role --role-name EKS-MyApp-Role

# Check IAM role trust policy
aws iam get-role --role-name EKS-MyApp-Role --query 'Role.AssumeRolePolicyDocument'

# Test AWS access from pod
kubectl exec -it myapp-7d6c9f5b-xyz -n production -- aws sts get-caller-identity
```

**Setup IRSA Correctly:**

```bash
# Step 1: Create IAM OIDC provider (if not exists)
eksctl utils associate-iam-oidc-provider \
  --cluster=my-cluster \
  --approve

# Step 2: Create IAM policy
cat <<EOF > policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name MyAppS3Policy \
  --policy-document file://policy.json

# Step 3: Create IAM role
eksctl create iamserviceaccount \
  --name myapp-sa \
  --namespace production \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::ACCOUNT_ID:policy/MyAppS3Policy \
  --approve

# Step 4: Update deployment to use service account
kubectl patch deployment myapp -n production -p \
  '{"spec":{"template":{"spec":{"serviceAccountName":"myapp-sa"}}}}'

# Verify
kubectl get sa myapp-sa -n production -o yaml
# Should show: eks.amazonaws.com/role-arn annotation

# Test from pod
kubectl exec -it deployment/myapp -n production -- aws s3 ls s3://my-bucket/
```

---

### 7. Cluster Autoscaler Not Working

**Symptoms:**
- Pods stuck in Pending despite autoscaler
- Nodes not scaling up when needed
- Nodes not scaling down when idle

**Diagnostic Commands:**
```bash
# Check if cluster autoscaler is running
kubectl get deployment cluster-autoscaler -n kube-system

# Check autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler -f

# Check autoscaler status
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# Check node group autoscaling config
aws eks describe-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --query 'nodegroup.scalingConfig'
```

**Common Issues & Fixes:**

```bash
# Scenario 1: Autoscaler not installed

# Install cluster autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Edit deployment to specify cluster name
kubectl edit deployment cluster-autoscaler -n kube-system
# Add to container args:
- --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster

# Scenario 2: Missing IAM permissions

# Create policy
cat <<EOF > autoscaler-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://autoscaler-policy.json

# Attach to node group role
aws iam attach-role-policy \
  --role-name <node-group-role> \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/AmazonEKSClusterAutoscalerPolicy

# Scenario 3: Node group misconfigured

# Check node group tags
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[].Tags'

# Should have:
# k8s.io/cluster-autoscaler/enabled = true
# k8s.io/cluster-autoscaler/my-cluster = owned

# Add tags if missing
aws autoscaling create-or-update-tags \
  --tags ResourceId=<asg-name>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=false
```

---

## 🔍 Advanced EKS Debugging

### Exec Into Running Container
```bash
# Get shell in container
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# Or sh if bash not available
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Run command without interactive shell
kubectl exec <pod-name> -n <namespace> -- curl localhost:8080/health

# For multi-container pods
kubectl exec -it <pod-name> -n <namespace> -c <container-name> -- /bin/bash
```

### Port Forwarding for Testing
```bash
# Forward local port to pod
kubectl port-forward pod/<pod-name> 8080:8080 -n <namespace>

# Forward to service
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>

# Test locally
curl localhost:8080
```

### Copy Files To/From Pod
```bash
# Copy from pod to local
kubectl cp <namespace>/<pod-name>:/path/to/file ./local-file

# Copy to pod
kubectl cp ./local-file <namespace>/<pod-name>:/path/to/file

# For multi-container pods
kubectl cp <namespace>/<pod-name>:/path/to/file ./local-file -c <container-name>
```

### Debug with Ephemeral Container
```bash
# Add debug container to running pod (K8s 1.18+)
kubectl debug <pod-name> -n <namespace> --image=busybox --target=<container-name>

# Run debug pod with same network namespace
kubectl debug <pod-name> -n <namespace> -it --image=nicolaka/netshoot --share-processes --copy-to=debug-pod
```

---

## 📊 Systematic EKS Troubleshooting Checklist

When EKS issues occur:

```bash
# 1. Check cluster health
aws eks describe-cluster --name my-cluster --query 'cluster.status'

# 2. Check nodes
kubectl get nodes

# 3. Check pods
kubectl get pods -A

# 4. Check recent events
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# 5. Check specific pod
kubectl describe pod <pod-name> -n <namespace>

# 6. Check logs
kubectl logs <pod-name> -n <namespace>

# 7. Check service endpoints
kubectl get endpoints -n <namespace>

# 8. Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 9. Check resource usage
kubectl top nodes
kubectl top pods -A

# 10. Check IAM/IRSA
kubectl get sa -A | grep <service-account>
```

---

## 💡 Key Takeaways

1. **Describe is your friend** - `kubectl describe` shows events and detailed state
2. **Logs reveal application issues** - Check current and previous container logs
3. **Events show scheduling issues** - Recent events often show root cause
4. **Resource limits matter** - Set appropriate requests and limits
5. **DNS is critical** - CoreDNS must be healthy
6. **IAM permissions** - Use IRSA for AWS service access
7. **Labels and selectors** - Must match exactly
8. **Node health** - Monitor for pressure conditions
9. **Security groups** - Must allow pod-to-pod and pod-to-API communication
10. **Version compatibility** - Keep cluster and tools updated

---

## 📚 Quick Reference Card

| Issue Type | First Command | Solution Pattern |
|------------|---------------|------------------|
| Pod Pending | `kubectl describe pod` | Check resources, PVC, node selector |
| CrashLoopBackOff | `kubectl logs` | Check app logs, liveness probe |
| Service Unreachable | `kubectl get endpoints` | Verify selector matches pod labels |
| DNS Failure | `kubectl logs -n kube-system -l k8s-app=kube-dns` | Check CoreDNS health |
| Node NotReady | `kubectl describe node` | Check disk/memory pressure |
| IAM Access Denied | `kubectl describe sa` | Verify IRSA configuration |
| Autoscaler Not Working | `kubectl logs cluster-autoscaler` | Check IAM, tags, config |

---

**Remember**: EKS issues are often multi-layered (pod → service → DNS → network → IAM). Follow systematic approach, check each layer, and verify assumptions. The describe and logs commands are your most powerful tools.
