# Basic usage guide

## Overview

This file describes the steps to create a new tenants and its users and main components.

## New tenant creation and configuration

1. Keystone steps:
    1. Tenant creation:

        ```
        keystone tenant-create --name TENANT_NAME --description "TENANT_DESCRIPTION"
        ```
    where _TENANT_NAME_ is the name of the tenant created arlier, and _TENANT_DESCRIPTION_.
    2. Create a user and attribute it to the tenant:

        ```
        keystone user-create --name USER_NAME --pass USER_PASS --email EMAIL_ADDRESS
        keystone user-role-add --tenant TENANT_NAME --user USER_NAME --role _member_
        ```
    where _TENANT_NAME_ is the name of the tenant created arlier, _USER_NAME_ the name of the user and _USER_PASS_ the password of the said user.
2. Source the credentials for a user of this tenant. The required variable environment that need to be provided are:

    ```
    export OS_TENANT_NAME=TENANT_NAME
    export OS_USERNAME=USER_NAME
    export OS_PASSWORD=USER_PASS
    export OS_AUTH_URL=http://pyro-core:35357/v2.0
    export OS_REGION_NAME=regionOne
    ```
where _TENANT_NAME_ is the name of the tenant created arlier, _USER_NAME_ the name of the user created earlier and _USER_PASS_ the password of the said user.
3. Neutron steps:
    1. Create the tenant network:
        
        ```
        neutron net-create NET_NAME
        ```
        where _NET_NAME_ is the name of the network.
    2. Create a subnet on the tenant network:

        ```
        neutron subnet-create NET_NAME --name SUBNET_NAME --gateway TENANT_NETWORK_GATEWAY TENANT_NETWORK_CIDR
        ```
        Replacing _NET_NAME_ with the name of the network created previously, _SUBNET_NAME_ with the name you want to give to the subnetwork, _TENANT_NETWORK_CIDR_ with the subnet you want to associate with the tenant network (usually a 192.168.x.0/24 subnetwork) and _TENANT_NETWORK_GATEWAY_ with the gateway you want to associate with it, typically the "x.y.z.1" IP address.
    3. Create the router:

        ```
        neutron router-create ROUTER_NAME
        ```
        where _ROUTER_NAME_ is the name you want to give to the router.
    4. Attach the router to the tenant subnet:

        ```
        neutron router-interface-add ROUTER_NAME SUBNET_NAME
        ```
        where _ROUTER_NAME_ is the name of the previously created router and _SUBNET_NAME_ the name of the previously created subnetwork.
    5. Attach the router to the external network by setting it as the gateway:
        
        ```
        neutron router-gateway-set ROUTER_NAME ext-net
        ```
        where _ROUTER_NAME_ is the name of the previously created router.
    6. You then have to change the security groups rules to allow some interaction with the instances that will be created on the networks (Has to be made for each tenant). Here is a non exhaustive list of rules that can be considered:

    - To authorize pinging to and from an instance: authorize Egress and Ingress ICMP communications
    - To authorize ssh connection to the instances: authorize Ingress TCP communication on port 22.
    - To authorize DHCP communication with the DHCP server: authorize Egress UDP communication on port 67 and Ingress UDP communication on port 68. (Ingress port may need to be adapted depending on your instance configuration)
