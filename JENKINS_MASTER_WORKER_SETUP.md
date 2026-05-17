# Jenkins Master-Worker Setup Guide for MySQL InnoDB Cluster Deployment
> Connect your Jenkins Master EC2 instance to 2 Worker EC2 nodes for distributed builds

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step-by-Step Setup](#step-by-step-setup)
4. [Verification & Troubleshooting](#verification--troubleshooting)
5. [Using the New Jenkinsfile](#using-the-new-jenkinsfile)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│ Your Local Machine                                              │
│ (where you're reading this)                                     │
└────────────────────┬──────────────────────────────────────────┘
                     │ (SSH / Browser)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ EC2 Instance 1: Jenkins Master                                  │
│ - Runs Jenkins service (port 8080)                              │
│ - Orchestrates builds                                           │
│ - Connects to Worker Nodes via SSH                              │
│ - Executes jobs on labeled agents (mysql-worker label)          │
└────────────────────┬──────────────────────────────────────────┘
                     │ (SSH)
        ┌────────────┴────────────┐
        ▼                         ▼
┌──────────────────┐       ┌──────────────────┐
│ EC2 Instance 2:  │       │ EC2 Instance 3:  │
│ Jenkins Worker 1 │       │ Jenkins Worker 2 │
│ (Ansible Host)   │       │ (Ansible Host)   │
│ - Runs JNLP Agent│       │ - Runs JNLP Agent│
│ - Executes Jobs  │       │ - Executes Jobs  │
│ - Labeled:       │       │ - Labeled:       │
│   mysql-worker   │       │   mysql-worker   │
└──────────────────┘       └──────────────────┘
        │                         │
        └────────────┬────────────┘
                     ▼
        (Ansible orchestrates MySQL deployment
         on 3 MySQL cluster nodes via SSH)
```

---

## Prerequisites

### On All EC2 Instances (Master + 2 Workers)
- Ubuntu 18.04 LTS or newer
- Java 11+ installed
- SSH access between master and workers
- Sudo privileges or passwordless sudo

### On Master EC2 Instance
- Jenkins installed and running (you've done this ✓)
- Ansible 2.9+ (for orchestration)
- Python 3.6+

### On Worker EC2 Instances
- Java 11+ (Jenkins agent requirement)
- Ansible 2.9+ (for executing Ansible playbooks)
- Python 3.6+
- SSH access to MySQL cluster nodes

### Network Requirements
- Master can SSH to both workers (port 22 by default)
- Workers can SSH to each other and to MySQL nodes
- Master runs Jenkins on port 8080 (or configured port)

---

## Step-by-Step Setup

### PHASE 1: Prepare Worker Nodes (EC2 Instance 2 & 3)

#### 1.1: Install Java on Workers

```bash
# SSH into Worker 1 & Worker 2, then run:
sudo apt update
sudo apt install -y openjdk-11-jre-headless git ansible python3 python3-pip curl wget

# Verify Java
java -version

# Verify Ansible
ansible --version

# Verify Python3
python3 --version
```

#### 1.2: Create Jenkins User on Workers

```bash
# On both worker nodes
sudo useradd -m -s /bin/bash -d /opt/jenkins jenkins 2>/dev/null || true
sudo usermod -aG sudo jenkins

# Allow passwordless sudo for jenkins user (optional but recommended)
echo "jenkins ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/jenkins
sudo chmod 440 /etc/sudoers.d/jenkins

# Verify
sudo su - jenkins
exit
```

#### 1.3: Create Jenkins Agent Directory on Workers

```bash
# On both worker nodes
sudo mkdir -p /opt/jenkins/agent
sudo chown -R jenkins:jenkins /opt/jenkins
chmod -R 755 /opt/jenkins
```

---

### PHASE 2: Generate SSH Keys for Master ↔ Worker Communication

#### 2.1: On Jenkins Master, Generate SSH Key Pair

```bash
# SSH into Jenkins Master (EC2 Instance 1)
ssh -i your-key.pem ubuntu@<MASTER_IP>

# Switch to jenkins user
sudo su - jenkins

# Generate SSH key (no passphrase for automated connections)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/jenkins-to-worker -N ""

# Verify key generated
ls -la ~/.ssh/jenkins-to-worker*

# Display public key (you'll need this)
cat ~/.ssh/jenkins-to-worker.pub
```

#### 2.2: Copy Public Key to Both Workers

```bash
# Still as jenkins user on Master, copy key to workers:

WORKER1_IP="10.0.1.20"  # Replace with your Worker 1 IP
WORKER2_IP="10.0.1.21"  # Replace with your Worker 2 IP

# Add public key to authorized_keys on both workers
for WORKER_IP in $WORKER1_IP $WORKER2_IP; do
    echo "Adding key to jenkins@$WORKER_IP..."
    
    ssh-copy-id -i ~/.ssh/jenkins-to-worker.pub \
        -o StrictHostKeyChecking=no \
        jenkins@$WORKER_IP || {
        
        # If ssh-copy-id fails, do it manually:
        echo "Manual key copy to $WORKER_IP..."
        ssh jenkins@$WORKER_IP "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
        cat ~/.ssh/jenkins-to-worker.pub | \
            ssh jenkins@$WORKER_IP "cat >> ~/.ssh/authorized_keys"
        ssh jenkins@$WORKER_IP "chmod 600 ~/.ssh/authorized_keys"
    }
    
    # Verify
    echo "Testing SSH to $WORKER_IP..."
    ssh -i ~/.ssh/jenkins-to-worker jenkins@$WORKER_IP "echo 'SSH works!' && whoami"
done

echo "✅ SSH keys distributed"
```

#### 2.3: Test SSH Connectivity from Master to Workers

```bash
# Still as jenkins user on Master
ssh -i ~/.ssh/jenkins-to-worker jenkins@10.0.1.20 "uname -a"
ssh -i ~/.ssh/jenkins-to-worker jenkins@10.0.1.21 "uname -a"

# Both should work without password
```

---

### PHASE 3: Configure Jenkins Master to Recognize Workers

#### 3.1: Open Jenkins Web UI

```bash
# From your local machine
open http://<MASTER_IP>:8080   # macOS
# OR
xdg-open http://<MASTER_IP>:8080  # Linux
# OR just paste in browser: http://<MASTER_IP>:8080
```

#### 3.2: Add SSH Private Key as Jenkins Credential

```
1. Go to: Jenkins Dashboard → Manage Jenkins → Manage Credentials
2. Click: "global" (or default store)
3. Click: "Add Credentials" (top left)

4. Fill in:
   - Kind: SSH Username with private key
   - Scope: Global
   - Username: jenkins
   - Private Key: (select) Enter directly
     - Paste content of ~/.ssh/jenkins-to-worker (from Master)
   - ID: jenkins-worker-ssh-key
   - Description: SSH key from Jenkins Master to Worker nodes
   
5. Click: "Create"
```

#### 3.3: Create Worker Nodes in Jenkins

**Create Worker 1:**

```
1. Go to: Jenkins Dashboard → Manage Jenkins → Manage Nodes and Clouds
2. Click: "New Node" (or "New Item" → "Node")
3. Fill in:
   - Node name: worker-1
   - Type: Permanent Agent
   - Click: "Create"

4. Configure:
   - # of executors: 2 (adjust based on node resources)
   - Remote root directory: /opt/jenkins/agent
   - Labels: mysql-worker, ansible-executor (IMPORTANT - must include mysql-worker)
   - Usage: Use this node as much as possible (or "Only build jobs with labels matching")
   - Launch method: Launch agent via SSH
     - Host: 10.0.1.20 (Worker 1 IP)
     - Credentials: jenkins-worker-ssh-key (the one you just created)
     - Host Key Verification Strategy: Non verifying Verification Strategy
   - Availability: Keep this agent online as much as possible
   
5. Click: "Save"
```

**Create Worker 2:**

```
Repeat above steps but with:
   - Node name: worker-2
   - Host: 10.0.1.21 (Worker 2 IP)
   - Labels: mysql-worker, ansible-executor
```

#### 3.4: Wait for Workers to Connect

```
- Go to Jenkins Dashboard → Manage Nodes
- You should see:
  • worker-1 → Status: Online ✓
  • worker-2 → Status: Online ✓

If Status is "Offline":
  - Click on the worker node
  - Check "System Log" for errors
  - Common issues:
    * SSH key permissions (should be 600)
    * Jenkins user doesn't exist on worker
    * Java not installed on worker
    * Firewall blocking SSH
```

---

### PHASE 4: Install Ansible on Workers

#### 4.1: Install Ansible & Dependencies on Both Workers

```bash
# SSH into each worker
ssh -i your-key.pem ubuntu@<WORKER_IP>
sudo su

# Install Ansible and dependencies
apt update
apt install -y ansible python3-pip
pip3 install boto3 botocore awscli

# Verify
ansible --version
pip3 list | grep boto
aws --version

exit  # Exit root
exit  # Exit worker SSH
```

#### 4.2: Copy Ansible Inventory to Workers (from Master)

```bash
# On Master, as jenkins user
cd /opt/jenkins/workspace/
ls -la  # Check for ansible directory with inventory files

# Copy to both workers
for WORKER in jenkins@10.0.1.20 jenkins@10.0.1.21; do
    scp -r ./ansible/inventory/* $WORKER:/opt/jenkins/ansible/inventory/
done
```

---

### PHASE 5: Update Your Jenkinsfile

#### 5.1: Use the New Ubuntu-Optimized Jenkinsfile

The new file `Jenkinsfile.ubuntu` includes:
- ✓ Worker node label: `mysql-worker`
- ✓ Ubuntu/Debian compatibility
- ✓ Pre-flight checks for dependencies
- ✓ SSH connectivity validation
- ✓ Better parameter handling

**Option A: Replace your current Jenkinsfile**
```bash
cd ~/mysql-innodb-cluster
cp Jenkinsfile Jenkinsfile.backup
cp Jenkinsfile.ubuntu Jenkinsfile
git add Jenkinsfile
git commit -m "Update Jenkinsfile for Ubuntu + worker nodes"
git push
```

**Option B: Keep both versions**
```bash
# In Jenkins UI: when creating pipeline job, select:
# Definition: Pipeline script from SCM
# Script Path: Jenkinsfile.ubuntu  # Instead of Jenkinsfile
```

#### 5.2: Update S3 Bucket Name in Jenkinsfile

Find this line in your Jenkinsfile:
```groovy
string(name: 'S3_BUCKET', defaultValue: 'your-org-mysql-binaries', ...)
```

Update `your-org-mysql-binaries` to your actual S3 bucket name.

---

## Verification & Troubleshooting

### Check Worker Status

```
Jenkins UI → Manage Nodes and Clouds
- Both workers should show as "Online"
- Click each to see:
  - System Load
  - Java version
  - Disk space
  - Connected since
```

### Test Agent Connectivity

```bash
# On Master, as jenkins user
ssh -i ~/.ssh/jenkins-to-worker jenkins@10.0.1.20 "java -version && ansible --version"
ssh -i ~/.ssh/jenkins-to-worker jenkins@10.0.1.21 "java -version && ansible --version"
```

### Troubleshooting Common Issues

| Problem | Solution |
|---------|----------|
| **Worker shows "Offline"** | Check Jenkins system log: Manage Nodes → worker-1 → System Log |
| **SSH: Permission denied** | Verify key permissions: `ls -l ~/.ssh/jenkins-to-worker*` should be `-rw-------` (600) |
| **Java not found on worker** | Run: `sudo apt install -y openjdk-11-jre-headless` on worker |
| **Ansible not found** | Run: `sudo apt install -y ansible` on worker |
| **Cannot write to /opt/jenkins** | Check ownership: `ls -ld /opt/jenkins` should be `jenkins:jenkins` |
| **SSH hangs when connecting** | Check firewall: `sudo ufw status` or check security groups if on AWS |

---

## Using the New Jenkinsfile

### Create a New Pipeline Job

```
1. Jenkins Dashboard → + New Item
2. Item name: mysql-innodb-cluster-ubuntu
3. Type: Pipeline
4. Click: OK

5. Configure:
   - Under "Pipeline":
     - Definition: Pipeline script from SCM
     - SCM: Git
     - Repository URL: https://github.com/YOUR-ORG/mysql-innodb-cluster.git
     - Branch: */main
     - Script Path: Jenkinsfile.ubuntu (or just Jenkinsfile if you replaced it)
   
6. Click: Save

7. First build → Build with Parameters:
   - See new fields with descriptions
   - START WITH DRY_RUN = true
   - Click: Build
```

### Key New Features

1. **Deployment Mode** - Choose what to deploy:
   - `full-stack` - Everything
   - `mysql-only` - Just MySQL
   - `validate-only` - Dry-run only

2. **SSH User** - Support for ubuntu, ec2-user, admin, etc.

3. **Advanced Options**:
   - `SKIP_PREFLIGHT` - Skip checks (not recommended)
   - `VERBOSE` - Enable verbose output for debugging
   - `DEBUG` - Extra debug logging

### Example Build Parameters

```
DEPLOYMENT_MODE:    full-stack
DRY_RUN:            true              (start here!)
NODE1_IP:           10.0.1.11         (MySQL node 1)
NODE2_IP:           10.0.1.12         (MySQL node 2)
NODE3_IP:           10.0.1.13         (MySQL node 3)
SSH_USER:           ubuntu
MYSQL_MINOR:        8.0.42-33
APP_NAME:           myapp
ENV_NAME:           poc
S3_BUCKET:          your-org-mysql-binaries
SSH_CRED_ID:        mysql-ssh-key
SUDO_PASS_CRED:     mysql-sudo-pass
```

---

## Next Steps

1. ✅ Install Java on both workers
2. ✅ Create jenkins user on workers
3. ✅ Generate SSH keys on master
4. ✅ Add workers to Jenkins
5. ✅ Verify workers are online
6. ✅ Install Ansible on workers
7. ✅ Update Jenkinsfile with S3 bucket name
8. ✅ Test with `DRY_RUN=true` build
9. ✅ Run real deployment with `DRY_RUN=false`

---

## Detailed Build Process Flow

When you click **Build**, here's what happens:

```
1. Jenkins Master receives build request
2. Master routes job to worker with label "mysql-worker"
   (worker-1 OR worker-2, whichever is available)

3. Worker stage: Pre-Flight Checks
   ✓ Verify Ansible installed
   ✓ Verify Python3 installed
   ✓ Verify SSH client available

4. Worker stage: Validate Parameters
   ✓ Check IP addresses valid
   ✓ Check ports in valid range
   ✓ Verify credentials exist in Jenkins

5. Worker stage: SSH Connectivity Test
   ✓ Test SSH to each of 3 MySQL nodes
   (from worker node, not from master!)

6. Worker stage: Resolve S3 Paths
   ✓ Map MySQL version to S3 URLs

7. Worker stage: Generate Inventory & Vars
   ✓ Create ansible/inventory/hosts.ini
   ✓ Create ansible/inventory/vars.yml

8. If DRY_RUN=true:
   Worker stage: Dry-Run
   ✓ Run: ansible-playbook --check
   ✓ Shows what WOULD happen (no changes made)
   ✓ Build completes

9. If DRY_RUN=false:
   Worker stage: Deploy
   ✓ Prompt for confirmation
   ✓ Run: ansible-playbook (live)
   ✓ Makes actual changes
   
   Worker stage: Validate Cluster
   ✓ Check cluster is healthy
   ✓ All 3 nodes online

10. Results archived and available in Jenkins UI
```

---

## Security Considerations

1. **SSH Keys**: Stored in Jenkins credentials (encrypted)
2. **Passwords**: Use Ansible Vault for sensitive data, not plain text
3. **Network**: Ensure security groups allow SSH between Master ↔ Workers ↔ MySQL nodes
4. **Logs**: Build logs contain node IPs (keep confidential)
5. **Audit**: Enable Jenkins audit logging for compliance

---

## Support & Additional Commands

### View Worker Status from CLI

```bash
# SSH to Master, as jenkins user
sudo su - jenkins

# List all worker nodes
curl -s http://localhost:8080/computer/api/json | grep displayName

# Check if specific worker is online
curl -s http://localhost:8080/computer/worker-1/api/json | grep offline
```

### Restart Worker Connection (if stuck)

```bash
# From Jenkins UI:
Manage Nodes and Clouds → worker-1 → Disconnect
[Wait 30 seconds]
Worker should reconnect automatically

# Or restart Jenkins:
ssh -i your-key.pem ubuntu@<MASTER_IP>
sudo systemctl restart jenkins
```

### View Agent Logs

```bash
On Master, Jenkins → Manage Nodes → worker-1 → System Log
(Shows agent connection attempts and errors)
```

---

## Questions?

- **Jenkins Agent Connection Issues**: Check `<JENKINS_HOME>/logs/agents.log`
- **Ansible Playbook Errors**: Check `logs/deploy-*.log` in Jenkins build
- **SSH Auth Failures**: Verify SSH key permissions with `ls -l ~/.ssh/`
- **Python Interpreter Not Found**: Ansible expects Python3, verify with `python3 --version` on nodes
