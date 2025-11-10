# Ansible-Proxmox-VM-Automation

This serves as a demonstration for the creation of pre configured Red Hat Enterprise Linux 9.6 virtual machines in a proxmox virtual environment.  The playbooks included will create the vms according to specified variables and do post configuration of the Java Runtime Environment.  

This document will cover:
- Setting up an api user in Proxmox for remote connection
- Configuring a template VM for conversion to a template
- Setting up variables and vault files required for the Ansible playbook to run

## Prequisites

### Software

- Proxmox VE: Version 8.0 or higher  (Tested on Proxmox 8.4)
- Ansible: Version 2.9 or higher (Tested on 2.19.4) 
- Python: Version 3.6 or higher (Tested on 3.13.9)
- Proxmoxer installed on host

### Ansible Collections

```bash
ansible-galaxy collection install community.proxmox
```

### Clone Template

You must have __qemu-guest-agent__ installed on your template VM

## Setup Instructions

### Step 1: Create Proxmox API User

For this to work, create a dedicated API user for Ansible.

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

### Step 2: Create your Template

#### Create a virtual machine
This will be the one and (hopefully) only time this process will require you to create a VM and install the OS.  Create your VM, specify your hardware parameters, boot the VM, and follow the instructions to install the OS.  The user you create during set up will be the user Ansilbe uses to log in and configure post clone configuations.  Once your VM has been created SSH into the machine (or use the Proxmox console GUI) to connect and finish configuring the template.

#### Install qemu-guest-agent

```bash
# Ensure the package is installed
dnf install qemu-guest-agent

#Ensure it is running and configured to start on boot
systemctl start qemu-guest-agent
systemctl enable qemu-guest-agent
```

#### Clean your template

Before converting to a template, clean up the VM:

```bash
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

This must be done to ensure that each created VM from this template will not share the same machine-id.  This will also ensure that ssh ckeys are not cloned across all vms for security purposes.


#### Convert the VM to a Template

On your proxmox host:

```bash
# convert VM to template with your template's id
qm template <VMID>
```

You can also convert the VM to a template by right clicking on your template in the Proxmox web GUI

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
# if you've renamed your vault_template use that name
ansible-vault encrypt /Path/To/Your/vault_template.yml 
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

Optionally you can modify any number of these at runtime by using the '-e' flag:

```bash
#example of a single single variable
ansible-playbook proxmox_vm_create.yml -e "vm_count=3" --ask-vault-pass
```

```bash
#example of multieple variables
ansible-playbook proxmox_vm_create.yml \
  -e "vm_base_name=test-machine" \
  -e "vm_count=5" \
  -e "vm_memory=16384" \
  -e "vm_cores=8" \
  -e "starting_ip_octet=50"
```

Add or remove desired variables for target VM config

#### Post Deployment Tasks


 proxmox_vm_post_config.yml serves as a template to configure the machine after it has been created, started, and added to the inventory.  Currently the task list is imported in a second play at the end of proxmox_vm_create.yml. In this applicaton, it is being used to configure firewalld and install JRE21.  Any number of post configuration tasks can be aded for your desired result of services and packages. You can either edit this task list to your needs, or remove the play from proxmox_vm_create.yml entirely to keep them as a standard template clone.

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

## License

This project is licensed under the MIT License - see the LICENSE file for details.