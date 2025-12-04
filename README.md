
# **Ansible Project 2 – End-to-End VM Monitoring & Reporting**

This project implements a **fully automated VM monitoring system** using Ansible. It dynamically collects **CPU, memory, and disk usage metrics** from AWS EC2 instances and generates a **consolidated animated HTML email report**. The entire setup is done on **Ubuntu EC2 instances**, leveraging Ansible for inventory management, task automation, and reporting.

---

## **Architecture Overview**

### **Tools & Workflow**

1. **EC2 Instances** – Ubuntu-based VMs, tagged for identification and filtering.

2. **Ansible Dynamic Inventory** – Uses the `amazon.aws.aws_ec2` plugin to automatically detect running instances.

3. **Playbooks**

   * `collect_metrics.yaml` – Gathers CPU, memory, and disk metrics from all EC2 instances.
   * `send_report.yaml` – Compiles metrics and sends an animated HTML email report.
   * `playbook.yaml` – Master playbook combining metrics collection and email reporting.

4. **Email Reporting**

   * Utilizes Ansible `mail` module.
   * HTML email template built with **Jinja2**, including animated visuals for metrics.
   * Timestamped using Ansible’s built-in `ansible_date_time`.

---

# **1. EC2 SETUP & SSH ACCESS**

### **Launch EC2 Instance**

* Ubuntu 22.04 LTS
* 2 vCPU / 4GB RAM recommended
* Download `.pem` file for SSH access

### **Move PEM File & Set Permissions**

```bash
cd ~/vm-monitor
cp /mnt/c/Users/<your-user>/Downloads/Ansible2.pem .
chmod 400 Ansible2.pem
```

### **SSH into EC2**

```bash
ssh -i Ansible2.pem ubuntu@<EC2-public-ip>
```

---

# **2. TAG EC2 INSTANCES**

### **Run Tag Script**

```bash
chmod +x tag.sh
./tag.sh
```

> Tags all EC2 instances sequentially for easier identification in inventory.

---

# **3. PASSWORDLESS SSH SETUP**

### **Generate SSH Key for Ansible Master**

```bash
ssh-keygen -t rsa -b 4096 -C "Ansible-Master"
```

### **Inject Key into EC2 Instances**

```bash
chmod +x copy-public-key.sh
./copy-public-key.sh
```

> This ensures Ansible can communicate with EC2 nodes **without password prompts**, enabling fully automated task execution.

---

# **4. DYNAMIC INVENTORY CONFIGURATION**

`inventory/aws_ec2.yaml` configuration:

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

> Automatically discovers **running EC2 instances** with `environment=dev` tags. Eliminates manual inventory updates.

---

# **5. EMAIL CONFIGURATION**

Set variables in `group_vars/all.yaml`:

```yaml
smtp_server: "smtp.gmail.com"
smtp_port: 587
email_user: "your_email@gmail.com"
email_pass: "your_app_password"
alert_recipient: "recipient_email@gmail.com"
```

> These are used by the **Ansible mail module** to send automated reports.

---

# **6. COLLECT VM METRICS**

### **Playbook: `collect_metrics.yaml`**

* Installs `sysstat` for CPU monitoring (`mpstat`)
* Collects:

  * CPU usage
  * Memory usage
  * Disk usage
* Stores metrics using `set_fact` for later use

### **Run Playbook**

```bash
ansible-playbook collect_metrics.yaml
```

> Aggregates metrics from all targeted EC2 instances dynamically.

---

# **7. SEND CONSOLIDATED EMAIL REPORT**

### **Playbook: `send_report.yaml`**

* Uses Ansible `mail` module
* Sends an **animated HTML report** using a Jinja2 template
* Includes timestamps via `ansible_date_time`

### **Run Playbook**

```bash
ansible-playbook send_report.yaml
```

> Ensures clear and visually appealing reporting of VM health.

---

# **8. MASTER PLAYBOOK**

`playbook.yaml` integrates both metrics collection and report sending:

```bash
ansible-playbook playbook.yaml
```

> One-command execution to **collect metrics and send reports** to the recipients.

---

# **9. PYTHON VIRTUAL ENVIRONMENT SETUP**

```bash
sudo apt install python3-venv -y
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install boto3 botocore docker
ansible-galaxy collection install amazon.aws
```

> Isolates Python dependencies for Ansible and AWS SDK libraries.

---

# **10. PROJECT STRUCTURE**

```
vm-monitor/
├── ansible.cfg                  # Ansible configuration
├── collect_metrics.yaml         # Playbook to collect VM metrics
├── send_report.yaml             # Playbook to send email report
├── playbook.yaml                # Master playbook
├── group_vars/
│   └── all.yaml                 # Global variables (email config)
├── inventory/
│   └── aws_ec2.yaml             # Dynamic AWS inventory
├── templates/
│   └── report_email_animated.html.j2  # Email template
├── tag.sh                       # EC2 tagging script
└── copy-public-key.sh            # SSH key injection script
```

---

# **11. VERSION CONTROL**

### **Push Project to GitHub**

```bash
git init
git add .
git commit -m "Initial commit: Ansible VM monitoring project"
gh repo create ansible-project-2 --public --source=. --remote=origin
git branch -M main
git push -u origin main


# **13. REFERENCES**

* [Ansible Docs – Dynamic Inventory](https://docs.ansible.com/ansible/latest/plugins/inventory/amazon_ec2.html)
* [Ansible Docs – Mail Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/mail_module.html)
* [Jinja2 Templates](https://jinja.palletsprojects.com/)
* [AWS CLI Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)

