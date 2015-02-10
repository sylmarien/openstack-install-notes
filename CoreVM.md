# Core VM

## Overview
This file contains notes on the Core VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Core VM.

**During development, all passwords are _localadmin_.**

## Basic configuration

Basic configuration is described in the [Controller.md](https://github.com/sylmarien/openstack-install-notes/blob/master/Controller.md) file. It is the shared configuration between all the VMs of the controller node.

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