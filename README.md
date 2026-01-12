# **Enterprise Multi-Node Ansible Automation Lab**

This lab demonstrates enterprise-grade Ansible automation across a multi-node environment. It showcases best practices including inventory management, role-based playbooks, idempotency, security hygiene, and orchestration hosted on AWS with user data.

## **How This Benefits Companies:**
### 👩🏽‍💻 Engineers

This lab demonstrates how Ansible enables consistent, repeatable, and scalable system configuration across multiple servers, eliminating configuration drift and manual setup errors. By automating user management, package installation, service configuration, and basic system hardening, engineers can provision environments faster, recover systems reliably, and focus on higher-value engineering work instead of repetitive operational tasks. The use of idempotent playbooks and roles reflects real-world infrastructure practices used in production environments.

### 📊 Managers

Automating system configuration with Ansible reduces operational risk and improves team efficiency by standardizing how servers are built and maintained. This approach shortens onboarding time for new environments, minimizes outages caused by misconfiguration, and ensures systems remain compliant with security and operational standards. The lab shows how infrastructure automation directly lowers support costs while increasing reliability and delivery speed.

### 🧠 Executives
This automation capability supports business continuity, scalability, and cost control by ensuring critical systems are deployed consistently and securely as the organization grows. Automated infrastructure reduces downtime, accelerates deployment timelines, and lowers dependence on manual processes that introduce risk. Ultimately, this approach enables faster time-to-market, stronger security posture, and more predictable IT operations aligned with business objectives.

## Target Architecture

```
ansible-control
├── managed-web
├── managed-db
└── managed-server
```

- **OS**: CentOS Stream 9 (or RHEL 9 compatible)
- **Infrastructure**: AWS EC2
- **Control Node**: `ansible-control` (t3.small recommended)
- **Managed Nodes**:
  - `managed-web` (t3.micro) → Apache web server (Nginx is optional)
  - `managed-db` (t3.micro) → MariaDB database (PostgreSQL is optional)
  - `managed-server` (t3.micro) → Generic server (common config only)
- **Networking**: All nodes in same VPC, same key pair, Security Group allows SSH only from control node
- **Access**: Non-root user `ansible` with passwordless sudo.

## Repository Structure

```
enterprise-ansible-lab/
├── inventory.ini
├── ansible.cfg
├── playbooks/
│   ├── site.yml
│   ├── bootstrap.yml
│   └── hardening.yml  # (future use)
├── roles/
│   ├── common/
│   ├── webserver/
│   └── database/
└── README.md
```

## Roles Explained

### common (Applied to all nodes)
- Creates `devops` group
- Creates standard user `deploy`
- Grants passwordless sudo to `devops` group via `/etc/sudoers.d/devops`

### webserver (Applied to managed-web)
- Installs and enables Apache (`httpd`)
- Deploys simple index page with hostname
- Opens HTTP port in firewalld

### database (Applied to managed-db)
- Installs MariaDB server and Python MySQL driver
- Starts and enables MariaDB service
- Opens port 3306/tcp in firewalld

## Security Practices Demonstrated

- EC2 instances booted with user data for consistent installation of base packages
- Key-based SSH only (no passwords)
- Non-root automation user with targeted sudo
- Firewalld enabled and configured with least-privilege ports
- Separate service-specific roles
- Idempotent playbooks (safe to run multiple times)

## How to Run

1. Configure AWS EC2 instances using user data to install Ansible and requirements during boot process. Recommended to create a key pair to be used across all nodes.
   * Control node user data (t3.small):

      ```
      #!/bin/bash
      dnf update -y
      dnf install -y ansible-core python3 python3-pip policycoreutils-python-utils 
      pip3  install ansible-navigator
      ansible-galaxy collection install ansible.posix
      ```
   * Managed nodes user data (t3.micro):

      ```
      #!/bin/bash
      dnf update -y
      dnf install -y python3 policycoreutils-python-utils
      useradd -m -s /bin/bash ansible
      echo "ansible ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/ansible
      chmod 0440 /etc/sudoers.d/ansible
      ```

2. Manually copy SSH key from control node
   * SSH in as `ec2-user` (default), then switch user to `ansible`
   * Run `ssh-keygen -t ed25519` for 
   * Then copy public key to all managed nodes for created user `ansible` and test SSH ability
      * `ssh-copy-id ansible@managed-web`
      * `ssh-copy-id ansible@managed-db`
      * `ssh-copy-id ansible@managed-server`
3. Clone this repo and cd into it
4. Test connectivity with managed EC2 instances by running: `ansible all -i inventory.ini -m ping`
5. Run Bootstrap nodes:
   ```bash
   ansible-playbook -i inventory.ini playbooks/bootstrap.yml
   ```
6. Deploy full configuration:
   ```bash
   ansible-playbook -i inventory.ini playbooks/site.yml
   ```

Run `site.yml` twice to verify idempotency.

## Verification

- Web: Access `http://<managed-web-public-ip>/` → Should show "Enterprise Web Server - managed-web"
- DB: Check if the service is running, enabled at boot, and has the appropriate firewall rules.

## Stretch Goals (Planned for later)

- Add secure MariaDB hardening (remove anonymous users, set root password)
- Implement Vault for secrets
- Add Apache/NGINX reverse proxy + Let's Encrypt
- CIS hardening role
- Monitoring with Prometheus/Node Exporter

This lab reflects real-world enterprise Ansible patterns used in production environments.