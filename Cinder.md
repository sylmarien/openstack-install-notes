# Cinder

## Overview

This file contains notes on the install procedure for the Cinder module of OpenStack. This documentation is based on the following official documentation pages:  
[Controller instructions](http://docs.openstack.org/icehouse/install-guide/install/apt/content/cinder-controller.html)  
[Storage service node instructions (Ubuntu)](http://docs.openstack.org/icehouse/install-guide/install/apt/content/cinder-node.html)  
[Storage service node instructions (Centos)](http://docs.openstack.org/icehouse/install-guide/install/yum/content/cinder-node.html)  
[Openstack packages for CentOS](http://docs.openstack.org/icehouse/install-guide/install/yum/content/basics-packages.html)  
[NetApp E-series Driver](http://netapp.github.io/openstack-deploy-ops-guide/icehouse/content/cinder.eseries.iscsi.configuration.html)  

Go to:
- [NetApp configuration](#netapp-configuration)
- [Store VM configuration](#store-vm-configuration)
- [Storage node configuration](#storage-node-configuration)

## NetApp configuration

## Install on the store VM

1. Install the packages:

    `apt-get install cinder-api cinder-scheduler`
2. Configure database access:
    1. **On the database VM**, create the _cinder_ database and set its password:

        ```
        mysql -u root -p
        ```
        ```
        CREATE DATABASE cinder;
        GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
        GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
        ```
        Where _CINDER_DBPASS_ is the password for this database.
    2. Modify `/etc/cinder/cinder.conf`:

        ```
        [database]
        ...
        connection = mysql://cinder:CINDER_DBPASS@pyro-database/cinder
        ```
        Where CINDER_DBPASS is the password you chose for the cinder database
    3. Populate the database:

        `su -s /bin/sh -c "cinder-manage db sync" cinder`
3. Create the _cinder_ user in Keystone:

        ```
        keystone user-create --name=cinder --pass=CINDER_PASS --email=cinder@example.com
        keystone user-role-add --user=cinder --tenant=service --role=admin
        ```
        Where _CINDER_PASS_ is an appropriate password.
4. Modify `/etc/cinder/cinder.conf` to:
    1. Configure the keystone credentials:

        ```
        [keystone_authtoken]
        ...
        auth_uri = http://pyro-core:5000
        auth_host = pyro-core
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = cinder
        admin_password = CINDER_PASS
        ```
        Where _CINDER_PASS_ is the password of the _cinder_ user.
    2. Configure the RabbitMQ message broker:
    
        ```
        [DEFAULT]
        ...
        rpc_backend = rabbit
        rabbit_host = pyro-core
        rabbit_port = 5672
        rabbit_userid = guest
        rabbit_password = RABBIT_PASS
        ```
        Where _RABBIT_PASS_ is the password of the guest user of rabbit.
5. Register the cinder service and its endpoints in Keystone (with both v1 and v2 of the API):

    ```
    keystone service-create --name=cinder --type=volume --description="OpenStack Block Storage"
    keystone endpoint-create --service-id=$(keystone service-list | awk '/ volume / {print $2}') --publicurl=http://pyro-store:8776/v1/%\(tenant_id\)s --internalurl=http://pyro-store:8776/v1/%\(tenant_id\)s --adminurl=http://pyro-store:8776/v1/%\(tenant_id\)s
    ```
    ```
    keystone service-create --name=cinderv2 --type=volumev2 --description="OpenStack Block Storage v2"
    keystone endpoint-create --service-id=$(keystone service-list | awk '/ volumev2 / {print $2}') --publicurl=http://pyro-store:8776/v2/%\(tenant_id\)s --internalurl=http://pyro-store:8776/v2/%\(tenant_id\)s --adminurl=http://pyro-store:8776/v2/%\(tenant_id\)s
    ```
6. Configure the NetApp backend in `/etc/cinder/cinder.conf`:
    1. Define the ESeries as the backend:  

        ```
        [DEFAULT]
        ...
        enabled_backends=NetApp_ESeries
        ```
    2. Configure the backend:

        ```
        [NetApp_ESeries]
        volume_backend_name=NetApp_ESeries
        volume_driver=cinder.volume.drivers.netapp.common.NetAppDriver
        netapp_server_hostname=FRONTEND_IP
        netapp_server_port=8080
        netapp_storage_protocol=iscsi
        netapp_storage_family=eseries
        netapp_login=NETAPP_USER
        netapp_password=NETAPP_USER_PASSWD
        netapp_sa_password=NETAPP_SA_PASSWD
        netapp_controller_ips=NETAPP_CTRL_IPS 
        netapp_storage_pools=NETAPP_STORAGE_POOLS
        ```
        Where _NETAPP_USER_ is the name of a NetApp user with read and write permissions, _NETAPP_USER_PASSWD_ its password, _NETAPP_SA_PASSWD_ the storage array password, _FRONTEND_IP_ the IP address of the frontend server of the NetApp (10.89.200.200 in our case), _NETAPP_CTRL_IPS_ a coma-separated list of the IP addresses of the controller ports of the NetApp (in our case: 10.89.210.201,10.89.210.202), and _NETAPP_STORAGE_POOLS_ a coma-separated list of the name of the storage pools on the NetApp to be used by Cinder (in our case: OpenStack_Pool1).
7. Restart the services:

    ```
    service cinder-scheduler restart
    service cinder-api restart
    ```

## Storage node configuration

**Openstack packages basic configuration:**

```
yum install yum-plugin-priorities
yum install http://repos.fedorapeople.org/repos/openstack/openstack-icehouse/rdo-release-icehouse-3.noarch.rpm
yum install http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
yum install openstack-utils
yum install openstack-selinux
yum upgrade
reboot
```


**Cinder configuration:**

1. Install the cinder packages:

    ```
    yum install openstack-cinder scsi-target-utils
    ```
2. Copy the `/etc/cinder/cinder.conf` from the store VM to use as a base for this server.
3. Add the following fields:

    ```
    [DEFAULT]
    ...
    my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS

    glance_host = pyro-store
    ```
    Where _MANAGEMENT_INTERFACE_IP_ADDRESS_ is the IP address of this server on the management network (10.89.200.200 in our case).
4. Restart the services and configure them to start on boot:

    ```
    service openstack-cinder-volume start
    service tgtd start
    chkconfig openstack-cinder-volume on
    chkconfig tgtd on
    ```
