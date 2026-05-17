# Quick Reference: Jenkins + MySQL InnoDB Cluster Deployment

## Files in This Repository

| File | Purpose |
|------|---------|
| `README.md` | Original setup guide (RHEL/CentOS focused) |
| `Jenkinsfile` | Original Jenkins pipeline script |
| `Jenkinsfile.ubuntu` | **NEW** - Ubuntu-optimized pipeline for worker nodes |
| `JENKINS_MASTER_WORKER_SETUP.md` | **NEW** - Connect Jenkins Master to Worker nodes |
| `ansible.txt` | Example directory structure and configuration |
| `ansible/` | Ansible playbooks & roles (needs to be created) |

---

## 🚀 QUICK START: 5-Minute Overview

### What You Need
- 3 Ubuntu EC2 instances (already running ✓)
- Jenkins installed on one of them (already done ✓)
- S3 bucket with MySQL binaries (to be done)
- SSH keys for authentication (to be done)

### What These Scripts Do
1. **Jenkinsfile.ubuntu** - Orchestrates MySQL cluster deployment on 3 nodes
2. **JENKINS_MASTER_WORKER_SETUP.md** - Connects Jenkins workers for scalability

### Timeline
- **Phase 1** (15 min): Setup EC2 prerequisites
- **Phase 2** (10 min): Upload binaries to S3
- **Phase 3** (20 min): Configure Jenkins credentials
- **Phase 4** (30 min): Setup Jenkins workers
- **Phase 5** (60 min): Test dry-run
- **Phase 6** (90 min): Full deployment

---

## 📋 EXECUTION CHECKLIST

### PRE-DEPLOYMENT
- [ ] All 3 EC2 nodes running Ubuntu
- [ ] Jenkins installed on master node and accessible
- [ ] SSH keys created (master → workers, master → MySQL nodes)
- [ ] S3 bucket created with MySQL binaries uploaded
- [ ] AWS credentials configured on Jenkins master
- [ ] Ansible installed on master AND both worker nodes
- [ ] Python 3.6+ on all nodes
- [ ] glibc 2.28+ verified: `ldd --version`

### JENKINS CONFIGURATION
- [ ] SSH credential "mysql-ssh-key" created in Jenkins
- [ ] Sudo password credential "mysql-sudo-pass" created in Jenkins
- [ ] Ansible Vault credential "ansible-vault-pass" created (optional)
- [ ] 2 worker nodes added to Jenkins and showing Online
- [ ] Both workers labeled with "mysql-worker"
- [ ] Git repository pushed to remote

### BUILD PARAMETERS READY
- [ ] 3 MySQL node IPs confirmed
- [ ] S3 bucket name correct in Jenkinsfile
- [ ] MySQL version selected (8.0.42-33 is default)
- [ ] App/Env/DC names finalized
- [ ] Ports available on all nodes

### FIRST RUN
- [ ] Build with DRY_RUN=true
- [ ] Review console output
- [ ] Verify no errors
- [ ] Check generated hosts.ini and vars.yml
- [ ] Build with DRY_RUN=false only after dry-run succeeds

---

## 🎯 COMMAND SEQUENCES

### On EC2 Master Node (Ubuntu)

```bash
# 1. Install Ansible & AWS CLI
sudo apt update && sudo apt install -y ansible awscli python3-pip
pip3 install boto3 botocore

# 2. Generate SSH key for worker connection
sudo su - jenkins
ssh-keygen -t rsa -b 4096 -f ~/.ssh/jenkins-to-worker -N ""

# 3. Test connectivity to workers
ssh -i ~/.ssh/jenkins-to-worker jenkins@WORKER1_IP "uname -a"
ssh -i ~/.ssh/jenkins-to-worker jenkins@WORKER2_IP "uname -a"
```

### On EC2 Worker Nodes (Ubuntu)

```bash
# 1. Install Java & dependencies
sudo apt update
sudo apt install -y openjdk-11-jre-headless ansible git python3-pip

# 2. Create jenkins user
sudo useradd -m -s /bin/bash -d /opt/jenkins jenkins
echo "jenkins ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/jenkins

# 3. Create agent directory
sudo mkdir -p /opt/jenkins/agent
sudo chown -R jenkins:jenkins /opt/jenkins

# 4. Setup SSH key from master
# (Jenkins Master will copy its public key here)
```

### In Jenkins UI

```
1. Go to: Manage Credentials → Add Credentials
   - SSH key for workers
   - Sudo password
   - Vault password (optional)

2. Go to: Manage Nodes → New Node (×2 times)
   - Name: worker-1, worker-2
   - Labels: mysql-worker
   - Host: worker IP addresses
   - Launch via SSH with credentials

3. Verify: Both workers should show Online

4. Create Pipeline Job
   - Name: mysql-innodb-cluster-ubuntu
   - Pipeline from SCM (Git)
   - Script Path: Jenkinsfile.ubuntu
```

---

## 🔄 TYPICAL BUILD FLOW

```
Click: Build with Parameters (on job in Jenkins)
     ↓
[USER INPUT]
- NODE IPs
- MySQL version
- S3 bucket name
     ↓
Jenkins Master assigns to available worker
     ↓
Worker runs: Pre-flight checks
     ↓
Worker validates: All parameters
     ↓
Worker tests: SSH connectivity to 3 MySQL nodes
     ↓
Worker generates: Ansible inventory & variables
     ↓
Worker runs: ansible-playbook --check (dry-run) OR
Worker runs: ansible-playbook (live)
     ↓
[RESULTS]
✅ Success → cluster deployed, logs archived
❌ Failure → detailed error messages, logs available
```

---

## 📊 PARAMETER REFERENCE

| Parameter | Default | What to Change |
|-----------|---------|-----------------|
| `NODE1_IP` | Empty | Primary MySQL IP (e.g., 10.0.1.11) |
| `NODE2_IP` | Empty | Secondary MySQL IP #1 (e.g., 10.0.1.12) |
| `NODE3_IP` | Empty | Secondary MySQL IP #2 (e.g., 10.0.1.13) |
| `SSH_USER` | ubuntu | Keep "ubuntu" for Ubuntu EC2 |
| `MYSQL_MINOR` | 8.0.42-33 | Version of Percona Server |
| `S3_BUCKET` | your-org-mysql-binaries | Your actual S3 bucket name |
| `MYSQL_PORT` | 3306 | MySQL port (if blocked, use 3307) |
| `APP_NAME` | myapp | Your application name |
| `ENV_NAME` | poc | Environment: poc, dev, qa, prod |
| `DRY_RUN` | true | **Always true for first run!** |

---

## 🐛 TROUBLESHOOTING GUIDE

### Jenkins Worker Shows "Offline"
```bash
1. Check connection log:
   Manage Nodes → worker-1 → System Log
   
2. Verify Jenkins SSH key:
   sudo su - jenkins
   cat ~/.ssh/jenkins-to-worker  # Should exist
   
3. Verify worker has Java:
   ssh jenkins@WORKER_IP java -version
   
4. Check firewall:
   sudo ufw status (if UFW enabled)
```

### Build Fails at "SSH Connectivity Test"
```bash
1. From Jenkins Master, test SSH manually:
   sudo su - jenkins
   ssh -i ~/.ssh/jenkins-to-worker jenkins@WORKER_IP \
       ssh -i ~/.ssh/id_rsa ubuntu@MYSQL_NODE_IP "echo OK"
   
2. Verify MySQL node IPs are correct
3. Check security groups allow SSH from worker nodes
```

### Ansible Syntax Error
```bash
1. Check playbook exists:
   ls -la ansible/playbooks/main.yml
   
2. Validate syntax:
   ansible-playbook --syntax-check -i inventory/hosts.ini playbooks/main.yml
   
3. Create playbooks if missing (see ansible.txt for structure)
```

### S3 Access Denied
```bash
1. Verify AWS credentials:
   aws sts get-caller-identity
   
2. Verify S3 bucket exists:
   aws s3 ls s3://YOUR_BUCKET/
   
3. Check files uploaded:
   aws s3 ls s3://YOUR_BUCKET/mysql/
```

---

## 📝 ANSIBLE PLAYBOOK STRUCTURE (To Be Created)

Your `ansible/` directory needs this structure:

```
ansible/
├── playbooks/
│   ├── main.yml                    # Entry point
│   └── validate_setup.yml          # Cluster health check
├── roles/
│   ├── mysql/
│   │   ├── tasks/
│   │   │   ├── install_dependencies.yml
│   │   │   ├── download_binaries.yml
│   │   │   ├── install_mysql.yml
│   │   │   ├── configure_group_replication.yml
│   │   │   └── configure_cluster.yml
│   │   └── templates/
│   │       ├── my.cnf.j2
│   │       └── mysql-service.j2
│   ├── maintenance/
│   │   └── tasks/
│   └── monitor/
│       └── tasks/
└── inventory/
    ├── hosts.ini        # Auto-generated by Jenkins
    └── vars.yml         # Auto-generated by Jenkins
```

---

## 🔐 CREDENTIALS CHECKLIST

Create these in Jenkins: Manage Jenkins → Credentials → Add Credentials

| ID | Type | Contains |
|---|---|---|
| `mysql-ssh-key` | SSH Username with private key | Private key from Master jenkins user |
| `mysql-sudo-pass` | Secret text | Sudo password (if passwordless not configured) |
| `ansible-vault-pass` | Secret text | Ansible Vault encryption password (optional) |

---

## ✨ FEATURES OF NEW JENKINSFILE

✅ **Ubuntu/Debian Optimized**
- Uses `python3`, `/usr/bin/python3` interpreter
- Compatible with `apt` package manager
- No RHEL/CentOS dependencies

✅ **Worker Node Support**
- Runs on agents labeled `mysql-worker`
- Distributes load across multiple workers
- Easy horizontal scaling

✅ **Pre-flight Validation**
- Checks all dependencies installed
- Validates parameters before execution
- Tests SSH connectivity
- Resolves S3 paths

✅ **Better Logging**
- Timestamped output
- Archived artifacts
- Separate log files per build
- Debug mode available

✅ **Safe Deployment**
- Dry-run mode by default
- Manual confirmation before live deployment
- Input wait on critical actions
- Rollback capability (via input)

✅ **Flexible Options**
- Multiple deployment modes
- SSH user customization
- Support for multiple MySQL versions
- Verbose/debug flags

---

## 📞 NEXT STEPS

1. **Review** [JENKINS_MASTER_WORKER_SETUP.md](JENKINS_MASTER_WORKER_SETUP.md) for worker configuration
2. **Update** Jenkinsfile.ubuntu with your S3 bucket name
3. **Create** Jenkins credentials in web UI
4. **Add** 2 worker nodes in Jenkins
5. **Test** with first dry-run build
6. **Deploy** for real after validation

---

## 📚 RELATED FILES

- 📖 [README.md](README.md) - Original setup guide
- 🔧 [JENKINS_MASTER_WORKER_SETUP.md](JENKINS_MASTER_WORKER_SETUP.md) - Master-Worker connection
- 📋 [ansible.txt](ansible.txt) - Ansible directory structure
- 🏗️ [Jenkinsfile.ubuntu](Jenkinsfile.ubuntu) - New Ubuntu-optimized pipeline
