# Database VM

## Overview
This file contains notes on the Database VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Database VM.

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
            address 10.89.201.4
            netmask 255.255.0.0
            broadcast 10.89.255.255
            gateway 10.89.0.1
            dns-nameservers 10.28.0.4 10.28.0.5
            dns-search uni.lux
        
        auto eth1
        iface eth1 inet static
            address 10.89.211.4
            netmask 255.255.0.0
            broadcast 10.89.255.255

**Database**  
We use MariaDB to manage the SQL database.

1. Install the packages:  
  `apt-get install mariadb-server python-mysqldb`
2. Choose a suitable password for the database root account.
3. Modify `/etc/mysql/my.cnf` to:
  1. Set the bind-address key to the management IP address of the controller node to enable access by other nodes va the management network:
  
    ```
    [mysqld]
    ...
    bind-address = 10.89.201.4
    ```
  2. Set the following keys to enable useful options and the UTF-8 character set:
  
    ```
    [mysqld]
    ...
    default-storage-engine = innodb
    innodb_file_per_table
    collation-server = utf8_general_ci
    init-connect = 'SET NAMES utf8'
    character-set-server = utf8
    ```
4. Restart the database service:  
  `service mysql restart`
5. Secure the database service:  
  `mysql_secure_installation`  
  Make sure to deactivate the remote root access.

The various databases will be created when needed during the various steps of the OpenStack configuration. (You will have to create it with the root user, so directly from the VM)