# Neutron

## Overview

This file contains notes on the install procedure for the Neutron module of OpenStack.

Go to:  
[Core VM installation](#core-vm-install)  
[Compute node installation](#compute-node-install)  
[Initial networks creation](#initial-networks-creation)

## Install on the nodes

As of now, I follow the instructions of the documentation at this page:  
http://docs.openstack.org/juno/install-guide/install/apt/content/neutron-controller-node.html

#### List of installed modules by node:

**Core VM:**
- neutron-server
- python-neutronclient
- python-mysqldb

**Network node:**
- neutron-plugin-ml2
- neutron-plugin-openvswitch-agent
- neutron-l3-agent
- neutron-dhcp-agent

**Compute node:**
- neutron-plugin-ml2
- neutron-plugin-openvswitch-agent


========

#### Core VM install

**Prerequisites:**  
Before installing and configure Neutron, we must create a database and Identity service credentials including endpoints

1. Create the database entry **on the database VM**:
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
    keystone endpoint-create --service-id $(keystone service-list | awk '/ network / {print $2}') --publicurl http://pyro-core:9696 --adminurl http://pyro-core:9696 --internalurl http://pyro-core:9696 --region regionOne
    ```

**Installation and configuration of the Network components**

1. Install the packages:

  ```
  apt-get install neutron-server neutron-plugin-ml2 
  ```
2. Modify `/etc/neutron/neutron.conf` to:
  1. Configure database access:
    
    ```
    [database]
    ...
    connection = mysql://neutron:NEUTRON_DBPASS@pyro-database/neutron
    ```  
    Replacing _NEUTRON_DBPASS_ with the password you chose for the database.
  2. Configure RabbitMQ message broker access:
  
    ```
    [DEFAULT]
    ...
    rpc_backend = neutron.openstack.common.rpc.impl_kombu
    rabbit_host = pyro-core
    rabbit_userid = guest
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
    auth_uri = http://pyro-core:5000
    auth_host = pyro-core
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
    nova_url = http://pyro-core:8774/v2
    nova_admin_auth_url = http://pyro-core:35357/v2.0
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
3. By default, distribution packages configure Compute to use legacy networking. You must reconfigure Compute to manage networks through Networking. Modify `/etc/nova/nova.conf` to:
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
    neutron_url = http://pyro-core:9696
    neutron_auth_strategy = keystone
    neutron_admin_auth_url = http://pyro-core:35357/v2.0
    neutron_admin_tenant_name = service
    neutron_admin_username = neutron
    neutron_admin_password = NEUTRON_PASS
    ```  
    Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.
4. Configure ML2 plug-in. Modify `/etc/neutron/plugins/ml2/ml2_conf.ini` to add the following keys:
    
    ```
    [ml2]
    ...
    type_drivers = flat,gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    ```
    ```
    [ml2_type_gre]
    ...
    tunnel_id_ranges = 1:1000
    ```
    That means that our network will be able to be create on physical networks named `prod` or `data`. Nothing else.
    ```
    [securitygroup]
    ...
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
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

#### Network node install

**Prerequisites:**  
Configure some kernel networking parameters:
  1. Modify `/etc/sysctl.conf` to add the following parameters:

    ```
    net.ipv4.ip_forward=1
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0
    ```
  2. Implement changes:  
    `sysctl -p`

Network settings. Modify `/etc/network/interfaces`:

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto em1
iface em1 inet static
  post-up ifconfig p2p1 down
  post-up ifconfig p2p1 up
  post-up ifconfig p2p2 down
  post-up ifconfig p2p2 up
  address 10.89.200.2
  netmask 255.255.0.0
  broadcast 10.89.255.255
  gateway 10.89.0.1
  dns-nameservers 10.28.0.4 10.28.0.5
  dns-search uni.lux

auto p2p1
iface p2p1 inet static
  address 10.89.210.2
  netmask 255.255.0.0
  broadcast 10.89.255.255
  mtu 9000

auto p2p2
iface p2p2 inet manual
  up ip link set dev $IFACE up
  up ip link set $IFACE promisc on
  down ip link set $IFACE promisc off
  down ip link set dev $IFACE down
```

**Installation and configuration**

1. Install packages:

    ```
    apt-get install neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent
    ```
2. Configure Neutron common components. Modifications are applied in `/etc/neutron/neutron.conf`:
  1. Configure Neutron to use the Identity service for authentication:

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
      auth_protocol = http
      auth_port = 35357
      admin_tenant_name = service
      admin_user = neutron
      admin_password = NEUTRON_PASS
      ```
      Where _NEUTRON_PASS_ is the password you chose for the _neutron_ user in the Identity service.
    2. Configure Neutron to use the message broker:

      ```
      [DEFAULT]
      ...
      rpc_backend = neutron.openstack.common.rpc.impl_kombu
      rabbit_host = pyro-core
      rabbit_userid = guest
      rabbit_password = RABBIT_PASS
      ```
      Where _RABBIT_PASS_ is the password you chose for the guest user in RabbitMQ.
    3. Configure Neutron to use the ML2 plug-in and associated services:

      ```
      [DEFAULT]
      ...
      core_plugin = ml2
      service_plugins = router
      allow_overlapping_ips = True
      ```
    4. Set the logging to verbose for troubleshooting purpose (optional):

        ```
        [DEFAULT]
        ...
        verbose = True
        ```
3. Configure the Layer-3 (L3) agent. Modify `/etc/neutron/l3_agent.ini` to:
  1. Configure the driver, enable network namespaces:
  
    ```
    [DEFAULT]
    ...
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    use_namespaces = True
    external_network_bridge = br-ex
    router_delete_namespaces = True
    ```
  2. Set the logging to verbose for troubleshooting purpose (optional):
  
    ```
    [DEFAULT]
    ...
    verbose = True
    ```
4. Configure the DHCP agent. Modifications applied in `/etc/neutron/dhcp_agent.ini`.
  1. Configure the drivers and enable namespaces:
  
    ```
    [DEFAULT]
    ...
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    use_namespaces = True
    dhcp_delete_namespaces = True
    ```
  2. Set the dnsmasq configuration file:

    ```
    [DEFAULT]
    ...
    dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
    ```
  3. Set the logging to verbose for troubleshooting purpose (optional):

    ```
    [DEFAULT]
    ...
    verbose = True
    ```
  4. Modify `/etc/neutron/dnsmasq-neutron.conf`:

    ```
    dhcp-option-force=26,1400
    no-resolv
    server=/10.28.0.4/10.28.0.5
    ```
  5. Kill any existing dnsmasq process:  
    `pkill dnsmasq`
5. Configure the metadata agent.
  1. Modify `/etc/neutron/metadata_agent.ini` to:
    1. Configure access parameters:
    
      ```
      [DEFAULT]
      ...
      auth_url = http://pyro-core:5000/v2.0
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
      nova_metadata_ip = CORE_IP
      ```
      Where _CORE_IP_ is the IP address of the Core VM (10.89.201.1). **It has to be the actual IP, not a name to be resolved.**
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
  2. Modify `/etc/nova/nova.conf` **on the core VM** to enable the metadata proxy and configure the secret:
  
    ```
    [DEFAULT]
    ...
    service_neutron_metadata_proxy = True
    neutron_metadata_proxy_shared_secret = METADATA_SECRET
    ```  
    Replacing _METADATA_SECRET_ with the secret you chose for the metadata proxy.
  3. Restart the Compute API service **on the core VM**:  
    `service nova-api restart`
6. Configure the ML2 plug-in. Modification applied in `/etc/neutron/plugins/ml2/ml2_conf.ini`:

    ```
    [ml2]
    ...
    type_drivers = flat,gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    ```
    ```
    [ml2_type_flat]
    ...
    flat_networks = external
    ```
    That means that our network will be able to be create on physical networks named `external`. Nothing else.
    ```
    [ml2_type_gre]
    ...
    tunnel_id_ranges = 1:1000
    ```
    ```
    [securitygroup]
    ...
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    ```
    ```
    [ovs]
    ...
    local_ip = INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS
    enable_tunneling = True
    bridge_mappings = external:br-ex
    ```  
    Replacing _INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS_ with the IP address of the instance tunnels network interface of the Networking node. (IP address on the data network)
    ```
    [agent]
    ...
    tunnel_types = gre
    ```
7. Configure the Open vSwitch (OVS) service.  
  1. Restart the OVS service:  
    `service openvswitch-switch restart`
  2. Add the external bridge:  
    `ovs-vsctl add-br br-ex`
  3. Add a port connecting this bridge to the interface of the Networking node and change the configuration accordingly (connectivity is going to be lost, so make sure your are not dependent on it. Or run all the following command at once in a script):  
    `ovs-vsctl add-port br-ex p2p2`
8. Restart the Networking services (not needed if the reboot has been done):

  ```
  service neutron-plugin-openvswitch-agent restart
  service neutron-l3-agent restart
  service neutron-dhcp-agent restart
  service neutron-metadata-agent restart
  ```
  
========
  
#### Compute node install
**Prerequisites:** Before you install and configure OpenStack Networking, you must configure certain kernel networking parameters.

1. Modify `/etc/sysctl.conf` to add the following parameters:

  ```
  net.ipv4.conf.all.rp_filter=0
  net.ipv4.conf.default.rp_filter=0
  ```
2. Implement changes:  
  `sysctl -p`

**Installation and configuration of the Networking components**

1. Install the packages:  
  `apt-get install neutron-common neutron-plugin-ml2 neutron-plugin-openvswitch-agent`
2. Modify `/etc/neutron/neutron.conf` to:
  1. Comment any _connection_ options in the `[database]` section because the compute nodes do not directly access the database.
  2. Configure RabbitMQ message broker access:
    
    ```
    [DEFAULT]
    ...
    rpc_backend = neutron.openstack.common.rpc.impl_kombu
    rabbit_host = pyro-core
    rabbit_userid = guest
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
    auth_uri = http://pyro-core:5000/v2.0
    auth_host = pyro-core
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
3. Configure the ML2 plug-in. Modify `/etc/neutron/plugins/ml2/ml2_conf.ini` to:
  1. Enable the generic routing encapsulation (GRE) network type drivers, GRE tenant networks, and the OVS mechanism driver:
  
    ```
    [ml2]
    ...
    type_drivers = flat,gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    ```
  2. Configure the tunnel identifier (id) range:
  
    ```
    [ml2_type_gre]
    ...
    tunnel_id_ranges = 1:1000
    ```
    That means that our network will be able to be create on physical networks named `prod` or `data`. Nothing else.
  3. Enable security groups and configure the OVS iptables firewall driver:
  
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
    enable_tunneling = True
    ```  
    Replacing _INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS_ with the IP address of the instance tunnels network interface on your compute node. (Interface that is in the floating IPs network)
  5. In the [agent] section, enable GRE tunnels:

    ```
    [agent]
    ...
    tunnel_types = gre
    ```
4. Configure the Open vSwitch (OVS) service.
  1.Restart the OVS service:  
    `service openvswitch-switch restart`
  2. Add the integration bridge:  
    `ovs-vsctl add-br br-int`
5. Configure Compute to use Networking. By default, distribution packages configure Compute to use legacy networking. You must reconfigure Compute to manage networks through Networking. Modify `/etc/nova/nova.conf` to:
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
    [DEFAULT]
    ...
    neutron_url = http://pyro-core:9696
    neutron_auth_strategy = keystone
    neutron_admin_auth_url = http://pyro-core:35357/v2.0
    neutron_admin_tenant_name = service
    neutron_admin_username = neutron
    neutron_admin_password = NEUTRON_PASS
    ```  
    Replacing _NEUTRON_PASS_ with the password you chose for the _neutron_ user in the Identity service.
6. Restart the Compute service:  
  `service nova-compute restart`
7. Restart the Open vSwitch (OVS) agent:  
  `service neutron-plugin-openvswitch-agent restart`

========

#### Initial networks creation
1. External network. **On the core VM**  
The external network typically provides Internet access for your instances. By default, this network only allows Internet access _from_ instances using Network Address Translation (NAT). You can enable Internet access _to_ individual instances using a floating IP address and suitable security group rules. The admin tenant owns this network because it provides external network access for multiple tenants. You must also enable sharing to allow access by those tenants.
  1. Create the external network:
    1. Source the _admin_ credentials to gain access to admin-only CLI commands:  
      `source admin-openrc.sh`
    2. Create the network:  
      `neutron net-create ext-net --shared --router:external True --provider:network_type flat --provider:physical_network external`
  2. Create a subnet on the external network:
    
    ```
    neutron subnet-create ext-net --name ext-subnet --allocation-pool start=FLOATING_IP_START,end=FLOATING_IP_END --disable-dhcp --gateway EXTERNAL_NETWORK_GATEWAY EXTERNAL_NETWORK_CIDR
    ```  
    Replacing _FLOATING_IP_START_ (10.89.1.0) and _FLOATING_IP_END_ (10.89.199.254) with the first and last IP addresses of the range that you want to allocate for floating IP addresses. Replace _EXTERNAL_NETWORK_CIDR_ (10.89.0.0/16) with the subnet associated with the physical network. Replace _EXTERNAL_NETWORK_GATEWAY_ (10.89.0.1) with the gateway associated with the physical network, typically the ".1" IP address. You should disable DHCP on this subnet because instances do not connect directly to the external network and floating IP addresses require manual assignment.
2. Tenant network. **On the core VM**  
The tenant network provides internal network access for instances. The architecture isolates this type of network from other tenants. The tenant owns this network because it only provides network access for instances within it. **You have to execute these commands as the tenant who will use them.**
  1. Create the tenant network:  
    `neutron net-create etriks-net`
  2. Create a subnet on the tenant network:
    
    ```
    neutron subnet-create etriks-net --name etriks-subnet --gateway TENANT_NETWORK_GATEWAY TENANT_NETWORK_CIDR
    ```  
    Replacing _TENANT_NETWORK_CIDR_ (192.168.1.0/24) with the subnet you want to associate with the tenant network and _TENANT_NETWORK_GATEWAY_ (192.168.1.1) with the gateway you want to associate with it, typically the ".1" IP address.
3. Create a router on the tenant network and attach the external and tenant network to it:
  1. Create the router:
    `neutron router-create etriks-router`
  2. Attach the router to the etriks tenant subnet:  
    `neutron router-interface-add etriks-router etriks-subnet`
  3. Attach the router to the external network by setting it as the gateway:  
    `neutron router-gateway-set etriks-router ext-net`
4. You then have to change the security groups rules to allow some interaction with the instances that will be created on the networks (Has to be made for each tenant). Here is a non exhaustive list of rules that can be considered:  
- To authorize pinging to and from an instance: authorize Egress and Ingress ICMP communications
- To authorize ssh connection to the instances: authorize Ingress TCP communication on port 22.
- To authorize DHCP communication with the DHCP server: authorize Egress UDP communication on port 67 and Ingress UDP communication on port 68. (Ingress port may need to be adapted depending on your instance configuration)
