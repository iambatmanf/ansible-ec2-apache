# Ansible EC3 Apache Automation

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


## Creating Key Pair
aws ec2 create-key-pair \
  --key-name ubuntu-slave-key \
  --query "KeyMaterial" \
  --output text > ubuntu-slave-key.pem \
  --region us-east-1

chmod 400 ubuntu-slave-key.pem


## If you get below error

ssh_askpass: No such file or directory
Host key verification failed

1.Ansible tried to SSH into the new EC2

2.SSH asked: “Do you trust this new server?”

3.Since this is automation, it cannot click “yes”

4.SSH failed → Ansible marked host as UNREACHABLE

Fix it with below command

DISABLE HOST KEY CHECKING (MANDATORY)
echo -e "[defaults]\nhost_key_checking = False" > ansible.cfg


## Useful commands

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

