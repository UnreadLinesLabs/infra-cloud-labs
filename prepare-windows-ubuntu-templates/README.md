# Prepare Windows and Ubuntu Virtual Machines as Templates

🎥 Video: [Add your YouTube link]  
🏷️ Theme: CLOUD & INFRA

📅 Last verified: 2026-07-14  
📝 Update log: Initial draft for template-based VM preparation

## Overview

This lab shows how to prepare a base Windows or Ubuntu virtual machine so it can be converted into a reusable template or golden image. The goal is to remove machine-specific data, shut the VM down safely, and make it ready for cloning or deployment on a hypervisor such as Proxmox VE, VMware vSphere, KVM, Hyper-V, or VirtualBox.

This guide is intended for administrators, lab builders, and engineers who need clean, repeatable virtual machine images for testing, demonstrations, and production-like environments.

## Prerequisites

- A virtual machine already installed with:
  - Windows 10/11 or Windows Server 2019/2022/2025
  - Ubuntu Server 26.04 LTS
  - Debian latest stable release
- Administrative access to the guest operating system
- A hypervisor or platform that supports cloning or template conversion
- Optional but recommended: cloud-init support on the target platform
- A snapshot or backup before starting the final preparation steps

## Architecture

You prepare one base VM per operating system, remove machine-specific identity and configuration data, then shut it down and convert it into a template. New clones created from that template can be customized at first boot with their own hostname, user account, SSH keys, and networking configuration.

## Steps

### 1. Prepare the Windows VM

Before running Sysprep, make sure the machine is in a good state:

1. Install Windows updates.
2. Install required applications and tools.
3. Install the hypervisor guest tools if applicable.
4. Remove temporary files and unnecessary software.
5. Restart Windows and confirm the system behaves normally.
6. Do not join the template to a production domain.

Run Sysprep as Administrator:

```powershell
C:\Windows\System32\Sysprep\Sysprep.exe /generalize /oobe /shutdown
```

The options do the following:

- `/generalize` removes machine-specific information from the installation.
- `/oobe` starts the Windows Out-of-Box Experience on the next boot.
- `/shutdown` powers off the VM when preparation is complete.

Do not boot the original VM again after this step. Convert or clone it as a template instead.

### 2. First boot of a Windows clone

When a clone boots for the first time, Windows will start OOBE and may ask for:

- Region and keyboard layout
- Computer name
- User account
- Password
- Privacy and network settings

The exact screens depend on the Windows edition and deployment configuration.

### 3. Prepare the Ubuntu VM

Ubuntu does not use Sysprep. The usual approach is to update the OS, clean machine-specific state, and rely on cloud-init for customization.

Update the system:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove --purge -y
sudo apt autoclean
sudo apt clean
```

If a new kernel or major component was installed, reboot:

```bash
sudo reboot
```

After reboot, confirm that no further upgrade is pending:

```bash
sudo apt update
sudo apt full-upgrade
```

### 4. Verify and install cloud-init

Check whether cloud-init is installed:

```bash
dpkg -l | grep cloud-init
```

Install it if necessary:

```bash
sudo apt update
sudo apt install cloud-init -y
```

> [!NOTE]
> Installing cloud-init alone does not create an interactive wizard. The hypervisor or deployment platform must provide metadata and user data to configure the hostname, user, SSH keys, password configuration, and networking.

### 5. Clean the Ubuntu template

Run the following commands only when the VM is ready to become a template:

```bash
sudo apt autoremove --purge -y
sudo apt clean

sudo cloud-init clean --logs

sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id

sudo rm -f /etc/ssh/ssh_host_*

sudo journalctl --rotate
sudo journalctl --vacuum-time=1s

sudo rm -rf /tmp/* /var/tmp/*

sudo sync
sudo shutdown -h now
```

After shutdown, convert the VM into a template without starting it again.

### 6. First boot of an Ubuntu clone

When the platform provides valid cloud-init data, the first boot can:

- Set a unique hostname
- Create or configure a user
- Enable or disable password authentication
- Add SSH public keys
- Configure networking
- Regenerate SSH host keys
- Run initial setup commands

Without a supported datasource, the clone will typically keep the existing hostname and user configuration and will not behave like a Sysprep-driven Windows deployment.

### Manual hostname and IP configuration for a classic lab

If the platform does not support cloud-init or if you are deploying in a simple lab environment, configure the clone manually after the first boot.

#### Ubuntu: change the hostname

Set a temporary hostname:

```bash
sudo hostnamectl set-hostname SRV-FILE01
```

Make it persistent by editing `/etc/hostname` and `/etc/hosts`:

```bash
sudo nano /etc/hostname
sudo nano /etc/hosts
```

Replace the existing hostname with your new one, then reboot:

```bash
sudo reboot
```

#### Ubuntu: configure the IP address

On Ubuntu, the Netplan configuration file is often created during installation. If you configured a fixed IP during setup, the file is commonly `/etc/netplan/00-installer-config.yaml`. If you left the network in DHCP, there may be no Netplan file at all, and you can create one manually.

Check the existing files:

```bash
ls /etc/netplan/
```

If there is no file, create one manually. First identify the network interface name:

```bash
ip a
```

Then create a Netplan file such as `/etc/netplan/00-installer-config.yaml`:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

For DHCP, use a simple Netplan config:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: true
```

For a fixed IP, use something like:

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Apply the configuration:

```bash
sudo netplan generate
sudo netplan apply
```

#### Windows: change the hostname and IP

For Windows, you can change the hostname with PowerShell:

```powershell
Rename-Computer -NewName SRV-FILE01 -Restart
```

For a static IP:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.100 -PrefixLength 24 -DefaultGateway 192.168.1.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 1.1.1.1, 8.8.8.8
```

For DHCP, simply enable DHCP on the adapter instead of assigning a static address.

### Example cloud-init configuration

The following example creates an administrative user and sets the hostname. Replace the sample SSH key before use.

```yaml
#cloud-config
hostname: ubuntu-server
manage_etc_hosts: true

users:
  - default
  - name: admin
    groups:
      - sudo
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-ed25519 REPLACE_WITH_YOUR_PUBLIC_KEY

ssh_pwauth: false
disable_root: true

package_update: true
```

## Platform notes

The method used to provide cloud-init data depends on the virtualization platform:

- Proxmox VE: add a Cloud-Init drive and configure the hostname, user, password, or SSH key in the VM Cloud-Init settings.
- VMware vSphere: use a supported guest customization or cloud-init workflow.
- OpenStack and public clouds: instance metadata normally provides cloud-init configuration automatically.
- KVM/QEMU: attach a NoCloud seed image containing user-data and meta-data.
- Hyper-V or VirtualBox: cloud-init requires a compatible datasource or an attached NoCloud seed image.

## Common Pitfalls

- Booting the original VM again after Sysprep has been run.
- Forgetting to clear machine-specific data on Ubuntu.
- Assuming cloud-init will work without platform metadata.
- Reusing a password or SSH key that should be unique per clone.
- Leaving the template joined to a production domain.
- Using the wrong interface name in Netplan examples.

## References

- [Microsoft: Sysprep command-line options](https://learn.microsoft.com/windows-hardware/manufacture/desktop/sysprep-command-line-options)
- [Microsoft: Sysprep process overview](https://learn.microsoft.com/windows-hardware/manufacture/desktop/sysprep-process-overview)
- [cloud-init documentation](https://cloudinit.readthedocs.io/)
- [Ubuntu documentation](https://documentation.ubuntu.com/)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
