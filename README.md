# Ansible EC2 Apache Automation

This project automates:
- EC2 instance creation on AWS
- Dynamic inventory handling
- Apache installation
- Web hosting setup

## Technologies Used
- Ansible
- AWS EC2
- Ubuntu 24.04
- Apache2

## How to Run
```bash
ansible-playbook create-slave.yml


# Ansible on AWS — Project Report & Runbook

**Project:** Create an EC2 instance with Ansible and deploy a simple web site (Apache)

**Environment:** Ubuntu (controller), AWS us-east-1, Ansible (virtualenv), amazon.aws collection

---

## Summary

A complete end-to-end automation was created that:

1. Provisions a Ubuntu EC2 instance in `us-east-1` using Ansible.
2. Adds the new instance to a dynamic Ansible inventory.
3. SSHs to the instance and installs Apache.
4. Deploys a sample `index.html` and starts/enables the Apache service.

This runbook contains the working commands, the final playbook, and a brief troubleshooting section that explains the common problems and how to avoid them.

---

## Prerequisites (controller machine)

* Ubuntu controller (we used Ubuntu 24.04).
* An AWS account and an IAM user with programmatic access (Access key ID + Secret access key) with enough permissions to create   EC2, ENI, Security Groups, and Key Pairs.
* A keypair file (PEM) created in the target region or uploaded to AWS.
* Subnet and Security Group IDs for the chosen VPC. Ensure the security group allows SSH (22) and HTTP (80).

### Tools installed on the controller

Commands used to prepare the controller:

```bash
sudo apt update
sudo apt install -y ansible unzip python3-venv
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

**Create and activate a Python virtual environment (recommended):**

```bash
python3 -m venv aws-env
source aws-env/bin/activate
pip install --upgrade pip
pip install ansible boto3 botocore
```

**Install/upgrade the amazon.aws collection (inside venv):**

```bash
ansible-galaxy collection install amazon.aws --force --upgrade
ansible-galaxy collection list | grep amazon
```

**Configure AWS CLI** (in the controller):

```bash
aws configure
# set AWS Access Key, Secret, region -> us-east-1, output -> json
aws configure get region
```

---

## Final Playbook (copy this as `create-slave.yml`)

Refer create-slave.yml

## Creating Key Pair
aws ec2 create-key-pair \
  --key-name ubuntu-slave-key \
  --query "KeyMaterial" \
  --output text > ubuntu-slave-key.pem \
  --region us-east-1

## Permission
chmod 400 ubuntu-slave-key.pem


## How to run

1. Activate your virtualenv (if used):

```bash
source aws-env/bin/activate
```

2. Ensure `create-slave.yml` is in your current directory and the private key `ubuntu-slave-key.pem` is present and permission-protected:

```bash
ls -l ubuntu-slave-key.pem
chmod 600 ubuntu-slave-key.pem
```

3. Run the playbook with the venv ansible-playbook (to avoid running system Ansible):

```bash
/home/ubuntu/aws-env/bin/ansible-playbook create-slave.yml -vvv
```

**After success**, open the browser to the printed IP (example: `http://3.219.233.201`) to see the sample web page.

---

## Troubleshooting — common errors we encountered (and fixes)

> This section documents the problems we hit during the session and the concrete fixes. Keep these steps handy — they are the fastest way to diagnose similar issues.

### 1. `ValueError: The following group names are not valid` (security group ID passed as name)

**Cause:** Older module syntax expected SG *names* instead of IDs or different param names.
**Fix:** Use the `network:` block or `security_groups:` with IDs depending on module version. For our final module (amazon.aws 10.x) we used `security_groups:` with the SG ID under `vpc_subnet_id`.

### 2. `TypeError: 'NoneType' object is not subscriptable` in build_network_spec

**Cause:** Old/partial amazon.aws collection or corrupt installation.
**Fix:** Clean out old collections and reinstall `amazon.aws` inside the virtualenv:

```bash
rm -rf ~/.ansible/collections/ansible_collections/amazon
ansible-galaxy collection install amazon.aws --force --upgrade
```

### 3. `pip3 install` blocked by "externally-managed-environment"

**Cause:** Ubuntu system Python prevented global pip installs.
**Fix:** Create and use a Python virtualenv (python3 -m venv aws-env). Install boto3/botocore/ansible inside the venv.

### 4. Collection compatibility issues (Ansible core vs collection)

**Cause:** System Ansible and collection versions mismatch.
**Fix:** Install Ansible inside the venv (`pip install ansible`) and ensure you run the venv `ansible-playbook` binary.

### 5. `InvalidAMIID.NotFound` or `InvalidAMIID.Malformed`

**Cause:** Using an AMI not accessible via API (marketplace/private) or AMI from a different region.
**Fix:** Use the public Canonical AMIs for the region. Example used: `ami-02e2d51e863c5c870` for `us-east-1`. Verify with:

```bash
aws ec2 describe-images --image-ids ami-02e2d51e863c5c870 --region us-east-1
```

### 6. `security_group_ids` not supported (Unsupported parameters)

**Cause:** Collection/module variant only accepts `security_groups` or `vpc_subnet_id`.
**Fix:** Use the parameters the installed module supports (in our case `security_groups:` + `vpc_subnet_id:`).

### 7. `InvalidParameterCombination` "instance type not eligible for Free Tier"

**Cause:** `t2.micro` not allowed or account not eligible.
**Fix:** Use `t3.micro` (free-tier eligible in most accounts) or an allowed instance type.

### 8. `ssh_askpass`: No such file or directory
Host key verification failed

1.Ansible tried to SSH into the new EC2

2.SSH asked: “Do you trust this new server?”

3.Since this is automation, it cannot click “yes”

4.SSH failed → Ansible marked host as UNREACHABLE

Fix it with below command

## Fix:**
DISABLE HOST KEY CHECKING (MANDATORY)
echo -e "[defaults]\nhost_key_checking = False" > ansible.cfg

---



## Final state (what we achieved)

* EC2 instance created (ID `i-0691a35a0e31b58c6`) in `us-east-1` with public IP `3.219.233.201`.
* Apache installed and sample page deployed to `/var/www/html/index.html`.
* Playbook is reproducible and safe to rerun (idempotent for the tasks used).

---

## Next steps / Improvements (suggested)

* Replace the sample static page with a deployment from a git repo.
* Use roles to split responsibilities (ec2 provisioning role, webserver role).
* Add tags and make the instance creation conditional for re-runs.
* Add proper error handling and logging (Ansible callbacks).
* Destroy resources: create a tear-down playbook to terminate the EC2 and clean up created resources.

---

## Useful commands (quick cheatsheet)

```bash
# Activate environment
source aws-env/bin/activate

# Run playbook
/home/ubuntu/aws-env/bin/ansible-playbook create-slave.yml -vvv

# Describe images (filter Canonical)
aws ec2 describe-images --owners 099720109477 --filters "Name=name,Values=*ubuntu-noble-24.04*" --region us-east-1 --query "Images[*].[ImageId,Name]" --output table

# SSH into host
ssh -i ubuntu-slave-key.pem ubuntu@3.219.233.201

# Terminate instance (if needed)
aws ec2 terminate-instances --instance-ids i-0691a35a0e31b58c6 --region us-east-1
```

---
