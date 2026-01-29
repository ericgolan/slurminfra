VM Requirements :
2 CPU
8 GB Memory
2 Nic's 1 For internet Access and 1 for Closed DHCP Laboratory
50 GB disk for OS
200 GB disk for local repository
Make Sure Secure Boot is Disabled On ALL VM's
Not mandatory :
_Make sure KDUMP is disabled, Security Profile is turned off
Software Selection is Minimal Install
Make sure to select the current Disk 50GB for the OS
Asia\Jerusalem is selected
After the OS is Installed :
Disable firewall,kdump,Fix SELINUX,Mount CDROM,Hostname
Ready your Second Drive For Local Repository
Infra VM Complete !
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld
systemctl stop kdump.service
systemctl disable kdump.service
systemctl status kdump.service
vi /etc/selinux/config
--> SELINUX=disabled
SHELL
Set the IP for the internal network
NOTE: At this stage you should have all repositories download or have internet
connection for the reposync to download
Creating LocalRepository
lsblk
mkfs.ext4 /dev/sdb
# Find the UUID
blkid /dev/sdb
vi /etc/fstab
UUID=YOUR-UUID-HERE /var/www/html/ ext4 defaults 0 2
mount -a
# Make sure /var/www/html is mounted to sdb
lsblk
mkdir -p /var/www/html/repos
# Verify
df -h | grep /var/www/html
# note : the next folder will be used as local repo for the environment
mkdir /var/www/html/repos/rocky9
mkdir /mnt/iso
mount -o loop /dev/sr0 /mnt/iso
# copy the CD content - needed for PXE later on
# note : next folder will be used for PXE Boot etc...
mkdir /var/www/html/rocky9
cp -av /mnt/iso/. /var/www/html/rocky9
SHELL
# set hostname
hostnamectl set-hostname Slurm-Infra
SHELL
# Ensure the current interface is used
ip -br a
# Configuring IP address and features for the second adapter
nmcli connection modify ens224 ipv4.method manual ipv4.addresses 50.50.50.1/24
ipv4.gateway "" ipv4.dns "" ipv6.method disabled
nmcli con down ens224
nmcli con up ens224
SHELL
note : the repo file will be created at later stage via ansible
Create new Repository File
Disable Online Repos if needed or no internet access
Install lightweight Networking Services as well as PXE Environment
dnf install -y epel-release-9
dnf config-manager --enable crb
dnf reposync -a noarch -a x86_64 --download-metadata --downloadcomps -p
/var/www/html/repos/rocky9
# -a - architecure
dnf config-manager --disable appstream,baseos,extras
dnf repolist --enabled # to verify
SHELL
cat << EOF > /etc/yum.repos.d/offline-rocky.repo
[Offline-baseos]
name=Offline BaseOS
baseurl=file:///var/www/html/repos/rocky9/baseos
gpgcheck=0
enabled=1
[Offline-appstream]
name=Offline AppStream
baseurl=file:///var/www/html/repos/rocky9/appstream
gpgcheck=0
enabled=1
[Offline-epel]
name=Offline EPEL
baseurl=file:///var/www/html/repos/rocky9/epel
enabled=1
gpgcheck=0
EOF
SHELL
dnf install rsync vim bash-completion tar zip tmux wget open-vm-tools tree
ansible-core
dnf clean all
yum clean all
SHELL
# disable online repositories
dnf config-manager --disable baseos,appstream,extras
SHELL
Configuring dnsmasq.conf
note : Identify the nic to be used as a local network for the environment
Configure the conf file for the dnsmasq
dnf install -y dnsmasq syslinux httpd
mkdir -p /var/lib/tftpboot
# Move to our TFTP directory
cd /var/lib/tftpboot
# Download the UEFI bootloader
# Make sure internet Connection is working or manualy transfer the file to required
location
curl -O https://boot.ipxe.org/ipxe.efi -o /var/lib/tftpboot/ipxe.efi
# Ensure permissions are correct so dnsmasq can read them
chmod 644 /var/lib/tftpboot/*
SHELL
# Backup Default dnsmasq.conf to original
mv /etc/dnsmasq.conf /etc/dnsmasq.conf.org
SHELL
nmcli dev
SHELL
Create script for ipxe menu in /var/lib/tftpboot/boot.ipxe
vi /etc/dnsmasq.conf
# Interface Settings - make sure the current interface is selected
interface=ens224
bind-interfaces
dhcp-authoritative
# DHCP Range (Subnet 50.50.50.0/24) - starting from 50.50.50.10 up to 50.50.50.100
dhcp-range=50.50.50.10,50.50.50.100,255.255.255.0,12h
# option 3 - Router
dhcp-option=3,50.50.50.1
#option 6 - DNS Server
dhcp-option=6,50.50.50.1
#option 6 - DNS Server
#option 15 - Domain Name
#option 42 - NTP Server
# Logging for Troubleshooting
log-dhcp
log-queries
# TFTP Settings
enable-tftp
tftp-root=/var/lib/tftpboot
# Bootloader Logic
# Detect UEFI (Arch 7 or 9) vs Legacy BIOS
dhcp-match=set:efi64,option:client-arch,7
dhcp-match=set:efi64,option:client-arch,9
# Step 1: Deliver the iPXE firmware
dhcp-boot=tag:efi64,ipxe.efi
dhcp-boot=tag:!efi64,undionly.kpxe
# Step 2: Deliver the iPXE script (once iPXE is running)
dhcp-userclass=set:ipxe,iPXE
dhcp-boot=tag:ipxe,boot.ipxe
SHELL
# use the following command to view the leases ...:
cat /var/lib/dnsmasq/dnsmasq.leases
awk '{print "MAC: " $2 " ==> IP: " $3}' /var/lib/dnsmasq/dnsmasq.leases
SHELL
vim /var/lib/tftpboot/boot.ipxe
#!ipxe
# 1. Colors and Settings
set menu-timeout 10000
set submenu-timeout ${menu-timeout}
# 2. Define the Menu
:start
menu PXE Boot Menu - Rocky 9 Lab
item --gap -- ------------------------- OS Installers -----------------
--------
item rocky9 Install Rocky Linux 9
item --gap -- ------------------------- Tools -------------------------
item shell iPXE Shell
item exit Exit and Boot from Local Drive
choose --default exit --timeout ${menu-timeout} target && goto ${target}
# 3. Handle Choices
:rocky9
set server_ip 50.50.50.1
set base_url http://${server_ip}/rocky9
echo Loading Starting Fully Automated Install...
# We point kernel/initrd to the root images folder
kernel ${base_url}/images/pxeboot/vmlinuz inst.repo=${base_url}/BaseOS
inst.stage2=${base_url} inst.ks=${base_url}/ks.cfg ip=dhcp devfs=nomconf || goto
error
initrd ${base_url}/images/pxeboot/initrd.img || goto error
boot || goto error
:shell
shell
:exit
SHELL
Enable Apache
Generate ssh key before moving to next file
Setting Up KickStart File
exit
# 3. Start and Enable Apache
sudo systemctl enable --now httpd
SHELL
ssh-keygen -t rsa -b 4096 -f ~/.ssh/golan-infra
cat ~/.ssh/golan-infra.pub
# You will need this in the following file
SHELL
vim /var/www/html/rocky9/ks.cfg
# ==============================================================================
# Rocky Linux 9 Kickstart - Infra Node (Generated by Ansible)
# ==============================================================================
# Ansible managed
# ------------------------------------------------------------------
# Installation mode & source
# ------------------------------------------------------------------
text
reboot
# IMPORTANT:
# This URL must point to the ROOT containing .treeinfo
url --url="http://50.50.50.1/rocky9"
lang en_US.UTF-8
keyboard us
timezone Asia/Jerusalem --utc
# ------------------------------------------------------------------
# Network (PXE → install → persistent)
# ------------------------------------------------------------------
network --bootproto=dhcp --device=link --activate --onboot=on
# ------------------------------------------------------------------
# Security
# ------------------------------------------------------------------
rootpw --plaintext P@ssw0rd
# Choose ONE policy (infra default = secure)
# selinux --enforcing
# firewall --enabled --ssh
# --- OR ---
selinux --disabled
firewall --disabled
# ------------------------------------------------------------------
# Disk handling (HARD RESET)
# ------------------------------------------------------------------
ignoredisk --only-use=sda
zerombr
clearpart --all --initlabel --drives=sda --disklabel=gpt
# ------------------------------------------------------------------
# EFI + /boot
# ------------------------------------------------------------------
part /boot/efi --fstype=efi --size=600 --ondisk=sda
part /boot --fstype=xfs --size=1024 --ondisk=sda
# ------------------------------------------------------------------
# LVM layout
SHELL
# ------------------------------------------------------------------
part pv.01 --fstype=lvmpv --ondisk=sda --size=1 --grow
volgroup vg_system pv.01
logvol swap --vgname=vg_system --name=swap --fstype=swap --size=4096
logvol / --vgname=vg_system --name=root --fstype=xfs --size=1 --grow
# ------------------------------------------------------------------
# Package set (minimal + Ansible support)
# ------------------------------------------------------------------
%packages
@^minimal-environment
python3
python3-dnf
curl
wget
vim
tmux
bash-completion
rsync
lsof
which
%end
# ------------------------------------------------------------------
# PRE: wipe residual metadata (CRITICAL)
# ------------------------------------------------------------------
%pre --log=/tmp/ks-pre.log
wipefs -a /dev/sda || true
%end
# ------------------------------------------------------------------
# POST: Infra bootstrap
# ------------------------------------------------------------------
%post --log=/root/ks-post.log
# --------------------------------------------------
# Create Ansible management user (NO auto home)
# --------------------------------------------------
useradd -M -d /var/lib/ansible -s /bin/bash ansible
echo "ansible:P@ssw0rd" | chpasswd
echo "ansible ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/ansible
chmod 0440 /etc/sudoers.d/ansible
# --------------------------------------------------
# Create and secure home manually
# --------------------------------------------------
mkdir -p /var/lib/ansible/.ssh
chown -R ansible:ansible /var/lib/ansible
chmod 0700 /var/lib/ansible
chmod 0700 /var/lib/ansible/.ssh
Finaly Ansible
# --------------------------------------------------
# Inject SSH key
# --------------------------------------------------
cat << 'EOF' > /var/lib/ansible/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...YOUR_KEY_HERE...
EOF
chmod 0600 /var/lib/ansible/.ssh/authorized_keys
chown ansible:ansible /var/lib/ansible/.ssh/authorized_keys
# --------------------------------------------------
# SSH hardening
# --------------------------------------------------
sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/'
/etc/ssh/sshd_config
sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/'
/etc/ssh/sshd_config
systemctl enable sshd
# --------------------------------------------------
# Branding
# --------------------------------------------------
echo "Provisioned via iPXE + Infra Kickstart" > /etc/motd
%end
chown apache:apache /var/www/html/rocky9/ks.cfg
# Test Apache and file destination
curl -I http://50.50.50.1/rocky9/ks.cfg
SHELL
# Already installed in previous steps
# dnf install -y ansible-core
SHELL
