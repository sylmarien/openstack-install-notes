# Network node

# Overview

This file contains notes on the Controller node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Controller node.

# Basic environment

The network configuration for this node is the following:
4 interfaces:

- eth0 : NAT
- eth1 : Management network (subnet 10.10.10.0/24)
- eth2 : Instance tunnels (subnet 10.20.20.0/24)
- eth3 : External network (API network) (subnet 192.168.100.0/24)

**Configure networking**

1. Modify /etc//network/interfaces to configure:
  1. eth1 :
    
    ```
    # Management network interface
    auto eth1
    iface eth1 inet static
    address 10.10.10.21
    netmask 255.255.255.0
    gateway 10.10.10.1
    ```
  2. eth2 :
    
    ```
    # Instance tunnels interface
    auto eth1
    iface eth1 inet static
    address 10.20.20.21
    netmask 255.255.255.0
    ```
  3. eth3 :
  
    ```
    # External network interface
    auto eth3
    iface eth3 inet manual
        up ip link set dev $IFACE up
        down ip link set dev $IFACE down
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
