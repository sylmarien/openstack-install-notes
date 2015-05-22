# Nova

## Overview

This file contains notes on the install procedure for the Nova module of OpenStack.  
***Important note: This document currently describe a very generic configuration, it should be adapted to describe the configuration that we will actually deploy. (Will be done as soon as we know more about this configuration)***

## Install on the nodes

As of now, I follow the instructions of the documentation at this page:  
[Nova Install in the OpenStack documentation](http://docs.openstack.org/juno/install-guide/install/apt/content/ch_nova.html)

Go to:  
[Core VM installation](https://github.com/sylmarien/openstack-install-notes/blob/master/Nova.md#core-vm-installation)  
[Compute node installation](https://github.com/sylmarien/openstack-install-notes/blob/master/Nova.md#compute-node-installation)

#### List of installed modules by node
**Controller side:**

- nova-api
- nova-cert
- nova-conductor
- nova-consoleauth
- nova-novncproxy
- nova-scheduler
- python-novaclient
- python-mysqldb

**Compute node:**

- nova-compute-kvm
- python-mysqldb

#### Core VM installation
**Prerequisites:** Before installing and configure Compute, we must create a database and Identity service credentials including endpoints.

1. Create the database:
    1. Access the database server as the _root_ user **on the databse VM**:  
        `mysql -u root -p`
    2. Create the _nova_ database **on the databse VM**:  
        `CREATE DATABASE nova;`
    3. Grant proper access to the _nova_ database **on the databse VM**:
    
        ```
        GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
        GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
        ```
    Replacing _NOVA_DBPASS_ with a suitable password.
2. Source the _admin_ credentials to gain access to admin-only CLI commands **on the core VM**:  
    `source admin-openrc.sh`
3. Create the Identity service credentials **on the core VM**:
    1. Create the _nova_ user:  
        `keystone user-create --name nova --pass NOVA_PASS`  
    Replacing _NOVA_PASS_ with a suitable password.
    2. Link the _nova_ user to the _service_ tenant and _admin_ role:  
        `keystone user-role-add --user nova --tenant service --role admin`
    3. Create the _nova_ service:  
        `keystone service-create --name nova --type compute --description "OpenStack Compute"`
4. Create the Compute service endpoints **on the core VM**:

    ```
    keystone endpoint-create --service-id $(keystone service-list | awk '/ compute / {print $2}') --publicurl http://pyro-core:8774/v2/%\(tenant_id\)s --internalurl http://pyro-core:8774/v2/%\(tenant_id\)s --adminurl http://pyro-core:8774/v2/%\(tenant_id\)s --region regionOne
    ```

**Installation and configuration of the Compute Controller component**

1. Install the packages:

    ```
    apt-get install nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient python-mysqldb
    ```
2. Modify `/etc/nova/nova.conf` to:
    1. Configure database access:
    
        ```
        [database]
        ...
        connection = mysql://nova:NOVA_DBPASS@pyro-database/nova
        ```
    Replacing _NOVA_DBPASS_ with the password you chose for the nova database.
    2. Configure RabbitMQ message broker access:
    
        ```
        [DEFAULT]  
        ...
        rpc_backend = rabbit
        rabbit_host = pyro-core
        rabbit_userid = guest
        rabbit_password = RABBIT_PASS
        ```
    Replacing _RABBIT_PASS_ with the password you chose for the guest account in RabbitMQ.
    3. Configure the Identity service access:
    
        ```
        [DEFAULT]
        ...
        auth_strategy = keystone
        ```
        ```
        [keystone_authtoken]
        ...
        auth_uri = http://pyro-core:5000
        auth_host = pyro-core
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = nova
        admin_password = NOVA_PASS
        ```
    Replacing _NOVA_PASS_ with the password you chose for the _nova_ user in the Identity service.
    4. Configure the IP to be the management interface IP address of the core VM and configure the VNC proxy to use this same IP:
    
        ```
        [DEFAULT]
        ...
        my_ip = MANAGEMENT_IP_ADDRESS
        vncserver_listen = MANAGEMENT_IP_ADDRESS
        vncserver_proxyclient_address = MANAGEMENT_IP_ADDRESS
        ```  
        Replacing _MANAGEMENT_IP_ADDRESS_ with the IP address of the management interface. (10.89.201.1 in this temporary configuration).
    5. Configure the location of the Image service:
    
        ```
        [DEFAULT]
        ...
        glance_host = pyro-store
        ```
    6. Set the logging to verbose for troubleshooting purpose (optional):
    
        ```
        [DEFAULT]
        ...
        verbose=True
        ```
3. Populate the Compute database:  
    `su -s /bin/sh -c "nova-manage db sync" nova`
4. Restart the Compute services:

    ```
    service nova-api restart
    service nova-cert restart
    service nova-consoleauth restart
    service nova-scheduler restart
    service nova-conductor restart
    service nova-novncproxy restart
    ```
5. Remove the unused default SQLite database (created by the package at the installation):  
    `rm -f /var/lib/nova/nova.sqlite`

#### Compute node installation

1. Install the packages:  
    `apt-get install nova-compute-kvm python-mysqldb`
2. Modify /etc/nova/nova.conf to:  
    1. Configure RabbitMQ message broker access:
    
        ```
        [DEFAULT]
        ...
        rpc_backend = rabbit
        rabbit_host = pyro-core
        rabbit_userid = guest
        rabbit_password = RABBIT_PASS
        ```
    Replacing _RABBIT_PASS_ with the password the password you chose for the guest account in RabbitMQ.
    2. Configure the Identity service access:
    
        ```
        [DEFAULT]
        ...
        auth_strategy = keystone
        ```
        ```
        [database]
        connection = mysql://nova:NOVA_DBPASS@pyro-database/nova
        ```
        Replacing _NOVA_DBPASS_ with the password you chose for the nova database.  
        ```
        [keystone_authtoken]
        ...
        auth_uri = http://pyro-core:5000
        auth_host = pyro-core
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = nova
        admin_password = NOVA_PASS
        ```
    Replacing _NOVA_PASS_ with the password you chose for the nova user in the Identity service.
    3. Configure _my_ip_ to be the IP address of the management interface of this Compute node:
    
        ```
        [DEFAULT]
        ...
        my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
        ```
    Replacing _MANAGEMENT_INTERFACE_IP_ADDRESS_ with the IP address of the management interface of the compute node. (In this example 10.89.200.11->14)
    4. Enable and configure remote console access:
    
        ```
        [DEFAULT]
        ...
        vnc_enabled = True
        vncserver_listen = 0.0.0.0
        vncserver_proxyclient_address = MANAGEMENT_INTERFACE_IP_ADDRESS
        novncproxy_base_url = http://pyro-core:6080/vnc_auto.html
        ```
    The server component listens on all IP addresses and the proxy component only listens on the management interface IP address of the compute node. The base URL indicates the location where you can use a web browser to access remote consoles of instances on this compute node.  
    Replacing _MANAGEMENT_INTERFACE_IP_ADDRESS_ with the IP address of the management interface of the compute node. (In this example 10.89.200.11->14)
    5. Configure the location of the Image service:
    
        ```
        [DEFAULT]
        ...
        glance_host = pyro-store
        ```
    6. Set the logging to verbose for troubleshooting purpose (optional):
    
        ```
        [DEFAULT]
        ...
        verbose=True
        ```
3. Determine whether your compute node supports hardware acceleration for virtual machines:  
    `egrep -c '(vmx|svm)' /proc/cpuinfo` or `kvm-ok`  
If this command returns a value of one or greater, your compute node supports hardware acceleration which typically requires no additional configuration.  
If this command returns a value of zero, your compute node does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM.
    1. Modify /etc/nova/nova-compute.conf to configure libvirt to use QEMU instead of KVM:
    
        ```
        [libvirt]
        ...
        virt_type = qemu
        ```
4. Remove the unused default SQLite database (created by the package at the installation):  
    `rm -f /var/lib/nova/nova.sqlite`
5. Restart the Compute service:  
    `service nova-compute restart`
