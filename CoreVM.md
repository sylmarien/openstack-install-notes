# Core VM

## Overview
This file contains notes on the Core VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Core VM.

Go to:
- [Keystone Configuration](#keystone-configuration)
- [Nova Configuration](#nova-configuration)
- [Neutron Configuration](#neutron-configuration)

## Basic configuration

Basic configuration is described in the [Controller.md](Controller.md) file. It is the shared configuration between all the VMs of the controller node.

**Network Configuration**

Modify `/etc/network/interfaces` to match the following content:

```
# The loopback network interface
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
  address 10.89.201.1
  netmask 255.255.0.0
  broadcast 10.89.255.255
  gateway 10.89.0.1
  dns-nameservers 10.28.0.4 10.28.0.5
  dns-search uni.lux
```

**Basic prerequisites**

1. Install _python-software-properties_ package to ease repository management:  
  `apt-get install python-software-properties`
2. Enable the OpenStack repository (not necessary on Ubuntu 14.04 since Icehouse is the default version):  
  `add-apt-repository cloud-archive:icehouse`

**Messaging server**  
We use RabbitMQ as message broker service.

1. Install the package:  
  `apt-get install rabbitmq-server`
2. Configure the message broker service (replace the guest password):  
  `rabbitmqctl change_password guest RABBIT_PASS`  
  Replacing _RABBIT_PASS_ with a suitable password.

## Keystone Configuration

Follow instructions at [keystone.md](Keystone.md).

## Nova Configuration

Please set up Glance before setting up Nova.

Follow instructions at [Nova.md](Nova.md#core-vm-installation)

## Neutron Configuration

Follow instructions at [Neutron.md](Neutron.md#core-vm-install)