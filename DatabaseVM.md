# Database VM

## Overview
This file contains notes on the Database VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Database VM.

**During development, all passwords are _localadmin_.**

## Basic configuration

Basic configuration is described in the [Controller.md](https://github.com/sylmarien/openstack-install-notes/blob/master/Controller.md) file. It is the shared configuration between all the VMs of the controller node.

**Database**  
We use MariaDB to manage the SQL database.

1. Install the packages:  
  `apt-get install mariadb-server python-mysqldb`
2. Choose a suitable password for the database root account.
3. Modify /etc/mysql/my.cnf to:
  1. Set the bind-address key to the management IP address of the controller node to enable access by other nodes va the management network:
  
    ```
    [mysqld]
    ...
    bind-address = 10.10.10.11
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