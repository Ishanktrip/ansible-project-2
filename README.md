
# **Ansible Project 2 – End-to-End VM Monitoring & Reporting**

This project implements a **fully automated VM monitoring system** using Ansible. It dynamically collects **CPU, memory, and disk usage metrics** from AWS EC2 instances and generates a **consolidated animated HTML email report**.

The entire setup is done on **Ubuntu EC2 instances**, leveraging Ansible for inventory management, automation, and reporting. This project demonstrates **end-to-end DevOps automation**, from provisioning to monitoring and reporting.

---

## **Architecture Overview**

### **Tools & Workflow**

1. **EC2 Instances** – Ubuntu-based VMs tagged for identification and filtering.
   *(1 Ansible master node, 10 target nodes)*

2. **Ansible Dynamic Inventory** – Uses the `amazon.aws.aws_ec2` plugin to automatically detect running instances. This avoids manual inventory updates every time a VM is added or removed.

3. **Playbooks**

   * `collect_metrics.yaml` – Collects CPU, memory, and disk metrics from all EC2 instances.
   * `send_report.yaml` – Compiles metrics and sends an animated HTML email report.
   * `playbook.yaml` – Master playbook that combines both metrics collection and report sending for **one-command execution**.

4. **Email Reporting**

   * Uses Ansible `mail` module for automated delivery.
   * HTML email template built with **Jinja2**, including animated visualizations for better readability.
   * Timestamped using Ansible’s built-in `ansible_date_time` to track reporting periods.

> This workflow allows you to **monitor multiple VMs in real-time** and send consolidated reports automatically, eliminating manual checks.

---

# **1. EC2 SETUP & SSH ACCESS**

### **Launch EC2 Instances**

* OS: Ubuntu 22.04 LTS
* Recommended: 2 vCPU / 4GB RAM for monitoring small to medium workloads
* Download the `.pem` file, which allows **secure SSH access** to the EC2 instance

> Keep the `.pem` file safe. Losing it means losing SSH access unless a new key pair is created.

### **Step 1: Update the System**

```bash
sudo apt update && sudo apt upgrade -y
```

> Ensures all packages are up-to-date and security patches are applied. Important for stability and security.

### **Step 2: Add Ansible PPA**

```bash
sudo add-apt-repository --yes --update ppa:ansible/ansible
```

> Ansible’s official PPA provides the **latest stable version**, including new modules not available in Ubuntu’s default repositories.

### **Step 3: Install Ansible**

```bash
sudo apt install ansible -y
```

> Installs Ansible and its dependencies. This becomes the **control node** from where we manage all target VMs.

### **Move PEM File & Set Permissions**

```bash
cd ~/vm-monitor
cp /mnt/c/Users/<your-user>/Downloads/Ansible2.pem .
chmod 400 Ansible2.pem
```

> SSH requires strict permissions on the key file (`chmod 400`) to avoid authentication errors.

### **SSH into EC2**

```bash
ssh -i Ansible2.pem ubuntu@<EC2-public-ip>
```

> Connect securely to the EC2 instance using the `.pem` key.

---

# **2. TAG EC2 INSTANCES**

Tagging instances allows you to **identify, filter, and group EC2 instances** for automation.

1. Create a tagging script:

```bash
vi tag.sh
```
write the tagging script into the file 

2. Install AWS CLI (if not installed):

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

> AWS CLI allows you to **interact with AWS resources programmatically**. Configuring it with credentials is required for tagging scripts to work.

3. Run the tagging script:

```bash
chmod +x tag.sh
./tag.sh
```

> Tags EC2 instances sequentially. Tags like `Name` and `Environment` make **dynamic inventory filtering** easier.

---

# **3. PASSWORDLESS SSH SETUP**

For Ansible automation, **passwordless SSH** is critical; otherwise, playbooks cannot execute tasks automatically.

### **Generate SSH Key for Ansible Master**

```bash
ssh-keygen -t rsa -b 4096 -C "Ansible-Master"
```

> Creates a public/private key pair on the Ansible master node.

### **Copy Public Key to EC2 Nodes**

Create a script that copies the public key to all target nodes:

```bash
vi copy-public-key.sh
chmod +x copy-public-key.sh
./copy-public-key.sh
```

> After this, Ansible can **connect to all EC2 instances without passwords**, enabling fully automated playbooks.

---

# **4. DYNAMIC INVENTORY CONFIGURATION**

Dynamic inventory allows Ansible to **automatically discover all active EC2 instances** without manual updates.

1. Create inventory directory:

```bash
mkdir inventory
vi inventory/aws_ec2.yaml
```

2. Add configuration:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:environment: dev
  instance-state-name: running
compose:
  ansible_host: public_ip_address
keyed_groups:
  - key: tags.Name
    prefix: name
  - key: tags.environment
    prefix: env
```

> Any new EC2 instance with `environment=dev` is **automatically included** in the inventory.

---

# **5. PYTHON VIRTUAL ENVIRONMENT SETUP**

An isolated Python environment ensures **package conflicts are avoided**.

```bash
sudo apt install python3-venv -y
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install boto3 botocore docker
ansible-galaxy collection install amazon.aws
ansible-inventory -i inventory/aws_ec2.yaml --graph
```

> Installs required AWS SDKs (`boto3`) and Docker integration for playbooks that may need container operations.

---

# **6. EMAIL CONFIGURATION**

Create global email variables for reporting:

```bash
mkdir group_vars
vi group_vars/all.yaml
```

### **Example Variables**

```yaml
smtp_server: "smtp.gmail.com"
smtp_port: 587
email_user: "your_email@gmail.com"
email_pass: "your_app_password"
alert_recipient: "recipient_email@gmail.com"
```

> These variables allow **automated report emails**. Gmail users must create an App Password for security.

---

# **7. COLLECT VM METRICS**

Create the metrics collection playbook:

```bash
vi collect_metrics.yaml
```

### **Example Playbook: `collect_metrics.yaml`**

```yaml
---
- name: Collect VM Metrics
  hosts: all
  gather_facts: no
  become: yes

  tasks:
    - name: Install sysstat for CPU monitoring
      apt:
        name: sysstat
        state: present
        update_cache: yes

    - name: Collect CPU usage
      command: mpstat 1 1
      register: cpu_usage

    - name: Collect Memory usage
      command: free -m
      register: memory_usage

    - name: Collect Disk usage
      command: df -h
      register: disk_usage

    - name: Set metrics facts
      set_fact:
        vm_metrics:
          cpu: "{{ cpu_usage.stdout }}"
          memory: "{{ memory_usage.stdout }}"
          disk: "{{ disk_usage.stdout }}"
```

> This playbook collects **CPU, memory, and disk usage** from all target nodes and stores them for reporting.

---

# **8. SEND CONSOLIDATED EMAIL REPORT**

Create the email reporting playbook:

```bash
vi send_report.yaml
```

### **Example Playbook: `send_report.yaml`**

```yaml
---
- name: Send VM Metrics Report
  hosts: localhost
  gather_facts: no

  vars_files:
    - group_vars/all.yaml

  tasks:
    - name: Render HTML report
      template:
        src: templates/report_email_animated.html.j2
        dest: /tmp/vm_report.html
      vars:
        vm_metrics: "{{ hostvars | dict2items | map(attribute='value.vm_metrics') | list }}"

    - name: Send Email Report
      mail:
        host: "{{ smtp_server }}"
        port: "{{ smtp_port }}"
        username: "{{ email_user }}"
        password: "{{ email_pass }}"
        to: "{{ alert_recipient }}"
        subject: "Automated VM Metrics Report - {{ ansible_date_time.date }}"
        body: "Please find attached the latest VM metrics report."
        attach: /tmp/vm_report.html
        secure: starttls
```

> Uses **Jinja2 templates** to generate an animated HTML report and sends it via email.

---

# **9. MASTER PLAYBOOK**

`playbook.yaml` combines **metric collection** and **report sending**:

```bash
vi playbook.yaml
```

```yaml
---
- import_playbook: collect_metrics.yaml
- import_playbook: send_report.yaml
```

> Running this executes **end-to-end monitoring and reporting** in one step:

```bash
ansible-playbook playbook.yaml
```

---

# **10. EMAIL TEMPLATE**

```bash
mkdir templates
vi templates/report_email_animated.html.j2
```

> Store HTML + Jinja2 placeholders for CPU, memory, and disk metrics. This template is used to generate the **animated email report**.

---

# **11. PROJECT STRUCTURE**

```
vm-monitor/
├── ansible.cfg                        # Ansible configuration
├── collect_metrics.yaml               # Playbook to collect VM metrics
├── send_report.yaml                   # Playbook to send email report
├── playbook.yaml                      # Master playbook
├── group_vars/
│   └── all.yaml                       # Global variables (email config)
├── inventory/
│   └── aws_ec2.yaml                   # Dynamic AWS inventory
├── templates/
│   └── report_email_animated.html.j2  # Email template
├── tag.sh                             # EC2 tagging script
└── copy-public-key.sh                  # SSH key injection script
```

---

# **12. VERSION CONTROL**

```bash
git init
git add .
git commit -m "Initial commit: Ansible VM monitoring project"
gh repo create ansible-project-2 --public --source=. --remote=origin
git branch -M main
git push -u origin main
```

> Push your project to GitHub for version control and collaboration.

---

# **13. REFERENCES**

* [Ansible Docs – Dynamic Inventory](https://docs.ansible.com/ansible/latest/plugins/inventory/amazon_ec2.html)
* [Ansible Docs – Mail Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/mail_module.html)
* [Jinja2 Templates](https://jinja.palletsprojects.com/)
* [AWS CLI Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)


