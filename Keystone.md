# Keystone

## Overview

This file contains notes on the install procedure for the Keystone module of OpenStack.

## Install on the Controller node

As of now, I follow the instructions of the documentation at this page:  
[Keystone Install in the OpenStack documentation](http://docs.openstack.org/juno/install-guide/install/apt/content/keystone-install.html)

#### List of installed modules:
- keystone
- python-keystoneclient
- python-mysqldb

#### Installation steps:
1. Create and configure database **on the database VM**
    1. Connect to the database as the _root_ user:  
        `mysql -u root -p`
    2. Create the keystone database:  
        `CREATE DATABASE keystone;`
    3. Grant proper access to the glance database:
    
        ```
        GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
        GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
        ```  
    Replacing _KEYSTONE_DBPASS_ with a suitable password.
2. Generate random value as temporary administrative token during initial configuration:  
    `openssl rand -hex 10`
    c05d9f83c7bcd1af3cce
3. Install the packages:  
    `apt-get install keystone python-keystoneclient python-mysqldb`
4. Modify /etc/keystone/keystone.conf to:
    1. Configure temporary admin access:
    
        ```
        [DEFAULT]
        ...
        admin_token=ADMIN_TOKEN
        ```
    where ADMIN_TOKEN is replaced with the randomly generated number mentioned before.
    2. Configure the access to the database:
    
        ```
        [database]
        ...
        connection = mysql://keystone:KEYSTONE_DBPASS@pyro-database/keystone
        ```
    where _KEYSTONE_DBPASS_ is the password for the database.
    3. Set the logging to verbose for troubleshooting purpose (optional):
    
        ```
        [DEFAULT]
        ...
        verbose=True
        ```
5. Populate the Identity service database:  
    `su -s /bin/sh -c "keystone-manage db_sync" keystone`
6. Restart the Identity service:  
    `service keystone restart`
7. Remove the unused default SQLite database (created by the package at the installation):  
    `rm -f /var/lib/keystone/keystone.db`
8. Configure a periodic task using cron to purge expired tokens on an hourly basis (by default, they are kept indefinitely):

    ```
    (crontab -l -u keystone 2>&1 | grep -q token_flush) || echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' >> /var/spool/cron/crontabs/keystone
    ```

#### Configuration steps:
**Prerequisites:** Configure the temporary access.

1. Temporary administrative token:  
    `export OS_SERVICE_TOKEN=ADMIN_TOKEN`  
where _ADMIN_TOKEN_ is the randomly generated number mentioned before.
2. Configure the endpoint:  
    `export OS_SERVICE_ENDPOINT=http://pyro-core:35357/v2.0`

**Tenants, users and roles creation:**

1. Administrative user:
    1. Create the _admin_ tenant:  
        `keystone tenant-create --name admin --description "Admin Tenant"`
    2. Create the _admin_ user:  
        `keystone user-create --name admin --pass ADMIN_PASS --email EMAIL_ADDRESS`  
    Replacing ADMIN_PASS and EMAIL_ADDRESS with suitable values.
    3. Create the _admin_ role:  
        `keystone role-create --name admin`
    4. Add the _admin_ tenant and user to the _admin_ role:  
        `keystone user-role-add --tenant admin --user admin --role admin`
    5. Create the _\_member\__ role and add the admin tenant and user to it (because by default the dashboard limits access to users with the _\_member\__ role):
    
        ```
        keystone role-create --name _member_
        keystone user-role-add --tenant admin --user admin --role _member_
        ```  
        Note that the _\_member\__ role may be created by default, then the first command will return an error.
2. Service tenant (services also require a tenant):  
    `keystone tenant-create --name service --description "Service Tenant"`
3. Typical tenants (to be defined depending of the needs. Example: lcsb and external could be two tenants in which users would be included depending of where they are from). Commands in the case of _lcsb_ tenant (not currently done, define the tenants first):  
    `keystone tenant-create --name lcsb --description "LCSB Tenant"`
4. Create your users and attach them to the right tenant and to the _\_member\__ role (here the user is _username_ and the tenant is _lcsb_):

    ```
    keystone user-create --name username --pass USERNAME_PASS --email EMAIL_ADDRESS
    keystone user-role-add --tenant lcsb --user username --role _member_
    ```
As always, replacing _USERNAME_PASS_ and _EMAIL_ADDRESS_ with the right values.

**Service entity and API endpoint**  
1) The Identity service manages a catalog of services in your OpenStack environment. Services use this catalog to locate other services in your environment.

2) The Identity service manages a catalog of API endpoints associated with the services in your OpenStack environment. Services use this catalog to determine how to communicate with other services in your environment.  
OpenStack provides three API endpoint variations for each service: admin, internal, and public. In a production environment, the variants might reside on separate networks that serve different types of users for security reasons. Also, OpenStack supports multiple regions for scalability. For simplicity (and since all the services are for now in the same network), this configuration uses the management network for all endpoint variations and the regionOne region.

1. Create the service _identity_ for the Identity service (keystone):  
    `keystone service-create --name keystone --type identity --description "OpenStack Identity"`
2. Create the API endpoint for the Identity service (Keystone):

    ```
    keystone endpoint-create --service-id $(keystone service-list | awk '/ identity / {print $2}') --publicurl http://pyro-core:5000/v2.0 --internalurl http://pyro-core:5000/v2.0 --adminurl http://pyro-core:35357/v2.0 --region regionOne
    ```

**Client Environment scripts**

At this point, you can use keystone, but the number of arguments to precise is quite long (--os-username admin --os-password passwd .......), so we are going to write scripts that will initialize environment variables so that once sourced, you won't need to precise all these arguments for the ongoing session.

Admin script: in a file (named for example *admin-openrc.sh*) write the following:

```bash
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://pyro-core:35357/v2.0
```  
Replacing _ADMIN_PASS_ with your admin password.

Do the same for the different users you'd have to login as.

Then just source (`source admin-openrc.sh`) and use Keystone.
For security reasons, please put the file in a directory only you have access to and secure the file so that only you can read and execute it, because anyone who would be able to read or execute it would be able to use your credentials.


### Not currently done (and not working as is)

**After having checked that the service works correctly, remove the administrative token from the keystone.conf and also modify `/etc/keystone/keystone-paste.ini` to remove these two lines:**

    ```
    [filter:admin_token_auth]
    paste.filter_factory = keystone.middleware:AdminTokenAuthMiddleware.factory
    ```


