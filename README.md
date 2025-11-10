# Ansible-Proxmox-VM-Automation

The repo serves as a demonstration for the creation of pre configured RHEL9.6 virtual machines in a proxmox virtual environment.  The playbooks included will create the vms according to specified variables and do post configuration of the Java Runtime Environment.

## Prequisites

### Software

- Proxmox VE: Version 8.0 or higher  (Tested on Proxmox 8.4)
- Ansible: Version 2.9 or higher (Tested on 2.19.4)
- Python: Version 3.6 or higher (Tested on 3.13.9)

### Ansible Collections

```bash
ansible-galaxy collection install community.proxmox
```

### Clone Template

You must have __qemu-gust-agent__ installed on your template VM

## Setup Instructions

### Step 1: Create Proxmox API User

For this to work, we create a dedicated API user for Ansible.

```bash
# Create API user
pveum user add ansible@pve

# Set password for the user
pveum passwd ansible@pve

# Grant full administrative permissions
pveum aclmod / -user ansible@pve -role Administrator

# Grant storage permissions
pveum aclmod /storage -user ansible@pve -role Administrator

# Grant SDN permissions (required for Proxmox 9.x)
pveum aclmod /sdn -user ansible@pve -role Administrator
```

#### Verify Permissions

```bash
# Check user permissions
pveum user permissions ansible@pve
```

You should see entries for:

- `/` (root path)
- `/storage`
- `/sdn`

### Step 2: Configuring your template

Once you've installed the OS.  SSH into the machine (or use the proxmox console GUI) to finish preparing the VM.

#### Install qemu-guest-agent

```bash
# Ensure the package is installed
apt-get install qemu-guest-agent

#Ensure it is running and configured to start on boot
systemctl start qemu-guest-agent
systemctl enable qemu-guest-agent
```

#### Clean your template

Before converting to a template, clean up the VM:

```bash
# Clean cloud-init state
cloud-init clean --logs --seed

# Remove machine-id (will be regenerated on first boot)
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id

# Remove SSH host keys (will be regenerated)
rm -f /etc/ssh/ssh_host_*

# Clean logs and temporary files
find /var/log -type f -exec truncate -s 0 {} \;
rm -rf /tmp/*
rm -rf /var/tmp/*

# Clear bash history
history -c
cat /dev/null > ~/.bash_history

# Shutdown the VM
shutdown -h now
```

#### Convert the VM to a Template

On your proxmox host:

```bash
# convert VM to template with your template's id
qm template <VMID>
```

You can also convert the VM to a template by right clicking on your template in the Rroxmox web GUI

### Step 3: Configuring Ansible Variables

This project manages ansible variables in the gorup_vars directory along with using ansible vault.

#### Clone this Repository

```bash
git clone https://github.com/Cbeniaris/Ansible-Proxmox-VM-Automation.git
cd Ansible-Proxmox-VM-Automation
```

#### Configure Variables

From the project directory, edit the vault_template.yml
This file is located in 
```
Ansible-Proxmox-VM-Automation/group_vars/all/vault_template.yml
```
You will need to edit this vault template file with the appropriate credentials

```yaml
---
vault_proxmox_password: "YOUR-PROXMOX-PASSWORD-HERE" # Password for your proxmox vm login
vault_vm_password: "VM-USER-PASSWORD-HERE" # password for user on created vm(s)
```

Once you've set your vault passwords, secure the file with ansible-vault

```bash
ansible-vault encrypt /Path/To/Your/vault_template.yml # if you've renamed the file from the template, use that name
```

#### Inventory File

Configure your inventory file to have your proxmox host

```bash
[new_vms]
#this will be populated as the playbook runs with the new vm information

[proxmox]
# Replace this with your proxmox host information
proxmox ansible_host=YOUR-PROXMOX-MACHINE-IP ansible_user=ansible #
```

### Step 4: Running the Playbook

To run the plabybook use:

```bash
#you will be asked for the password you used to secure your vault file with
ansible-playbook proxmox_vm_create.yml --ask-vault-pass
```

By default the variables for VM creation are:

```yaml
# Customize your variables variables
    vm_base_name: "rhel-clone" # Base name for VMs
    vm_count: 1 # Number of VMs to create
    vm_memory: 4096 # Memory in MB
    vm_cores: 2 # CPU cores
```

Optionally you can modify these at runtime by using:

```bash
ansible-playbook proxmox_vm_create.yml \
  -e "vm_base_name=test-machine" \
  -e "vm_count=5" \
  -e "vm_memory=16384" \
  -e "vm_cores=8" \
  -e "starting_ip_octet=50"
```

#### Post Deployment Tasks

post_vm_config.yml serves as a template to configure the machine after it has been created, started, and added to the inventory.  In this instance it is being used to configure firewalld and install JRE21.  You can either edit this task list to your needs, or remove the play from proxmox_vm_create.yml to keep them as a standard template clone.

## File Structure

```
Ansible-Proxmox-VM-Automation
├── ansible.cfg
├── group_vars
│   ├── all
│   │   ├── vault_template
│   │   └── vault.yml
│   └── new_vms.yml
├── inventory
├── inventory_template
├── LICENSE
├── proxmox_vm_create.yml
├── proxmox_vm_post_config.yml
└── README.md
```

## Security Best Practices

1. __Use Ansible Vault__ for sensitive data:

    This demo covers this but it warrants a reminder that your sensitive data should always be secured.

    ```bash
    ansible-vault encrypt /PATH/TO/YOUR/VAULT_FILE.yml
    ```

2. __Use API Tokens__ instead of passwords when possible

3. __Limit API user permissions__ to only what's needed:

    ```bash
    # Instead of Administrator, use specific roles
    pveum aclmod / -user ansible@pve -role PVEVMAdmin
    ```

4. __Use SSH keys__ instead of passwords for VM access

5. __Enable firewall__ on Proxmox and VMs

    This is taken care of if you leave the vm_post_config.yml as is.

6. __Keep templates updated__ with latest security patches