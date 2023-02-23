# Linux Server Playbooks

Ansible playbooks to configure linux servers for various development or application hosting tasks. My goal is for these playbooks and scripts to be as cloud vendor agnostic as possible, and let me get a Linux server configured quickly and easily.

Most of the playbooks are set up to work with Rocky Linux (RHEL) or Ubuntu (Debian) distributions. 

At the moment, the playbooks use only core [ansible](https://github.com/ansible/ansible), without external extensions or user add ons. This is a personal preference, I like to keep things simple and minimize dependencies wherever possible.

`pip install ansible` and whatever cloud provider CLI are the only dependencies. 

## Requirements

1. Ansible (`ansible [core 2.14.2]`)
2. Hashed password for the new ansible user (created with the `playbooks/ansible_user.yml`) playbook. Create the password (on linux) with `mkpasswd <your password> --method=sha-512` and save the hash in the `PASSWORD_HASH` environmental variable

## Contents

| Script/Playbook              | Purpose                                                              | Compatible with |
|------------------------------|----------------------------------------------------------------------|-----------------|
| `playbooks/ansible_user.yml` | Create the ansible user and add ansible ssh key for other playbooks  | Ubuntu, Rocky   |
| `playbooks/config.yml`       | Set hostname, update packages, check defaults, configure fail2ban    | Ubuntu, Rocky   |
| `playbooks/docker.yml`       | Install and configure Docker, configure `firewalld` for docker swarm | Rocky           |
| `playbooks/rust.yml`         | In progress                                                          |                 |
| `playbooks/postgres.yml`     | In progress                                                          |                 |
| `playbooks/valgrind.yml`     | In progress (for performance profiling/benchmarking)                 |                 |

## Inventory

The `inventory` file defines two variables for each server, and looks like so:

```shell
[servers]
<ip-addr> ansible_user=ansible new_hostname=<your-new-hostname> 
#...
```

# New Server Configuration

When the server starts up (ansible and main SSH public keys already added via cloud provider), I add the IP addresses to `inventory` and the first playbook I run uses `root` to login and create a new user with password-less sudo. 

So once I've spun up the servers, I'd run:

```shell
source .secrets # to set the PASSWORD_HASH variable. previously made with mkpasswd
ansible-playbook playbooks/ansible_user.yml --extra-vars='ansible_user=root'
```

This creates a new user for future ansible playbooks with username `ansible` and the password set to what is stored in `PASSWORD_HASH`. For now, user has password-less sudo. Further playbooks are run with:

```shell
ansible-playbook playbooks/config.yml
```

# Example with a Digital Ocean Ubuntu Droplet

Startup and configure a Digital Ocean Ubuntu Droplet ([Reference](https://docs.digitalocean.com/reference/doctl/reference/compute/droplet/create/))

```shell
doctl compute droplet create perf-dev \
    --image ubuntu-22-04-x64 \ # rocky is rockylinux-9-x64
    --size s-1vcpu-2gb \
    --region nyc1 \
    --ssh-keys id-here,other-id-here \
    --wait \
    --format ID,Name,Public\ IPv4
```

For the flags, use the following commands to get available options and use the `Slug` field in the output (except for `ssh-keys`, in that case use the ID):

| Flag         | doctl command                                                  |
|--------------|----------------------------------------------------------------|
| `--image`    | `doctl compute image list --public --format Slug,Distribution` |
| `--size`     | `doctl compute size list`                                      |
| `--region`   | `doctl compute region list`                                    |
| `--ssh-keys` | `doctl compute ssh-key list`                                   |

The command should return the IP address and name, but otherwise you can check the status and get the new IP address using:

```
doctl compute droplet list --format ID,Name,Public\ IPv4
```

Add the IP to `inventory` and use the ansible playbook: `config.yml` via:

```shell
source .secrets # to set the PASSWORD_HASH variable. previously made with mkpasswd
ansible-playbook playbooks/ansible_user.yml --extra-vars='ansible_user=root'
ansible-playbook playbooks/config.yml
```

Now run other ansible playbooks as needed.

# Example with an AWS EC2 Instance

https://mrunalgorde.medium.com/how-to-use-aws-cli-to-launch-an-ec2-instance-f68273a749ef

EC2 is more complicated from the CLI. You need a VPC, security group, SSH key, image details, and instance type. The command looks like:

```shell
aws ec2 run-instances \
    --image-id $UBUNTU \
    --instance-type t2.medium \
    --count 1 \
    --key-name ansible@aws \
    --security-group-ids $GroupId 
    
# get the IP

aws ec2 describe-instances
```

Add the IP to `inventory` and confirm ansible can ping via: `ansible all -m ping` and use the ansible playbook: `config.yml` via:

Where $USER is whatever platform user, for this AMI it is 'ubuntu'

```shell
source .secrets # to set the PASSWORD_HASH variable. previously made with mkpasswd
ansible-playbook playbooks/ansible_user.yml --extra-vars='ansible_user=$USER'
ansible-playbook playbooks/config.yml
```

Now run other ansible playbooks as needed.

Below explains how to setup the security group, SSH key, image, and instance type to prepare the command above.

## SSH Keys

First, add a key pair or list existing key pairs:

```shell
# Add key pair
aws ec2 import-key-pair --key-name 'ansible@aws' --public-key-material fileb://path/to/ssh/ansible_ed.pub
# or list:
aws ec2 describe-key-pairs
```
Save the `KeyPairId`

## VPC

I'll use my default VPC, you can list all VPCs with:

```shell
aws ec2 describe-vpcs
```

Save the `VpcId`

To create a new VPC, see: [aws ec2 create-vpc](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-vpc.html)

## Security Group

[Create a new security group](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-security-group.html) (or list existing) and associate your VPC:

```shell
aws ec2 create-security-group --group-name linux-vm-sg --description "Default Linux SG" --vpc-id $VpcId
# List existing
aws ec2 describe-security-groups
```
Save the `GroupId`

Allow TCP and HTTP ingress ([aws ec2 authorize-security-group-ingress](https://docs.aws.amazon.com/cli/latest/reference/ec2/authorize-security-group-ingress.html))

```shell
# SSH
aws ec2 authorize-security-group-ingress \
    --group-id $GroupId \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0

# HTTP
aws ec2 authorize-security-group-ingress \
    --group-id $GroupId \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```
## AMI and instance type

From the console (Feb 2023), Ubuntu 22.04LTS is `ami-0557a15b87f6559cf` (saved as `UBUNTU`), and choose your [instance size](https://aws.amazon.com/ec2/instance-types/).

# Ansible and Server Reference

Shortcuts and commands for other useful functions

## Getting Ansible Facts

Get all facts from all servers:

```shell
ansible all -m gather_facts
```

Gather all facts from one server:

```shell
ansible all -m gather_facts --limit <IP ADDR>
```

Gather specific facts using a filter (in this case the prefix `ansible_distribution`):

```shell
ansible all -m gather_facts -a 'filter=ansible_distribution*'
```

Run just one task from a playbook based on a tag

```shell
ansible-playbook playbooks/rust.yml --tags "config"
```
Note that when you `become: true`, you become `root`. You can specify:

```
become: yes
become_user: "{{ ansible_user }}"
```
To specify the user. 
x
## Checking fail2ban logs/jails

To check fail2ban jail logs (in this case for `sshd`):

```shell
fail2ban-client status sshd
```

