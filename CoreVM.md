# Core VM

## Overview
This file contains notes on the Core VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Core VM.

Go to:
- [Keystone Configuration](#keystone-configuration)
- [Nova Configuration](#nova-configuration)
- [Neutron Configuration](#neutron-configuration)

**During development, all passwords are _localadmin_.**

## Basic configuration

Basic configuration is described in the [Controller.md](https://github.com/sylmarien/openstack-install-notes/blob/master/Controller.md) file. It is the shared configuration between all the VMs of the controller node.

**Network Configuration**

Modify `/etc/network/interfaces` to match the following content:

        # The loopback network interface
        auto lo
        iface lo inet loopback
        
        # The primary network interface
        auto eth0
        iface eth0 inet static
            post-up ifconfig eth1 down
            post-up ifconfig eth1 up
            address 10.89.201.1
            netmask 255.255.0.0
            broadcast 10.89.255.255
            gateway 10.89.0.1
            dns-nameservers 10.28.0.4 10.28.0.5
            dns-search uni.lux
        
        auto eth1
        iface eth1 inet static
            address 10.89.211.1
            netmask 255.255.0.0
            broadcast 10.89.255.255

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

**Note:** During development, we use the guest account that is automatically created by RabbitMQ, but for production, we should create a unique user with suitable password.  
Then, we would have to configure the _rabbit_userid_ and _rabbit_password_ keys accordingly in the configuration files of each OpenStack service that use the message broker.

## Keystone Configuration

Follow instructions at [keystone.md](https://github.com/sylmarien/openstack-install-notes/blob/master/Keystone.md).

## Nova Configuration

Please set up Glance before setting up Nova.

Follow instructions at [Nova.md](https://github.com/sylmarien/openstack-install-notes/blob/master/Nova.md#core-vm-installation)

## Neutron Configuration

Follow instructions at [Neutron.md](https://github.com/sylmarien/openstack-install-notes/blob/master/Neutron.md#core-vm-install)
