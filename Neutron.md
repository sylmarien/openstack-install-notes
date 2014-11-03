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
Before installing and configure Neutron, we must create a database and Identity service credentials including endpoints

1. Create the database entry:
  1. Access the database as _root_:  
    `mysql -u root -p`
  2. Create the _neutron_ database:  
    `CREATE DATABASE neutron;`
  3. Grant proper access to the _neutron_ database:
  
    ```
    GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
    GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
    ```  
    Replacing _NEUTRON_DBPASS_ with a suitable password.
2. Source the _admin_ credentials to gain access to admin-only CLI commands:  
    `source admin-openrc.sh`
3. Create the Identity service credentials:
  1. Create the _neutron_ user:  
    `keystone user-create --name neutron --pass NEUTRON_PASS`  
    Replacing _NEUTRON_PASS_ with a suitable password.
  2. Link the _neutron_ user to the _service_ tenant and _admin_ role:  
    `keystone user-role-add --user neutron --tenant service --role admin`
  3. Create the _neutron_ service:  
    `keystone service-create --name neutron --type network --description "OpenStack Networking"`
  4. Create the Identity service endpoints:
  
    ```
    keystone endpoint-create --service-id $(keystone service-list | awk '/ network / {print $2}') --publicurl http://controller:9696 --adminurl http://controller:9696 --internalurl http://controller:9696 --region regionOne
    ```

**Installation and configuration of the Network component**

1. Install the packages:  
  `apt-get install neutron-server neutron-plugin-ml2 python-neutronclient`
2. Modify /etc/neutron/neutron.conf to:
  1. Configure database access:
    
    ```
    [database]
    ...
    connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron
    ```  
    Replacing _NEUTRON_DBPASS_ with the password you chose for the database.
  2. Configure RabbitMQ message broker access:
  
    ```
    [DEFAULT]
    ...
    rpc_backend = rabbit
    rabbit_host = controller
    rabbit_password = RABBIT_PASS
    ```  
    Replacing _RABBIT_PASS_ with the password you chose for the guest account in RabbitMQ.
  3. Configure Identity service access:
  
    ```
    [DEFAULT]
    ...
    auth_strategy = keystone
    ```
    ```
    [keystone_authtoken]
    ...
    auth_uri = http://controller:5000/v2.0
    identity_uri = http://controller:35357
    admin_tenant_name = service
    admin_user = neutron
    admin_password = NEUTRON_PASS
    ```  
    Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.  
    **Note:** Comment any auth_host, auth_port, and auth_protocol options because the identity_uri option replaces them.
  4. Enable the Modlular Layer 2 (ML2) plug-in, router service, and overlapping IP addresses:
    
    ```
    [DEFAULT]
    ...
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    ```
  5. Configure Networking to notify Compute of network topology changes:
  
    ```
    [DEFAULT]
    ...
    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True
    nova_url = http://controller:8774/v2
    nova_admin_auth_url = http://controller:35357/v2.0
    nova_region_name = regionOne
    nova_admin_username = nova
    nova_admin_tenant_id = SERVICE_TENANT_ID
    nova_admin_password = NOVA_PASS
    ```  
    Replacing _SERVICE_TENANT_ID_ with the _service_ tenant identifier (id) in the Identity service  
    and _NOVA_PASS_ with the password you chose for the _nova_ user in the Identity service.  
      **Note:** To obtain _service_ tenant identifier:  
        `keystone tenant-get service`
  6. Set the logging to verbose for troubleshooting purpose (optional):
  
    ```
    [DEFAULT]
    ...
    verbose = True
    ```
3. By default, distribution packages configure Compute to use legacy networking. You must reconfigure Compute to manage networks through Networking. Modify /etc/nova/nova.conf to:
  1. Configure the APIs and drivers:
  
    ```
    [DEFAULT]
    ...
    network_api_class = nova.network.neutronv2.api.API
    security_group_api = neutron
    linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    ```  
    Note: By default, Compute uses an internal firewall service. Since Networking includes a firewall service, you must disable the Compute firewall service by using the nova.virt.firewall.NoopFirewallDriver firewall driver.
  2. Configure access parameters:
  
    ```
    [neutron]
    ...
    url = http://controller:9696
    auth_strategy = keystone
    admin_auth_url = http://controller:35357/v2.0
    admin_tenant_name = service
    admin_username = neutron
    admin_password = NEUTRON_PASS
    ```  
    Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.
4. Populate the database:

  ```
  su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade juno" neutron
  ```
5. Restart the Compute services:

  ```
  service nova-api restart
  service nova-scheduler restart
  service nova-conductor restart
  ```
6. Restart the Networkin service:  
  `service neutron-server restart`

#### Network node install
