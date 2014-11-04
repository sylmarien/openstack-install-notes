# Compute node

## Overview

This file contains notes on the Controller node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Controller node.

## Basic environment

The network configuration for this node is the following:  
3 interfaces :

- eth0 : NAT
- eth1 : Management interface (subnet 10.10.10.0/24)
- eth2 : Instance tunnels interface (subnet 10.20.20.0/24)

**Configure networking**

1. Modify /etc//network/interfaces to configure:
  1. eth1 :
  
    ```
    # Management network interface
    auto eth1
    iface eth1 inet static
    address 10.10.10.31
    netmask 255.255.255.0
    # On VM, no gateway because it is set by the DHCP on the NAT interface
    # gateway 10.10.10.1
    ```
  2. eth2 :
  
    ```
    # Instance tunnels interface
    auto eth2
    iface eth2 inet static
    address 10.20.20.31
    netmask 255.255.255.0
    ```
2. Modify /etc/hosts to configure the name resolution:

  ```
  # controller
  10.10.10.11       controller

  # network
  10.10.10.21       network

  # compute1
  10.10.10.31       compute1
  ```  
  Comment any other line.
3. Reboot to activate changes.

**Configure NTP**

1. Install the NTP service:  
  `apt-get install ntp`
2. Configure the NTP service:
  1. Modify /etc/ntp.conf to comment or remove all server key but one to reference the controller node:  
    `server controller iburst`
  2. Remove the /var/lib/ntp/ntp.conf.dhcp file if it exists.
3. Restart the NTP service:  
  `service ntp restart`

**Basic prerequisites**

1. Install _python-software-properties_ package to ease repository management:  
  `apt-get install python-software-properties`
2. Enable the OpenStack repository (seems not necessary on Ubuntu 14.04 since Icehouse is the default version):  
  `add-apt-repository cloud-archive:icehouse`
3. Upgrade the packages on the system:  
  `apt-get update && apt-get dist-upgrade`
