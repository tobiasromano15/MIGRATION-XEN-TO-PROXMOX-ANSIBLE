# MIGRATION-FROM-GANETI-TO-PROXMOX
THIS IS A QUICK STEP TO STEP VM MIGRATION FROM GANETI WITH XEN HYPERVISOR TO PROXMOX KVM.
IT IS MADE TO WORK FOR EVERY DEBIAN & UBUNTU VERSION.

THIS REPOSITORY ALSO HAS AN ANSIBLE SCRIPT TO PREPARE A VM FOR MIGRATION, THE ONLY EXCEPTION THAT IT DOES NOT INSTALL A NEW LINUX-IMAGE, WHICH IS A FUNDAMENTAL STEP TO MAKE IT WORK.

Ganeti to Proxmox Migration Steps
Steps to migrate a virtual machine from the Ganeti cluster to the Proxmox cluster.

Virtual Machine Preparation
1- First, update the packages:

apt update
apt upgrade

2- Virtual machines in Ganeti do not have the boot master record installed, as boot devices in Ganeti are generally /dev/xvda. Therefore, the following steps must be taken:

UBUNTU

grub-install /dev/xvda

DEBIAN

install grub-pc
reboot (to verify it doesn’t break in Ganeti)
upgrade-from-grub-legacy
reboot
grub-install /dev/xvda

3- A kernel needs to be installed on the virtual machine

In some cases, use apt to avoid having to remove packages.

For UBUNTU 22

aptitude install linux-image-5.19.0-50-generic
aptitude install linux-image-6.5.0-41-generic

For UBUNTU 20

aptitude install linux-image-5.15.0-97-generic

For UBUNTU 18

aptitude install linux-image-5.4.0-150-generic

For UBUNTU 16

aptitude install linux-image-4.15.0-142-generic

For UBUNTU 14

We must force the installation of linux-image-4.15.0-142-generic. This is necessary to activate the VirtIO drivers. Therefore, the following steps must be taken:

Edit the file /etc/apt/sources.list and modify it as follows:

#deb http://debian.unicen.edu.ar:9999/ubuntu trusty main restricted universe multiverse
#deb http://debian.unicen.edu.ar:9999/ubuntu trusty-updates main restricted universe multiverse
#deb http://debian.unicen.edu.ar:9999/ubuntu trusty-security main restricted universe multiverse
deb http://debian.unicen.edu.ar:9999/ubuntu xenial main restricted universe multiverse
deb http://debian.unicen.edu.ar:9999/ubuntu xenial-updates main restricted universe multiverse
Run:

apt update
aptitude install linux-image-4.15.0-142-generic
Restore /etc/apt/sources.list to its original state.

Run:

apt update
init 6

For DEBIAN 8

Point to stretch:

apt install linux-image-4.9.0-13-amd64

Point back to jessie.

4- The virtual machine needs the VirtIO modules to understand the drivers provided by Proxmox.

First, verify that the modules exist:


modprobe virtio_blk
modprobe virtio_net
lsmod | grep vir
If the modules were loaded by the previous command, make them persistent:

echo "virtio_blk" | tee -a /etc/modules
echo "virtio_net" | tee -a /etc/modules

5- Install the QEMU guest agent to better manage the virtual machine when it is on Proxmox

aptitude install qemu-guest-agent

6- Virtual machines mount the device by name in fstab, so it needs to be changed to use the UUID.

First, identify the UUID:

blkid
Once you have the UUID (generally for the /dev/xvda1 device), modify the entry in /etc/fstab:

/dev/xvda1 / ext3 noatime,nodiratime,errors=remount-ro 0 1

It should be changed to:

UUID=XXXXXXX-XXXXX (YOUR UUID) / ext3 noatime,nodiratime,errors=remount-ro 0 1

7- The network device in Proxmox changes its name, from eth0 to ens18

CAUTION: In Ubuntu 14, it remains eth0

Some virtual machines are configured with networking, and others with netplan. You need to edit /etc/network/interfaces or /etc/netplan/00_unicen.yaml.

Change eth0 to ens18 (NOT IN UBUNTU)

Virtual Machine Export
1- Identify the logical volume of the virtual machine to be migrated

On the Ganeti master node:

gnt-instance info <virtual>
You will find something like this:

child devices: 
  - child 0: plain, size 20.0G
    logical_id: xenvg/84feb67c-21dc-4a46-a41a-134e0e61075e.disk0_data
    on primary: /dev/xenvg/84feb67c-21dc-4a46-a41a-134e0e61075e.disk0_data (253:27)
    on secondary: /dev/xenvg/84feb67c-21dc-4a46-a41a-134e0e61075e.disk0_data (253:14)
    
2- Perform the actual export

On the primary node:

qemu-img convert -p -f raw -O qcow2 /dev/xenvg/087078e0-7cf7-4e1a-ae0c-58d70a87bc9b.disk0_data /tmp/<virtual>.qcow2

1- Create the virtual machine without a disk from the Proxmox web console.

2- Copy the newly created image to a Proxmox node

On a Proxmox node:

scp soporte@nodoXX.unicen.edu.ar:/<tmp o data>/<virtual>.qcow2 .

3- Import the disk, associating it with the ID of the created virtual machine (e.g., 107):

qm importdisk vmid /tmp/<virtual>.qcow2 local-lvm --format qcow2

In the Proxmox console:

Add the disk: In the hardware section of the associated virtual machine, you will see the disk as "Unused Disk 0". Select it and click "Add disk".

Boot Order: Finally, in the Options section, under Boot Order, deselect other options and leave only the imported disk.

IMPORTANT
Virtual machines in Proxmox using Ubuntu 14.04 may get stuck on boot with a screen displaying Booting from Hard Disk. This does not mean the machine has not booted; it works correctly when accessed via SSH.
