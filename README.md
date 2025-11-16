# Ansible-Proxmox-VM-Automation

This serves as a demonstration for the creation of pre configured Red Hat Enterprise Linux 10.1 virtual machines in a proxmox virtual environment.  The playbooks included will create the vms according to specified variables and do post configuration of the Java Runtime Environment.  

This document will cover:
- Setting up an api user in Proxmox for remote connection
- Configuring a template VM for conversion to a cloning template
- Setting up variables and vault files required for the Ansible playbook to run

## Prequisites

### Software/Hardware

#### Hardware

A machine running - Proxmox VE: Version 8.0 or higher  (Tested on Proxmox 8.4)

#### Software

Required on Host:
- Ansible: Version 2.9 or higher (Tested on 2.19.4) 
- Python: Version 3.6 or higher (Tested on 3.13.9)
- Proxmoxer installed on host
- Ansible Proxmox Collection

To install the community collection for Proxmox you use this command on your host machine:
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
```bash
- / (root path)  # root file system
- /storage  # access for VM disks, ISOs, container templates, etc.
- /sdn  # Software Defined Networking. Virtual networks, zones, subnets, etc.
```

Why we're doing this:
The API allows for a consistent interface with predefined endoints, convenient error handling, and explicit permissions.  API calls are also logged separately, making tracking automation vs. manual changes easier.  For one off operations, ssh is great.  For larger automation tasks, the API is the way to go.

### Step 2: Create your Template

#### Create a virtual machine
This will be the one and (hopefully) only time this process will require you to create a VM and install the OS.  Create your VM on your Proxmox machine, specify your hardware parameters, boot the VM, and follow the instructions to install the OS.  The user you create during set up will be the user Ansilbe uses to log in and configure post clone configuations.  Once your VM has been created SSH into the machine (or use the Proxmox console GUI) to connect and finish configuring the template.

#### Install qemu-guest-agent

```bash
# Ensure the package is installed
dnf install qemu-guest-agent

#Ensure it is running and configured to start on boot
systemctl start qemu-guest-agent
systemctl enable qemu-guest-agent
```

Qemu-guest-agent a helper daemon that allows the hypervisor to communicate more effectively with the guest VM.  IT allowes for proper shutdown ofthe guest with ACPI calls, freezing the guest file system when making backups, and time syncronization on boot.

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

This must be done to ensure that each created VM from this template will not share the same machine-id.  Should the be skipped, it would cause conflicts with networking, bootloader functionality, and application and system stability. This will also ensure that ssh keys are not cloned across all vms for security purposes.


#### Convert the VM to a Template

Back on your Proxmox machine you're ready to convert the VM to a template eligible for cloning:

```bash
# convert VM to template with your template's id
qm template <VMID>
```

You can also convert the VM to a template by right clicking on your template in the Proxmox web GUI

### Step 3: Configuring Ansible Variables

#### Clone this Repository

```bash
git clone https://github.com/Cbeniaris/Ansible-Proxmox-VM-Automation.git
cd Ansible-Proxmox-VM-Automation
```

#### Configure Variables

This project manages ansible variables in the gorup_vars directory along with using ansible vault.

```
Ansible-Proxmox-VM-Automation
├── ansible.cfg
├── group_vars
│   ├── all
│   │   └── vault_template
│   └── new_vms.yml
├── inventory
├── LICENSE
├── proxmox_vm_create.yml
├── proxmox_vm_post_config.yml
└── README.md
```

From the project directory, edit the vault_template.yml
This file is located in 

```
Ansible-Proxmox-VM-Automation/group_vars/all/vault_template.yml
```
You will need to edit this vault template file with the appropriate credentials.  

```yaml
---
vault_proxmox_password: "YOUR-PROXMOX-PASSWORD-HERE" # Password for your proxmox vm login
vault_vm_password: "VM-USER-PASSWORD-HERE" # password for user on created vm(s)
```

When you are finished rename this file to "vault.yml" and secure the file with ansible-vault.  You will be asked for a password to secure the vault file with.  This password will be required when you run the playbook.

```bash
ansible-vault encrypt /Path/To/Your/vault.yml 
```

### Step 4: Running the Playbook

At this point you are ready to take advantage of the benefits of ansible.  You've configured your Proxmox machine, created a VM template that will be used to clone, and secured your vault passwords with ansible-vault.  Now it's time to set the variables for the playbook and run it.

The variables for VM creation are in the beginning of the first play of proxmox_vm_create.yml  Feel free to edit these for your application.

```yaml
# Customize your variables variables
    vm_base_name: "rhel-clone" # Base name for VMs
    vm_count: 1 # Number of VMs to create
    vm_memory: 2048 # Memory in MB.  This is the minimum for RHEL10.1  (4096 MB recommended)
    vm_cores: 2 # CPU cores.  This is the minimum for RHEL10.1 
```
To run the play book, use:

```bash
#you will be asked for the password you used to secure your vault file with
ansible-playbook proxmox_vm_create.yml --ask-vault-pass
```

#### Post Deployment Tasks

 proxmox_vm_post_config.yml serves as a template to configure the machine after it has been created, started, and added to the inventory.  Currently the task list is imported in a second play at the end of proxmox_vm_create.yml. In this applicaton, it is being used to configure firewalld and install JRE21.  Any number of post configuration tasks can be aded for your desired result of services and packages. You can either edit this task list to your needs, or remove the play from proxmox_vm_create.yml entirely to keep them as a standard template clone.


### Why Ansible?

From a distance, it's easy to look at a project like this and think, "Ok, so you cloned a few VMs.  What's the big deal?"  Yes, it is more than possible to clone each VM one by one and install packages to each instance.  However, that line of thinking assumes that ansible is no longer useful after initial setup.  By having each VM added to your ansible inventory, you can go on to write play books that target some or all of these for even more automation.  A possible exercise for this would be to write a play book that backs up each VM you created to a shared storage and have ansible run that play book weekly.  Possibilities are only limited by your needs and access to an end point for ansible to connect to.

## License

This project is licensed under the MIT License - see the LICENSE file for details.