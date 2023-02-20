# Linux Server Playbooks

Ansible playbooks and shell scripts to configure linux servers for various development or application hosting tasks. My goal is for these playbooks and scripts to be as cloud vendor agnostic as possible.

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
    --image ubuntu-22-04-x64 \
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

Add the IP to `inventory` and confirm ansible can ping via: `ansible all -m ping` and use the ansible playbook: `config.yml` via:

```shell
source .secrets # to set the PASSWORD_HASH variable. previously made with mkpasswd
ansible-playbook playbooks/ansible_user.yml --extra-vars='ansible_user=root'
ansible-playbook playbooks/config.yml
```

Now run other ansible playbooks as needed.

# Maintenance and reference

Shortcuts and commands for other useful functions

## Getting Ansible Facts

Get all facts from all servers:

```
ansible all -m gather_facts
```

Gather all facts from one server:

```
ansible all -m gather_facts --limit <IP ADDR>
```

Gather specific facts using a filter (in this case the prefix `ansible_distribution`):

```
ansible all -m gather_facts -a 'filter=ansible_distribution*'
```

## Checking fail2ban logs/jails

To check fail2ban jail logs (in this case for `sshd`):

```
fail2ban-client status sshd
```
