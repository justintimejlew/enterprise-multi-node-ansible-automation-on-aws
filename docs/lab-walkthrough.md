## Lab Execution Walkthrough

### 1. Infrastructure launch (AWS EC2)
- Launch one t3.small instance for control node and three t3.micro instances for managed nodes in same VPC. CentOS is located under **AWS Marketplace AMIs**.
    - Recommend creating a **Launch Template**
    - When prompted, save key pair into local host `~/.ssh` directory.
        - By using a **Key Pair**, this only allows the host with the key pair to SSH.
- Security Group: On managed nodes, only allow SSH only from control node after it has been created.
- User data: package pre-install (ansible-core on control, python3 on managed).

    <img src="https://raw.githubusercontent.com/justintimejlew/enterprise-multi-node-ansible-automation-on-aws/refs/heads/main/docs/Screenshots/1.png"/>

### 2. Control node access and prep
- Run `ssh -i ~/.ssh/ansible-control.pem ec2-user@<publicipaddress>` to access `ansible-control` node.
- Run `sudo cat /var/log/cloud-init-output.log | tail -n 30` on control node to confirm user data script ran successfully.

### 3. SSH key distribution (critical step)
Run the following commands:
- `ssh-keygen -t ed25519`
- `ssh-copy-id ansible@managed-web`
- Repeated this step for `db` and `server`
    - Configure `/etc/hosts` file on control node prior to SSH
    - Remember the **Key Pair** must be saved on your control node as well as your local host.
    - Run `cat ~/.ssh/id_ed25519.pub | ssh -i ~/.ssh/ansible-control.pem ec2-user@managed-web 'sudo mkdir -p /home/ansible/.ssh && sudo tee -a /home/ansible/.ssh/authorized_keys'` if you run into issues copying the public key to the ansible user.

### 4. First connectivity test

- Run `ansible all -i inventory.ini -m ping` and the results should show:
    
    <img src="https://raw.githubusercontent.com/justintimejlew/enterprise-multi-node-ansible-automation-on-aws/refs/heads/main/docs/Screenshots/2.png"/>

### 5. Bootstrap → Full deployment
- Run `ansible-playbook -i inventory.ini playbooks/bootstrap.yml` and the results should show:

    <img src="https://raw.githubusercontent.com/justintimejlew/enterprise-multi-node-ansible-automation-on-aws/refs/heads/main/docs/Screenshots/3.png"/>
- Run `ansible-playbook -i inventory.ini playbooks/site.yml` twice
    ## **idempotent run #1:**
    <img src="https://raw.githubusercontent.com/justintimejlew/enterprise-multi-node-ansible-automation-on-aws/refs/heads/main/docs/Screenshots/4.png"/>
    
    ## **idempotent run #2 (0 changes):**
    <img src="https://raw.githubusercontent.com/justintimejlew/enterprise-multi-node-ansible-automation-on-aws/refs/heads/main/docs/Screenshots/5.png"/>

### 6. Verification
- From control node or managed-db node: run `curl http://managed-web` → it should show "Enterprise Web Server - managed-web"
    - If it timeouts on the request, allow TCP/80  inbound traffic for the control node.
- From managed-db node run the following commands:
    - `systemctl status mariadb` → Should be enabled and active.
    - `ss -tulnp | grep 3306` → Is TCP listening on port 3306?
    - `sudo firewall-cmd --list-ports` → Does port 3306 show?