# Ansible Configuration Management An Automated Infrastructure Provisioning

## Project Summary

This project implements **Configuration Management** across a multi-server AWS environment using Ansible, replacing manual SSH-based server administration with a single declarative automation workflow.

The Jenkins server (renamed `Jenkins-Ansible`) acts as both the CI/CD engine and Ansible control node, a **Jump Server / Bastion Host** pattern. Any code push to `main` triggers an automated Jenkins build, and Ansible playbooks are executed against the full server fleet without ever logging into individual machines.

---

## Architecture

```
GitHub Repository
       │
       │  webhook (push event)
       ▼
Jenkins-Ansible (Control Node / Bastion Host)
  ├── Elastic IP (stable CI/CD endpoint)
  ├── Ansible installed via pip
  └── SSH Agent Forwarding enabled
             │
      ┌──────┼──────────────────────┐
      ▼      ▼          ▼           ▼
  Web × 2   NFS        DB          LB
  (RHEL)   (RHEL)   (Ubuntu)    (Ubuntu)
```

**Why a Bastion Host?** The web servers sit in a private subnet, not directly reachable from the internet. The Jenkins-Ansible node is the single authorised entry point, reducing the attack surface and centralising audit logging.

---

## Technology Stack

| Layer | Tool |
|---|---|
| Cloud Infrastructure | AWS EC2 (RHEL + Ubuntu AMIs) |
| Configuration Management | Ansible (pip-installed, core 2.17.x) |
| CI/CD | Jenkins (Freestyle job + GitHub webhook) |
| Source Control | GitHub |
| Task Tracking | Jira (Free tier, DEL project) |
| Editor | VS Code with Remote-SSH extension |

---

## Repository Structure

```
ansible-config-mgt/
├── inventory/
│   ├── dev.yml        # Development environment hosts
│   ├── staging.yml
│   ├── uat.yml
│   └── prod.yml
└── playbooks/
    └── common.yml     # Cross-platform base configuration playbook
```

---

## Step 1 — Prepare the Jenkins-Ansible Server

### 1.1 Rename the EC2 Instance

In AWS Console: **EC2 → Instances → Select Jenkins instance → Tags → Edit**

Set `Name = Jenkins-Ansible`. This server serves dual purpose as the Jenkins CI node and Ansible control node.

### 1.2 Allocate an Elastic IP

Without a static IP, the GitHub webhook URL changes every time the instance restarts, silently breaking the CI pipeline.

**AWS Console → EC2 → Elastic IPs → Allocate Elastic IP Address → Associate with Jenkins-Ansible**

Save the allocated IP, this becomes the permanent webhook endpoint.

### 1.3 Install Ansible

SSH into the Jenkins-Ansible server:

```bash
ssh ubuntu@<JENKINS_PUBLIC_IP>
```

Install Ansible via pip for the latest stable version (the apt package lags behind):

```bash
sudo apt update
sudo apt install python3-pip -y
pip install --upgrade pip
pip install ansible
```
![01-sudo-update-ansible-install.png](images/01-sudo-update-ansible-install.png)

Pip installs binaries to `~/.local/bin`, which is not on `$PATH` by default on some Ubuntu configurations. Fix this permanently:

```bash
export PATH=$PATH:/home/ubuntu/.local/bin
echo 'export PATH=$PATH:/home/ubuntu/.local/bin' >> ~/.bashrc
source ~/.bashrc
```

Verify the installation:

```bash
ansible --version
```
Expected output:

```
ansible [core 2.17.x]
  executable location = /home/ubuntu/.local/bin/ansible
  python version = 3.10.x
```
![02-ansible-version.png](images/02-ansible-version.png)

### 1.4 Install the tree utility

```bash
sudo apt install tree -y
```

---

## Step 2 — Create the GitHub Repository

On GitHub, create a new repository named `ansible-config-mgt` (public or private).

Clone it to the Jenkins-Ansible server:

```bash
git clone https://github.com/<your-username>/ansible-config-mgt.git
cd ansible-config-mgt
```
![03-clone to jenkins-ansible.png](images/03-clone%20to%20jenkins-ansible.png)

---

## Step 3 — Configure Jenkins

### 3.1 Create a Freestyle Job

In the Jenkins UI: **New Item → Freestyle Project → Name: `ansible` → OK**
![04-create-new-freestyle.png](images/04-create-new-freestyle.png)

![04-new-freestyle-job.png](images/04-new-freeestyle-job.png)

### 3.2 Connect to the GitHub Repository

Under **Source Code Management**:
- Select **Git**
- Repository URL: `https://github.com/<your-username>/ansible-config-mgt.git`
- Branch: `*/main`
- Credentials: Add a GitHub Personal Access Token (PAT) as `Username with password`; username is your GitHub handle, password is the PAT

> **Why PAT instead of password?** GitHub deprecated password authentication for Git operations. Without a PAT, Jenkins throws `Failed to connect to repository: git ls-remote HEAD`. Generate a PAT at: GitHub → Settings → Developer Settings → Personal Access Tokens → Generate new token → select `repo` scope.

![05-connect-github-repo.png](images/05-connect-github-repo.png)

![05-github-repo-connect.png](images/05-github-repo-connect.png)

### 3.3 Set the Build Trigger

Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**.

### 3.4 Archive Build Artifacts

Under **Post-build Actions**, add **Archive the artifacts** with files pattern: `**`

![06-artifacts-configure.png](images/06-artifacts-configure.png)

![06-configure-artifacts.png](images/06-configure-artifacts.png)

Save the job. Artifacts will be stored at:

```
/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```

### 3.5 Configure the GitHub Webhook

In the GitHub repository: **Settings → Webhooks → Add webhook**
![07-setup-github-webhook.png](images/07-setup-github-webhook.png)
![07-webhook.png](images/07-webhook.png)
![07-webhook-successful.png](images/07-webhook-successful.png)


```
Payload URL:  http://<ELASTIC_IP>:8080/github-webhook/
Content type: application/json
Events:       Just the push event
```
![07-webhook-url](images/07-webhook-url.png)

Save. GitHub sends a ping request, a green tick confirms the connection.

### 3.6 Test the Pipeline End-to-End

```bash
cd ansible-config-mgt
echo "## Webhook test" >> README.md
git add README.md
git commit -m "Test webhook trigger"
git push origin main
```
![08-webhook-test.png](images/08-webhook-test.png)

Open Jenkins; a build should start automatically within seconds. Once complete, verify artifacts were archived:
![08-jenkins-build.png](images/08-jenkins-build.png)
![08-jenkins-build-console.png](images/08-jenkins-build-console.png)
![08-jenkins-build-status.png](images/08-jenkins-build-status.png)

```bash
cd /var/lib/jenkins/jobs/ansible/builds/
ls
# Enter the latest build number
cd <build_number>/archive/
ls -l
cat README.md
```
![09-archifacts-archive.png](images/09-archifacts-archive.png)
![09-first-github-commit.png](images/09-first-github-commit.png)
![09-archifacts-path.png](images/09-artifacts-path.png)

Seeing your README content confirms the full pipeline is working: push → webhook → Jenkins pull → archive.

---

## Step 4 — Task Tracking with Jira

Jira was introduced to simulate a real DevOps workflow. Tickets provide traceability between work items and code changes, a standard practice in engineering teams.

Three tickets were created under the DEL project:

| Ticket | Summary |
|---|---|
| DEL-1 | Setup Jenkins-Ansible Integration |
| DEL-2 | Configure Ansible Directory Structure |
| DEL-3 | Automate Configuration Deployment |

Branch names reference the ticket number, making it immediately clear what code change relates to what work item.

![10-jira-dashboard.png](images/10-jira-dashboard.png)
![10-jira-ticket-create.png](images/10-jira-ticket-create.png)
![10-ticket.png](images/10-ticket.png)

---

## Step 5 — Create the Feature Branch

```bash
cd ansible-config-mgt
git checkout -b feature/DEL-3-ansible-setup
git status
```
![11-create-your-feature-branch.png](images/11-create-your-feature-branch.png)

All development work happens on this branch. Nothing reaches `main` without a pull request review; enforcing the **Four Eyes Principle** (no code deployed without a second reviewer).

---

## Step 6 — Build the Project Structure

### 6.1 Create directories

```bash
mkdir playbooks inventory
```

### 6.2 Create required files

```bash
touch playbooks/common.yml
touch inventory/dev.yml
touch inventory/staging.yml
touch inventory/uat.yml
touch inventory/prod.yml
```
![12-create-project-structure-and-files.png](images/12-create-project-structure-and-files.png)

### 6.3 Verify the layout

```bash
tree
```

Expected output:

```
ansible-config-mgt/
├── inventory/
│   ├── dev.yml
│   ├── staging.yml
│   ├── uat.yml
│   └── prod.yml
├── playbooks/
│   └── common.yml
└── README.md
```
![13-verify-structure-.png](images/13-verify%20-structure-.png)
---

## Step 7 — Configure the Inventory

`inventory/dev.yml` groups hosts by function and assigns the correct SSH user per AMI type.

```bash
nano inventory/dev.yml
```

Paste the following, replacing IPs with your actual private IP addresses:

```ini
[webservers]
172.31.24.97 ansible_ssh_user=ec2-user
172.31.18.88 ansible_ssh_user=ec2-user

[nfs]
172.31.22.134 ansible_ssh_user=ec2-user

[db]
172.31.21.250 ansible_ssh_user=ubuntu

[lb]
172.31.16.84 ansible_ssh_user=ubuntu

[all:vars]
ansible_ssh_private_key_file=~/.ssh/My-StegHub-Keypair.pem
```

Save and exit: `Ctrl+O → Enter → Ctrl+X`

**Why separate SSH users matter:** RHEL-based AMIs use `ec2-user`; Ubuntu AMIs use `ubuntu`. Using the wrong user causes `Permission denied (publickey)` — an error that looks like a network or Ansible problem but is purely an authentication mismatch at the OS level.

---

## Step 8 — Write the Playbook

```bash
nano playbooks/common.yml
```

Paste the following:

```yaml
---
- name: Update web, nfs and db servers
  hosts: webservers,nfs,db
  become: yes

  tasks:
    - name: Ensure wireshark is latest (RHEL)
      yum:
        name: wireshark
        state: latest
      when: ansible_os_family == "RedHat"

    - name: Ensure wireshark is latest (Ubuntu)
      apt:
        name: wireshark
        state: latest
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Create devops directory
      file:
        path: /opt/devops
        state: directory

    - name: Create info file
      file:
        path: /opt/devops/info.txt
        state: touch

    - name: Set timezone
      timezone:
        name: Africa/Lagos

- name: Update Load Balancer
  hosts: lb
  become: yes

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install wireshark
      apt:
        name: wireshark
        state: latest

    - name: Create devops directory on LB
      file:
        path: /opt/devops
        state: directory
        mode: '0755'

    - name: Write automation marker
      copy:
        content: "Ansible Automation Completed"
        dest: /opt/devops/info.txt
        owner: ubuntu
        group: ubuntu
        mode: '0644'

    - name: Set timezone on LB
      timezone:
        name: Africa/Lagos
```

Save and exit

**Design decisions:**
- `when: ansible_os_family` guards prevent `yum` from running on Ubuntu hosts and `apt` from running on RHEL hosts — this is what allows a single playbook to manage a mixed OS fleet without separate files per OS.
- `become: yes` elevates to root for package installation and system-level changes.
- The `/opt/devops/info.txt` marker file provides a quick idempotency check during verification.

---

## Step 9 — Set Up SSH Authentication

Ansible connects to remote hosts over SSH. The private key must exist on the Jenkins-Ansible control node, not just your local machine.

### 9.1 Copy the key to the control node

From your local machine:

```bash
scp -i <local-key-path> My-StegHub-Keypair.pem ubuntu@<JENKINS_PUBLIC_IP>:~/.ssh/
```

### 9.2 Set correct permissions

```bash
chmod 400 ~/.ssh/your-Keypair.pem
```

SSH (and Ansible) will refuse to use a key file with open permissions.

### 9.3 Load the key into SSH agent

```bash
eval $(ssh-agent -s)
ssh-add ~/.ssh/your-Keypair.pem

# Confirm the key is loaded
ssh-add -l
```

### 9.4 Pre-trust each host

Fresh EC2 instances have unknown fingerprints. SSH blocks connections to untrusted hosts with `Host key verification failed`. SSH into each server once to accept the fingerprint:

```bash
ssh -i ~/.ssh/your-Keypair.pem ec2-user@172.31.24.97   # Web 1
ssh -i ~/.ssh/your-Keypair.pem ec2-user@172.31.18.88   # Web 2
ssh -i ~/.ssh/your-Keypair.pem ec2-user@172.31.22.134  # NFS
ssh -i ~/.ssh/your-Keypair.pem ubuntu@172.31.21.250    # DB
ssh -i ~/.ssh/your-Keypair.pem ubuntu@172.31.16.84     # LB
```

Type `yes` at each fingerprint prompt. Each server's fingerprint is added to `~/.ssh/known_hosts`.

---

## Step 10 — GitOps Workflow: Commit, PR, and Merge

```bash
git status
git add .
git commit -m "DEL-3 Add inventory structure and common playbook"
git push -u origin feature/DEL-3-ansible-setup
```
![14-git-commit.png](images/14-git-commit.png)
![15-branch.png](images/15-git-branch.png)

On GitHub:

1. Navigate to the repository; GitHub will prompt to compare the branch
2. Click **Compare & pull request**
3. Base: `main` | Compare: `feature/DEL-3-ansible-setup`
4. Review the diff as if you were a second engineer
5. Merge the pull request

Pull the updated `main` back to the server:

![15-branch-view.png](images/15-branch-view.png)
![16-create-pull-request.png](images/16-create-pull-request.png)
![17-merge-pull-request.png](images/17-merge-pull-request.png)
![18-confirm-merge.png](images/18-confirm-merge.png)
![19-merge-succssful.png](images/19-merge-successful.png)

```bash
git checkout main
git pull origin main
```

Jenkins detects the merge to `main` and triggers an automatic build. Verify:

```bash
ls /var/lib/jenkins/jobs/ansible/builds/
```

A new build number should appear.

---

## Step 11 — Test Connectivity Before Running the Playbook

Always verify Ansible can reach all hosts before executing tasks:

```bash
ansible all -i inventory/dev.yml -m ping
```

Expected output — all five hosts returning `pong`:

```
172.31.24.97  | SUCCESS => {"ping": "pong"}
172.31.18.88  | SUCCESS => {"ping": "pong"}
172.31.22.134 | SUCCESS => {"ping": "pong"}
172.31.16.84  | SUCCESS => {"ping": "pong"}
172.31.21.250 | SUCCESS => {"ping": "pong"}
```
![20-ansible-playbook-inventory.png](images/20-ansible-playbook-inventory.png)

If any host returns `UNREACHABLE`, the issue is in the SSH layer — check the key, user, and known_hosts before investigating the playbook.

---

## Step 12 — Execute the Playbook

```bash
cd ansible-config-mgt
ansible-playbook -i inventory/dev.yml playbooks/common.yml
```
![20-ansible-playbook-inventory-dev-playbooks-common.png](images/20-ansible-playbook-inventory-dev-playbooks-common.png)

A successful run shows all tasks as `ok` or `changed`, with `failed=0` and `unreachable=0` in the final play recap.

---

## Step 13 — Verify the End State

Automation is only reliable when the expected end state is verifiable. SSH into each server and confirm all tasks completed correctly.

**Web Server 1:**
```bash
ssh -i ~/.ssh/your-Keypair.pem ec2-user@172.31.24.97
which wireshark
timedatectl
ls /opt/devops
cat /opt/devops/info.txt
```

**Web Server 2:**
```bash
ssh -i ~/.ssh/your-Keypair.pem ec2-user@172.31.18.88
which wireshark && timedatectl && ls /opt/devops
```
![21-verify-wireshark.png](images/21-verify-wireshark.png)

**NFS Server:**
```bash
ssh -i ~/.ssh/your-Keypair.pem ec2-user@172.31.22.134
which wireshark && timedatectl && ls /opt/devops
```

**Database Server:**
```bash
ssh -i ~/.ssh/your-Keypair.pem ubuntu@172.31.21.250
which wireshark && timedatectl && ls /opt/devops
```
![21-verify-wireshark-servers.png](images/21-verify-wirshark-servers.png)

**Load Balancer:**
```bash
ssh -i ~/.ssh/your-Keypair.pem ubuntu@172.31.16.84
which wireshark && timedatectl && cat /opt/devops/info.txt
```
![21-verify-wireshark-final.png](images/21-verify-wireshark-final.png)

**Expected state on all servers:**

| Check | Expected Result |
|---|---|
| `which wireshark` | `/usr/bin/wireshark` |
| `timedatectl` | `Time zone: Africa/Lagos` |
| `ls /opt/devops` | `info.txt` listed |
| `cat /opt/devops/info.txt` | File exists with content |

These checks also confirm **idempotency**; running the playbook again after verification produces zero changes, because the desired state is already met.

---

## Troubleshooting Log

### Jenkins GitHub Authentication Failure

**Error:** `Failed to connect to repository: git ls-remote HEAD`

**Root cause:** GitHub deprecated password authentication for Git operations. Jenkins had no valid credentials configured.

**Fix:** Generated a GitHub PAT with `repo` scope. Added to Jenkins at **Manage Jenkins → Credentials → Global → Add Credentials** as `Username with password`. Selected this credential in the job's SCM configuration.

---

### Ansible SSH Failures

**Error:** `UNREACHABLE! => Permission denied (publickey)`

**Root cause analysis:**
- DB server was a Ubuntu AMI but inventory specified `ansible_ssh_user=ec2-user` — the `ec2-user` account does not exist on Ubuntu AMIs, so authentication fails before the key is even checked.
- Private key `My-S***-Keypair.pem` existed on the local laptop but was absent from the Jenkins-Ansible control node.
- Fresh EC2 instances have unknown fingerprints, causing `Host key verification failed`.

**Fixes:**
1. Corrected SSH user per host to match the AMI type.
2. Copied the private key to `~/.ssh/` on the control node via `scp`, then set `chmod 400`.
3. Pre-trusted each host with a manual SSH connection, accepting fingerprints into `~/.ssh/known_hosts`.

**Core lesson:** Ansible uses SSH as its transport layer. Every Ansible connectivity failure is an SSH failure first — always debug SSH before investigating the playbook logic.

---

### Ansible Binary Not Found After pip Install

**Error:** `ansible: command not found` immediately after successful `pip install ansible`

**Root cause:** pip installs binaries to `~/.local/bin`, which is not included in `$PATH` on some Ubuntu configurations by default.

**Fix:**
```bash
export PATH=$PATH:/home/ubuntu/.local/bin
echo 'export PATH=$PATH:/home/ubuntu/.local/bin' >> ~/.bashrc
source ~/.bashrc
ansible --version  # Confirm fix
```

---

## Key Engineering Concepts Demonstrated

**Idempotency**; Ansible tasks describe desired state, not imperative commands. Running the playbook twice leaves the system unchanged on the second run if it is already compliant.

**Cross-platform compatibility**; `when: ansible_os_family` guards allow a single playbook to manage RHEL and Ubuntu hosts without maintaining separate files per OS type.

**Bastion Host pattern**; All Ansible connections originate from one controlled node. Private-subnet servers are never exposed to the internet directly, reducing the attack surface.

**SSH agent forwarding**; The control node authenticates using key material without storing a decrypted copy on the jump host, preserving key security across hops.

**GitOps discipline**; Infrastructure changes follow the same review process as application code: branch → commit with ticket reference → PR → review → merge → automated apply.


---

## Author

Oluwasomidotun Adepitan

Email: Anuoluwapodotun@gmail.com

Linkedln: Https://www.linkedin.com/in/oluwasomidotun-adepitan