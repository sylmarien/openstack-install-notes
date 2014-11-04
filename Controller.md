# Controller node

## Overview
This file contains notes on the Controller node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Controller node.

# Basic environment

The network configuration for this node is the following:  
3 interfaces:

- eth0 : NAT
- eth1 : management network (on subnet 10.10.10.0/24)

**Configure networking**

Configure eth1:

  ```
  IP address : 10.10.10.11
  netmask 255.255.255.0
  gateway 10.10.10.1
  ```
