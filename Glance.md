# Glance

## Overview

This file contains notes on the install procedure for the Glance module of OpenStack.

## Install on the store node

As of now, I follow the instructions of the documentation at this page:  
[Glance Install in the OpenStack documentation](http://docs.openstack.org/juno/install-guide/install/apt/content/image-service-overview.html)

#### List of installed modules
- glance-api
- glance-registry
- python-mysqldb

#### Install steps:
**prerequisites:** Identity service (Keystone)  
Before you install and configure the Image Service, you must create a database and Identity service credentials including endpoints.

1. Create database:
    1. Connect to the database as the _root_ user:  
        `mysql -u root -p`
    2. Create the glance database:  
        `CREATE DATABASE glance;`
    3. Grant proper access to the glance database:
    
        ```
        GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
        GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
        ```
  Replacing _GLANCE_DBPASS_ with a suitable password.
2. Source the admin credentials to gain access to admin-only CLI commands **on the Core VM**:  
    `source admin-openrc.sh`
3. Create the Identity service credentials **on the Core VM**:
    1. Create _glance_ user:  
        `keystone user-create --name glance --pass GLANCE_PASS`  
    Replacing _GLANCE_PASS_ by a suitable password.
    2. Link the _glance_ user to the _service_ tenant and the _admin_ role:  
        `keystone user-role-add --user glance --tenant service --role admin`
    3. Create the _glance_ service:  
        `keystone service-create --name glance --type image --description "OpenStack Image Service"`
4. Create the Identity service endpoint **on the Core VM**:

    ```
    keystone endpoint-create --service-id $(keystone service-list | awk '/ image / {print $2}') --publicurl http://pyro-store:9292 --internalurl http://pyro-store:9292 --adminurl http://pyro-store:9292 --region regionOne
    ```
    
**Install and configure the Image Service (Glance) components**

1. Install the packages:  
    `apt-get install glance python-glanceclient python-mysqldb`
2. Modify /etc/glance/glance-api.conf to:  
    1. Configure database acccess:
    
        ```
        [database]
        ...
        connection = mysql://glance:GLANCE_DBPASS@pyro-database/glance
        ```
    Replacing _GLANCE_DBPASS_ with the password you chose for the Image Service database.
    2. Configure Identity service access:
    
        ```
        [keystone_authtoken]
        ...
        auth_uri = http://pyro-core:5000
        auth_host = pyro-core
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = glance
        admin_password = GLANCE_PASS
        ```
        ```
        [paste_deploy]
        ...  
        flavor = keystone
        ```  
        Replacing _GLANCE_PASS_ with the password you chose for the _glance_ user in the Identity service.
    3. Configure the local file system store and location of image files:
    
        ```
        [DEFAULT]
        ...
        default_store = file
        filesystem_store_datadir = /var/lib/glance/images/
        ```  
        **_file_ will be changed for the final deployment to support the Object storage or OpenStack.**
    4. Change RabbitMQ configuration:

        ```
        rabbit_host = pyro-core
        rabbit_password = RABBIT_PASSWORD
        ```
        Where `RABBIT_PASSWORD` is the password for the RaabitMQ guest account.
    5. Set the logging to verbose for troubleshooting purpose (optional):
    
        ```
        [DEFAULT]
        ...
        verbose=True
        ```
3. Modify /etc/glance/glance-registry.conf to:
    1. Configure database access:
    
        ```
        [database]
        ...
        connection = mysql://glance:GLANCE_DBPASS@pyro-database/glance
        ```  
        Replacing _GLANCE_DBPASS_ with the password you chose for the Image Service database.
    2. Configure the Identity service access:
    
        ```
        [keystone_authtoken]
        ...
        auth_uri = http://pyro-core:5000
        auth_host = pyro-core
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = glance
        admin_password = GLANCE_PASS
        ```  
        ```
        [paste_deploy]
        ...
        flavor = keystone
        ```  
        Replacing _GLANCE_PASS_ with the password you chose for the _glance_ user in the Identity service.
    3. Set the logging to verbose for troubleshooting purpose (optional):
    
        ```
        [DEFAULT]
        ...
        verbose=True
        ```
4. Populate the Image Service database:  
    `su -s /bin/sh -c "glance-manage db_sync" glance`
5. Finalization:
    1. Restart the Image Service services:
    
        ```
        service glance-registry restart
        service glance-api restart
        ```
    2. Remove the unused default SQLite database:  
        `rm -f /var/lib/glance/glance.sqlite`
