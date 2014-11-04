# Controller node

## Overview
This file contains notes on the Controller node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Controller node.

# Basic environment

The network configuration for this node is the following:  
2 interfaces:

- eth0 : NAT
- eth1 : management network (on subnet 10.10.10.0/24)

**Configure networking**

1. Modify /etc/network/interfaces to configure eth1:

  ```
  # Management network interface
  auto eth1
  iface eth1 inet static
  address 10.10.10.11
  netmask 255.255.255.0
  gateway 10.10.10.1
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
