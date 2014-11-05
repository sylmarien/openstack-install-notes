# Neutron

## Overview

This file contains notes on the install procedure for the Neutron module of OpenStack.
Important note: This document currently describe a very generic configuration, it should be adapted to describe the configuration that we will actually deploy. (Will be done as soon as we know more about this configuration)

Go to:  
[Controller node installation](#controller-node-install)  
[Network node installation](#network-node-install)  
[Compute node installation](#compute-node-install)  
[Initial networks creation](#initial-networks-creation)

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


========

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

**Installation and configuration of the Network components**

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
    rpc_backend = neutron.openstack.common.rpc.impl_kombu
    rabbit_host = controller
    rabbit_password = RABBIT_PASS
    ```  
    Replacing _RABBIT_PASS_ with the password you chose for the _guest_ account in RabbitMQ.
  3. Configure Identity service access:
  
    ```
    [DEFAULT]
    ...
    auth_strategy = keystone
    ```
    ```
    [keystone_authtoken]
    ...
    auth_uri = http://controller:5000
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = neutron
    admin_password = NEUTRON_PASS
    ```  
    Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.  
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
    [DEFAULT]
    ...
    neutron_url = http://controller:9696
    neutron_auth_strategy = keystone
    neutron_admin_auth_url = http://controller:35357/v2.0
    neutron_admin_tenant_name = service
    neutron_admin_username = neutron
    neutron_admin_password = NEUTRON_PASS
    ```  
    Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.
4. Configure ML2 plug-in. Modify /etc/neutron/plugins/ml2/ml2_conf.ini to add the following keys:
    
    ```
    [ml2]
    ...
    type_drivers = gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    ```
    ```
    [ml2_type_gre]
    ...
    tunnel_id_ranges = 1:1000
    ```
    ```
    [securitygroup]
    ...
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    enable_security_group = True
    ```
5. Restart the Compute services:

  ```
  service nova-api restart
  service nova-scheduler restart
  service nova-conductor restart
  ```
6. Restart the Networkin service:  
  `service neutron-server restart`

**Troubleshooting**  
Unlike other services, Networking typically does not require a separate step to populate the database because the _neutron-server_ service populates it automatically. However, the packages for these distributions sometimes require running the _neutron-db-manage_ command prior to starting the _neutron-server_ service. We recommend attempting to start the service before manually populating the database. If the service returns database errors, perform the following operations:

1. Configure Networking to use long plug-in names:

  ```
  openstack-config --set /etc/neutron/neutron.conf DEFAULT core_plugin neutron.plugins.ml2.plugin.Ml2Plugin
  openstack-config --set /etc/neutron/neutron.conf DEFAULT service_plugins neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
  ```
2.Populate the database:

  ```
  su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade icehouse" neutron
  ```
3. Attempt to start the _neutron-server_ service again. You can return the _core_plugin_ and _service_plugins_ configuration keys to short plug-in names.

========

#### Network node install
**Prerequisites:** Before you install and configure OpenStack Networking, you must configure certain kernel networking parameters.

1. Modify /etc/sysctl.conf to add the following parameters:

  ```
  net.ipv4.ip_forward=1
  net.ipv4.conf.all.rp_filter=0
  net.ipv4.conf.default.rp_filter=0
  ```
2. Implement changes:  
  `sysctl -p`

**Installation and configuration of the Network components**

1. Install the packages:

  ```
  apt-get install neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent
  ```
2. Modify /etc/neutron/neutron.conf to:
  1. Comment any _connection_ options in the `[database]` section because network nodes do not directly access the database.
  2. Configure RabbitMQ message broker access:
  
    ```
    [DEFAULT]
    rpc_backend = neutron.openstack.common.rpc.impl_kombu
    rabbit_host = controller
    rabbit_password = RABBIT_PASS
    ```  
    Replacing _RABBIT_PASS_ with the password you chose for the _guest_ account in RabbitMQ.
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
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = neutron
    admin_password = NEUTRON_PASS
    ```  
    Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.  
  4. Enable the ML2 plug-in, router service and overlapping addresses:
  
    ```
    [DEFAULT]
    ...
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    ```
  5. Set the logging to verbose for troubleshooting purpose (optional):
  
    ```
    [DEFAULT]
    ...
    verbose = True
    ```
3. Configure the ML2 plug-in. Modify /etc/neutron/plugins/ml2/ml2_conf.ini to:
  1. enable the generic routing encapsulation (GRE) network type drivers, GRE tenant networks, and the OVS mechanism driver:
  
    ```
    [ml2]
    ...
    type_drivers = gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    ```
  2. Configure the tunnel identifier (id) range:
  
    ```
    [ml2_type_gre]
    ...
    tunnel_id_ranges = 1:1000
    ```
  3. Enable security groups and configure the OVS iptables firewall drivers:
  
    ```
    [securitygroup]
    ...
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    ```
  4. Configure the Open vSwitch (OVS) agent:
  
    ```
    [ovs]
    ...
    local_ip = INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS
    tunnel_type = gre
    enable_tunneling = True
    ```  
    Replacing _INSTANCE_TUNNELS_INTERFACE_IP_ADDRESSES_ with the IP address of the instance tunnels network interface on your network node. (In this example: 10.20.20.21)
4. Configure the Layer-3 (L3) agent. Modify /etc/neutron/l3_agent.ini to:
  1. Configure the driver, enable network namespaces:
  
    ```
    [DEFAULT]
    ...
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    use_namespaces = True
    ```
  2. Set the logging to verbose for troubleshooting purpose (optional):
  
    ```
    [DEFAULT]
    ...
    verbose = True
    ```
5. Configure the DHCP agent.
  1. Modify /etc/neutron/dhcp_agent.ini to:
    1. Configure the drivers and enable namespaces:
    
      ```
      [DEFAULT]
      ...
      interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
      dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
      use_namespaces = True
      ```
    2. Set the logging to verbose for troubleshooting purpose (optional):
  
      ```
      [DEFAULT]
      ...
      verbose = True
      ```
  2. (Optionnal)  
  Tunneling protocols such as GRE include additional packet headers that increase overhead and decrease space available for the payload or user data. Without knowledge of the virtual network infrastructure, instances attempt to send packets using the default Ethernet maximum transmission unit (MTU) of 1500 bytes. Internet protocol (IP) networks contain the path MTU discovery (PMTUD) mechanism to detect end-to-end MTU and adjust packet size accordingly. However, some operating systems and networks block or otherwise lack support for PMTUD causing performance degradation or connectivity failure.  
Ideally, you can prevent these problems by enabling jumbo frames on the physical network that contains your tenant virtual networks. Jumbo frames support MTUs up to approximately 9000 bytes which negates the impact of GRE overhead on virtual networks. However, many network devices lack support for jumbo frames and OpenStack administrators often lack control over network infrastructure. Given the latter complications, you can also prevent MTU problems by reducing the instance MTU to account for GRE overhead. Determining the proper MTU value often takes experimentation, but 1454 bytes works in most environments. You can configure the DHCP server that assigns IP addresses to your instances to also adjust the MTU.  
**Note:** Some cloud images ignore the DHCP MTU option in which case you should configure it using metadata, script, or other suitable method.  
      1. Modify /etc/neutron/dhcp_agent.ini to enable the dnsmask configuration file:
    
        ```
        [DEFAULT]
        ...
        dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
        ```
      2. Create and edit /etc/neutron/dhcp_agent.ini to enable the DHCP MTU option (26) and configure it to 1454 bytes:  
      `dhcp-option-force=26,1454`
      3. Kill any existing dnsmask processes:  
        `pkill dnsmask`
5. Configure the metadata agent.
  1. Modify /etc/neutron/metadata_agent.ini to:
    1. Configure access parameters:
    
      ```
      [DEFAULT]
      ...
      auth_url = http://controller:5000/v2.0
      auth_region = regionOne
      admin_tenant_name = service
      admin_user = neutron
      admin_password = NEUTRON_PASS
      ```  
      Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.
    2. Configure the metadata host:
    
      ```
      [DEFAULT]
      ...
      nova_metadata_ip = controller
      ```
    3. Configure the metadata proxy shared secret:
    
      ```
      [DEFAULT]
      ...
      metadata_proxy_shared_secret = METADATA_SECRET
      ```  
      Replacing _METADATA_SECRET_ with a suitable secret for the metadata proxy.
    4. Set the logging to verbose for troubleshooting purpose (optional):
  
      ```
      [DEFAULT]
      ...
      verbose = True
      ```
  2. **On the controller node**, modify /etc/nova/nova.conf to enable the metadata proxy and configure the secret:
  
    ```
    [DEFAULT]
    ...
    service_neutron_metadata_proxy = True
    neutron_metadata_proxy_shared_secret = METADATA_SECRET
    ```  
    Replacing _METADATA_SECRET_ with the secret you chose for the metadata proxy.
  3. **On the controller node**, restart the Compute API service:  
    `service nova-api restart`
6. Configure the Open vSwitch (OVS) service.
  1. Restart the OVS service:  
    `service openvswitch-switch restart`
  2. Add the external bridge:  
    `ovs-vsctl add-br br-ex`
  3. Add a port to the external bridge that connects to the physical external network interface:  
    `ovs-vsctl add-port br-ex INTERFACE_NAME`  
  Replacing _INTERFACE_NAME_ with the actual interface name (for example _eth2_)  
  **Note:** Depending on your network interface driver, you may need to disable generic receive offload (GRO) to achieve suitable throughput between your instances and the external network.  
  To temporarily disable GRO on the external network interface while testing your environment:  
    `ethtool -K INTERFACE_NAME gro off`
7. Restart the Networking services:

  ```
  service neutron-plugin-openvswitch-agent restart
  service neutron-l3-agent restart
  service neutron-dhcp-agent restart
  service neutron-metadata-agent restart
  ```
  
  
========
  
#### Compute node install
**Prerequisites:** Before you install and configure OpenStack Networking, you must configure certain kernel networking parameters.

1. Modify /etc/sysctl.conf to add the following parameters:

  ```
  net.ipv4.conf.all.rp_filter=0
  net.ipv4.conf.default.rp_filter=0
  ```
2. Implement changes:  
  `sysctl -p`

**Installation and configuration of the Networking components**

1. Install the packages:  
  `apt-get install neutron-plugin-ml2 neutron-plugin-openvswitch-agent`
2. Modify /etc/neutron/neutron.conf to:
  1. Comment any _connection_ options in the `[database]` section because the compute nodes do not directly access the database.
  2. Configure RabbitMQ message broker access:
    
    ```
    [DEFAULT]
    ...
    rpc_backend = neutron.openstack.common.rpc.impl_kombu
    rabbit_host = controller
    rabbit_password = RABBIT_PASS
    ```  
    Replacing _RABBIT_PASS_ with the password you chose for the _guest_ account in RabbitMQ.
  3. Configure the Identity service access:
  
    ```
    [DEFAULT]
    ...
    auth_strategy = keystone
    ```
    ```
    [keystone_authtoken]
    ...
    auth_uri = http://controller:5000/v2.0
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = neutron
    admin_password = NEUTRON_PASS
    ```  
    Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.  
  4. Enable the ML2 plug-in, router service and overlapping addresses:
  
    ```
    [DEFAULT]
    ...
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    ```
  5. Set the logging to verbose for troubleshooting purpose (optional):
  
    ```
    [DEFAULT]
    ...
    verbose = True
    ```
3. Configure the ML2 plug-in. Modify /etc/neutron/plugins/ml2/ml2_conf.ini to:
  1. Enable the generic routing encapsulation (GRE) network type drivers, GRE tenant networks, and the OVS mechanism driver:
  
    ```
    [ml2]
    ...
    type_drivers = gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    ```
  2. Configure the tunnel identifier (id) range:
  
    ```
    [ml2_type_gre]
    ...
    tunnel_id_ranges = 1:1000
    ```
  3. Enable security groups, enable ipset, and configure the OVS iptables firewall driver:
  
    ```
    [securitygroup]
    ...
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    ```
  4. Configure the Open vSwitch (OVS) agent:
  
    ```
    [ovs]
    ...
    local_ip = INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS
    tunnel_type = gre
    enable_tunneling = True
    ```  
    Replacing _INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS_ with the IP address of the instance tunnels network interface on your compute node. (In this example 10.20.20.31)
4. Configure the Open vSwitch (OVS) service. Restart the OVS service:  
  `service openvswitch-switch restart`
5. Configure Compute to use Networking. By default, distribution packages configure Compute to use legacy networking. You must reconfigure Compute to manage networks through Networking. Modify /etc/nova/nova.conf to:
  1. Configure the APIs and drivers:
  
    ```
    [DEFAULT]
    ...
    network_api_class = nova.network.neutronv2.api.API
    security_group_api = neutron
    linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    ```  
    **Note:** By default, Compute uses an internal firewall service. Since Networking includes a firewall service, you must disable the Compute firewall service by using the nova.virt.firewall.NoopFirewallDriver firewall driver.
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
6. Restart the Compute service:  
  `service nova-compute restart`
7. Restart the Open vSwitch (OVS) agent:  
  `service neutron-plugin-openvswitch-agent restart`


========

#### Initial networks creation
1. External network. **On the Controller node**  
The external network typically provides Internet access for your instances. By default, this network only allows Internet access _from_ instances using Network Address Translation (NAT). You can enable Internet access _to_ individual instances using a floating IP address and suitable security group rules. The admin tenant owns this network because it provides external network access for multiple tenants. You must also enable sharing to allow access by those tenants.
  1. Create the external network:
    1. Source the _admin_ credentials to gain access to admin-only CLI commands:  
      `source admin-openrc.sh`
    2. Create the network:
      
      ```
      neutron net-create ext-net --shared --router:external True --provider:physical_network external --provider:network_type flat
      ```
  2. Create a subnet on the external network:
    
    ```
    neutron subnet-create ext-net --name ext-subnet --allocation-pool start=FLOATING_IP_START,end=FLOATING_IP_END --disable-dhcp --gateway EXTERNAL_NETWORK_GATEWAY EXTERNAL_NETWORK_CIDR
    ```  
    Replacing _FLOATING_IP_START_ and _FLOATING_IP_END_ with the first and last IP addresses of the range that you want to allocate for floating IP addresses. Replace _EXTERNAL_NETWORK_CIDR_ with the subnet associated with the physical network. Replace _EXTERNAL_NETWORK_GATEWAY_ with the gateway associated with the physical network, typically the ".1" IP address. You should disable DHCP on this subnet because instances do not connect directly to the external network and floating IP addresses require manual assignment.
2. Tenant network. **On the Controller node**  
The tenant network provides internal network access for instances. The architecture isolates this type of network from other tenants. The demo tenant owns this network because it only provides network access for instances within it.
  1. Create the tenant network:  
    `neutron net-create demo-net`
  2. Create a subnet on the tenant network:
    
    ```
    neutron subnet-create demo-net --name demo-subnet --gateway TENANT_NETWORK_GATEWAY TENANT_NETWORK_CIDR
    ```  
    Replacing _TENANT_NETWORK_CIDR_ with the subnet you want to associate with the tenant network and _TENANT_NETWORK_GATEWAY_ with the gateway you want to associate with it, typically the ".1" IP address.
3. Create a router on the tenant network and attach the external and tenant network to it:
  1. Create the router:
    `neutron router-create demo-router`
  2. Attach the router to the demo tenant subnet:  
    `neutron router-interface-add demo-router demo-subnet`
  3. Attach the router to the external network by setting it as the gateway:  
    `neutron router-gateway-set demo-router ext-net`
