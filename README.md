# OS-Install

# Ubuntu Installation Instructions

## Overview

These instructions explain how to install the latest version of Ubuntu from a flash drive and manually partition the drive.

## Steps

### 1. Download Ubuntu

Download the ISO file for the latest version of Ubuntu from the website. Using a flashing tool to flash the ISO to the flash drive.

### 2. Shut Down the Computer

Shut down the computer completely.

### 3. Insert the Drive

Insert the hard drive or flash drive that contains the Ubuntu installer.

### 4. Boot From the Drive

Turn on the computer and press `F12` to open the boot menu.

Choose the file or drive that contains the Ubuntu installer.

When prompted, select:

```text
Try or Install Ubuntu
```

### 5. Check the Ubuntu Version

Once Ubuntu starts, open the terminal and check that you are using the correct version.

Run:

```bash
lsb_release -a
```

### 6. Continue the Installation

Continue through the Ubuntu installer until you reach the partitioning section.

Choose manual partitioning.

### 7. Remove Old Partitions

Click the minus button to remove older Ubuntu versions or any other partitions you no longer need from the drive.

Be careful not to delete your Windows partition unless you intend to erase Windows.

### 8. Create a New Partition

Click the plus button to create a new partition in the `nvme0n1` section.

Create a new Ubuntu partition in the unallocated space on the NVMe drive, such as `nvme0n1`. Make sure you are not deleting the Windows, EFI, or recovery partitions unless you intend to erase them. Leave enough space for Windows if you are dual-booting.

### 9. Finish Installation

Continue through the installer and complete the Ubuntu installation.

Restart the computer when prompted.

## Notes

* Make sure important files are backed up before deleting or resizing partitions.
* Double-check which partition belongs to Windows before making changes.
* The drive name may look similar to `nvme0n1`, but it can vary depending on your computer.


# Ubuntu Studio Audio Setup with Ansible

This repo documents the exact setup used to control an Ubuntu Studio machine from a Mac using Ansible and install audio software.

This setup uses:

```text
Mac = Ansible control machine
Ubuntu Studio = Ansible target machine
Target user = l42i
```

The installed software includes:

```text
REAPER
SuperCollider
IEM Plug-in Suite
```

## Important Note About the IP Address

The Ubuntu machine's IP address can change when it switches networks.

For example, during setup the target IP was first:

```text
10.11.12.245
```

After sharing internet from the Mac, the target IP changed to:

```text
192.168.2.2
```

Before running Ansible, always check the current Ubuntu IP address on the Ubuntu machine:

```bash
ip addr
```

For Wi-Fi, use:

```bash
ip addr show wlp129s0f0
```

Look for the line that starts with `inet`.

Example:

```text
inet 192.168.2.2/24
```

That means the IP address is:

```text
192.168.2.2
```

Update `inventory.ini` whenever this IP changes.

## Repo Files

The repo contains:

```text
inventory.ini
anisble.cfg
install_reaper.yaml
install_superCollider.yml
install_iem.yml
```

Note: the config file is currently named:

```text
anisble.cfg
```

It should be renamed to:

```text
ansible.cfg
```

Ansible automatically looks for `ansible.cfg`. If the name is misspelled, Ansible may ignore it.

## 1. Enable SSH on Ubuntu Studio

On the Ubuntu Studio machine, install SSH:

```bash
sudo apt update
sudo apt install openssh-server -y
```

Start SSH:

```bash
sudo systemctl start ssh
```

Enable SSH so it starts automatically:

```bash
sudo systemctl enable ssh
```

Check SSH status:

```bash
systemctl status ssh
```

## 2. Test SSH from the Mac

From the Mac, test SSH:

```bash
ssh l42i@CURRENT_IP_HERE
```

Example:

```bash
ssh l42i@192.168.2.2
```

If it asks for the password and logs in, SSH is working.

## 3. Install Ansible on the Mac

Install Ansible on the Mac. If using Homebrew:

```bash
brew install ansible
```

Check Ansible:

```bash
ansible --version
```

## 4. Inventory File

The inventory file tells Ansible which machine to control.

`inventory.ini`:

```ini
[studio]
192.168.2.2 ansible_user=l42i ansible_python_interpreter=/usr/bin/python3
```

If the Ubuntu IP changes, replace `192.168.2.2` with the current IP.

Example:

```ini
[studio]
10.11.12.245 ansible_user=l42i ansible_python_interpreter=/usr/bin/python3
```

## 5. Ansible Config File

The config file should be named:

```text
ansible.cfg
```

Current config contents:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = True
```

This sets the default inventory file and configures sudo/become.

## 6. Test Ansible SSH

From the Mac, run:

```bash
ansible -i inventory.ini studio -m ping -u l42i -k
```

`-k` means Ansible asks for the SSH password.

Expected result:

```text
pong
```

## 7. Test Ansible Sudo/Become

Run:

```bash
ansible -i inventory.ini studio -m command -a "whoami" -u l42i -k -b -K
```

Meaning:

```text
-k = ask for SSH password
-b = become root using sudo
-K = ask for sudo/become password
```

Expected output:

```text
root
```

This confirms Ansible can SSH into Ubuntu and run sudo commands.

## 8. Make `l42i` Passwordless for Sudo

We made the `l42i` user passwordless for sudo so Ansible can run sudo commands without repeatedly asking for the sudo password.

On Ubuntu, run:

```bash
sudo visudo
```

Add this line:

```text
l42i ALL=(ALL) NOPASSWD:ALL
```

Save and exit.

Now this command should work from the Mac:

```bash
ansible -i inventory.ini studio -m command -a "whoami" -b
```

Expected output:

```text
root
```

That means Ansible sudo/become is working.

## 9. Internet Issue During Setup

The Ubuntu machine was originally on a network that allowed local SSH access from the Mac, but did not provide internet.

That meant Ansible could connect, but `apt` could not download packages.

The error looked like this:

```text
Temporary failure resolving 'security.ubuntu.com'
```

Testing internet:

```bash
ping -c 4 8.8.8.8
```

Testing DNS:

```bash
ping -c 4 security.ubuntu.com
```

The fix was to share the Mac's wired internet connection with the Ubuntu machine.

After internet sharing worked, this succeeded:

```bash
ping -c 4 8.8.8.8
ping -c 4 security.ubuntu.com
```

Then `apt` and Ansible package installs worked.

## 10. Install REAPER

The REAPER tar file was located on the Mac at:

```text
/Users/adityarpawar/Desktop/Summer2026/l42i/reaper_install/reaper774_linux_x86_64.tar.xz
```

Playbook file:

```text
install_reaper.yaml
```

Run from the Mac:

```bash
ansible-playbook -i inventory.ini install_reaper.yaml -u l42i -k -K
```

If passwordless sudo is working, this may also work:

```bash
ansible-playbook -i inventory.ini install_reaper.yaml -u l42i -k
```

Verify REAPER:

```bash
ansible -i inventory.ini studio -m command -a "which reaper" -u l42i -k
```

Expected output:

```text
/usr/local/bin/reaper
```

## 11. Install SuperCollider

Playbook file:

```text
install_superCollider.yml
```

Run from the Mac:

```bash
ansible-playbook -i inventory.ini install_superCollider.yml -u l42i -k -K
```

The successful output showed:

```text
sclang installed at /usr/bin/sclang
scsynth installed at /usr/bin/scsynth
```

Verify on Ubuntu:

```bash
which sclang
which scsynth
which scide
```

Expected output:

```text
/usr/bin/sclang
/usr/bin/scsynth
/usr/bin/scide
```

Open the SuperCollider IDE on Ubuntu:

```bash
scide
```

## 12. Install IEM Plug-in Suite

First, we checked that the IEM packages existed:

```bash
apt search iem-plugin-suite
```

The packages found were:

```text
iem-plugin-suite-standalone
iem-plugin-suite-vst
iem-plugin-suite-vst3
```

Playbook file:

```text
install_iem.yml
```

Run from the Mac:

```bash
ansible-playbook -i inventory.ini install_iem.yml -u l42i -k -K
```

Verify on Ubuntu:

```bash
dpkg -l | grep -i iem-plugin-suite
```

Check plugin folders:

```bash
ls /usr/lib/vst
ls /usr/lib/vst3
```

The VST folder showed an IEM folder:

```text
iem.at
```

In REAPER, add or confirm these plugin paths:

```text
/usr/lib/vst
/usr/lib/vst3
```

Then rescan plugins in REAPER:

```text
Options > Preferences > Plug-ins > VST > Re-scan
```

## 13. Common Commands

Check Ubuntu IP:

```bash
ip addr show wlp129s0f0
```

Test SSH manually:

```bash
ssh l42i@CURRENT_IP_HERE
```

Test Ansible connection:

```bash
ansible -i inventory.ini studio -m ping -u l42i -k
```

Test Ansible sudo:

```bash
ansible -i inventory.ini studio -m command -a "whoami" -u l42i -k -b -K
```

Check REAPER:

```bash
which reaper
```

Check SuperCollider:

```bash
which sclang
which scsynth
which scide
```

Check IEM packages:

```bash
dpkg -l | grep -i iem-plugin-suite
```

Check internet:

```bash
ping -c 4 8.8.8.8
```

Check DNS:

```bash
ping -c 4 security.ubuntu.com
```

## Final State

At the end of this setup:

```text
Mac controls Ubuntu Studio with Ansible
SSH works
Ansible sudo/become works
l42i has passwordless sudo
REAPER is installed
SuperCollider is installed
IEM Plug-in Suite is installed
```
