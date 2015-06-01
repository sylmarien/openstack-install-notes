# Compute node

## Overview

This file contains notes on the Compute nodes of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Controller node.

## Basic environment

The network configuration for these nodes is the following:  

- em1: 10.89.200.11->14
- em2: 10.89.210.11->14

**Configure networking**

1. Content of `/etc/network/interfaces` should look like:
  
    ```
# The loopback network interface
auto lo
iface lo inet loopback

auto em1
iface em1 inet static
  post-up ifconfig p2p1 down
  post-up ifconfig p2p1 up
  address 10.89.200.11
  netmask 255.255.0.0
  broadcast 10.89.255.255
  gateway 10.89.0.1
  dns-nameservers 10.28.0.4 10.28.0.5
  dns-search uni.lux

auto p2p1
iface p2p1 inet static
  address 10.89.210.11
  netmask 255.255.0.0
  broadcast 10.89.255.255
  mtu 9000
    ```
2. Modify /etc/hosts to configure the name resolution:

  ```
127.0.0.1       localhost

# Controller nodes
10.89.200.1       pyro-ctrl1
10.89.200.2       pyro-ctrl2

# Compute nodes
10.89.200.11      pyro-comp1
10.89.200.12      pyro-comp2
10.89.200.13      pyro-comp3
10.89.200.14      pyro-comp4

# Storage node (NetApp front-end)
10.89.200.200     pyro-storage

# Horizon
10.89.201.2       pyro-horizon

# Core
10.89.201.1       pyro-core

# Store
10.89.201.3       pyro-store

# Database
10.89.201.4       pyro-database
  ```
3. Reboot to activate changes.

**Configure NTP**

1. Install the NTP service:  
  `apt-get update && apt-get dist-upgrade && apt-get install ntp`
2. Configure the NTP service:
  1. Modify `/etc/ntp.conf` to comment or remove all server key but one to reference the controller node:

      ```
      # OpenStack architecture reference
      server pyro-ctrl1 iburst dynamic
      ```
  2. Remove the `/var/lib/ntp/ntp.conf.dhcp` file if it exists.
3. Restart the NTP service:  
  `service ntp restart`

**Database**

Install the packages:  
  `apt-get install mariadb-client python-mysqldb`

**Basic prerequisites**

1. Install _python-software-properties_ package to ease repository management:  
  `apt-get install python-software-properties`
2. Enable the OpenStack repository (seems not necessary on Ubuntu 14.04 since Icehouse is the default version):  
  `add-apt-repository cloud-archive:icehouse`
3. Upgrade the packages on the system:  
  `apt-get update && apt-get dist-upgrade`

## Nova configuration

Follow instructions at [Nova.md](https://github.com/sylmarien/openstack-install-notes/blob/master/Nova.md#compute-node-installation)