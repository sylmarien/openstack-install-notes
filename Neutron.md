# Neutron

## Overview

This file contains notes on the install procedure for the Neutron module of OpenStack.
Important note: This document currently describe a very generic configuration, it should be adapted to describe the configuration that we will actually deploy. (Will be done as soon as we know more about this configuration)

## Install on the nodes

As of now, I follow the instructions of the documentation at this page:  
http://docs.openstack.org/juno/install-guide/install/apt/content/neutron-controller-node.html

#### List of installed modules by node:

**Controller node:**
- neutron-server
- neutron-plugin-ml2
- python-neutronclient

**Network node:**
- neutron-plugin-ml2
- neutron-plugin-openvswitch-agent
- neutron-l3-agent
- neutron-dhcp-agent

**Compute node:**
- neutron-plugin-ml2
- neutron-plugin-openvswitch-agent

#### Controller node install
**Prerequisites:**
