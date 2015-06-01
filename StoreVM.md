# Store VM

## Overview
This file contains notes on the Store VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Store VM.

## Basic configuration

Basic configuration is described in the [Controller.md](https://github.com/sylmarien/openstack-install-notes/blob/master/Controller.md) file. It is the shared configuration between all the VMs of the controller node.

**Network Configuration**

Modify `/etc/network/interfaces` to match the following content:

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
    #post-up ifconfig eth1 down
    #post-up ifconfig eth1 up
    address 10.89.201.3
    netmask 255.255.0.0
    network 10.89.0.0
    broadcast 10.89.255.255
    gateway 10.89.0.1
    # dns-* options are implemented by the resolvconf package, if installed
    dns-nameservers 10.28.0.4 10.28.0.5
```

**Basic prerequisites**

1. Install _python-software-properties_ package to ease repository management:  
  `apt-get install python-software-properties`
2. Enable the OpenStack repository (not necessary on Ubuntu 14.04 since Icehouse is the default version):  
  `add-apt-repository cloud-archive:icehouse`