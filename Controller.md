# Controller node

## Overview
This file contains notes on the Controller node of the architecture.

**During development, all passwords are _localadmin_.**

## Basic environment

The system has been set up on an a disk managed through LVM with the following configuration:  
- 1 VG containing all the storage available (all the PVs) named vg0
- a _root_ LV of 100Go that contains the OS
- a _horizon_ LV of 10Go that will contain the Horizon service of OpenStack.
- a _core_ LV of 10Go that will contain the Nova, Neutron, Keystone services of Openstack as well as the RabbitMQ service.
- a _store_ LV of 50Go that will contain the Glance, Cinder and Swift services of OpenStack.
- a _Database_ LV of 50Go that will contain the SQL database (MariaDB).

The network configuration for this node is the following:  

- em1 : physical interface connect to the network
- br0: virtual interface (bridge) used to connect all the VMs to em1

**Configure networking**

1. Modify /etc/network/interfaces to configure eth1:

  ```
# The loopback network interface
auto lo
iface lo inet loopback

# Set up interfaces manually, avoiding conflicts with, e.g., network manager
auto em1
iface em1 inet manual

# Bridge setup
auto br0
iface br0 inet static
  bridge_ports em1
  address 10.79.6.7
  netmask 255.255.0.0
  broadcast 10.79.255.255
  gateway 10.79.0.1
  dns-nameservers 10.28.0.4 10.28.0.5
  dns-search uni.lux
  ```
2. Modify /etc/hosts to configure the name resolution:

  ```
127.0.0.1       localhost
127.0.1.1       openstack-ctrl1

# Horizon
10.79.7.1       horizon

# Core
10.79.7.2       core

# Store
10.79.7.3       store

# Database
10.79.7.4       database
  ```
3. Reboot to activate changes.

## Virtual Machines configuration

1. Requisite packages:  
`apt-get install qemu-kvm libvirt-bin virtinst`
2. Download most recent ubuntu server image (14.04.1 at this time). For the next commands I'll name it ubuntu-14.04.1-server-amd64.iso
3. Create logical volumes as stated in the basic environment section:

```
lvcreate -n horizon -L 10G vg0
lvcreate -n core -L 10G vg0
lvcreate -n store -L 50G vg0
lvcreate -n database -L 50G vg0
```
4. Create the VMs using the ubuntu server image and the virt-install command:

```
virt-install --name=horizon --description="VM hosting the OpenStack Horizon service." --ram=4096 --vcpus=3 --os-variant=ubuntutrusty --location /home/localadmin/ubuntu-14.04.1-server-amd64.iso --extra-args='console=tty0 console=ttyS0,115200n8 serial' --disk path=/dev/vg0/horizon --hvm --network bridge=br0 --autostart --nographics
virt-install --name=core --description="VM hosting the OpenStack core services: Nova, Neutron, Keystone, RabbitMQ." --ram=4096 --vcpus=3 --os-variant=ubuntutrusty --location /home/localadmin/ubuntu-14.04.1-server-amd64.iso --extra-args='console=tty0 console=ttyS0,115200n8 serial' --disk path=/dev/vg0/core --hvm --network bridge=br0 --autostart --nographics
virt-install --name=store --description="VM hosting the OpenStack storage services: Glance, Cinder, Swift." --ram=4096 --vcpus=3 --os-variant=ubuntutrusty --location /home/localadmin/ubuntu-14.04.1-server-amd64.iso --extra-args='console=tty0 console=ttyS0,115200n8 serial' --disk path=/dev/vg0/store --hvm --network bridge=br0 --autostart --nographics
virt-install --name=database --description="VM hosting the OpenStack database." --ram=4096 --vcpus=3 --os-variant=ubuntutrusty --location /home/localadmin/ubuntu-14.04.1-server-amd64.iso --extra-args='console=tty0 console=ttyS0,115200n8 serial' --disk path=/dev/vg0/database --hvm --network bridge=br0 --autostart --nographics
```
5. For each of them follow the installation steps and configure them accordingly to the wanted configuration.  
In our case, here are the majors informations (temporary):

```
DNS:
10.28.0.4
10.28.0.5

Gateway: 10.79.0.1

openstack-ctrl1:
Horizon : 10.79.7.1 (hostname: horizon, no domain name). user: localadmin mdp: localadmin
Core: 10.79.7.2 (hostname: core, no domain name). user: localadmin mdp: localadmin
Store: 10.79.7.3 (hostname: store, no domain name). user: localadmin mdp: localadmin
Database: 10.79.7.4 (hostname: database, no domain name). user: localadmin mdp: localadmin
```
6. Once the VM are running, before connecting to them, make sure to make them launch automatically when the server boots:

```
virsh autostart horizon
virsh autostart core
virsh autostart store
virsh autostart database
```
7. Once that is done, connect to the various VMs to perform the following manipulation (example is shown for horizon, just apply the same command on all the VMs):
  1. Connect to the VM:
  `virsh console horizon`
  2. Modify `/etc/hosts` with the following changes:

  ```
  # Horizon
  10.79.7.1       horizon

  # Core
  10.79.7.2       core

  # Store
  10.79.7.3       store

  # Database
  10.79.7.4       database
  ```
  3. Apply updates and install the ntp package:  
  `apt-get update && apt-get dist-upgrade && apt-get install ntp`
8. The common configuration is now finished, follow the VM-specific configurations in the corresponding files.
