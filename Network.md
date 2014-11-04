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

