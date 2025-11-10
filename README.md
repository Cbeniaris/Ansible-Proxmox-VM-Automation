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

Navigate from the project directory to ./group_vars/all/
You will need to edit this vault template file with the appropriate credentials

```yaml
---
vault_proxmox_password: "YOUR-PROXMOX-PASSWORD-HERE" # Password for you proxmox vm login
vault_vm_password: "VM-USER-PASSWORD-HERE" # password for user on created vm(s)
```

Ensure your edit these variables according to your proxmox ansible user's login, and your configured user in your vm template.

#### Inventory File
