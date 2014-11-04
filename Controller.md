# Controller node

## Overview
This file contains notes on the Controller node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Controller node.

**During development, all passwords are _kikoo_.**

## Basic environment

The network configuration for this node is the following:  
2 interfaces:

- eth0 : NAT
- eth1 : management network (on subnet 10.10.10.0/24)

**Configure networking**

1. Modify /etc/network/interfaces to configure eth1:

  ```
  # Management network interface
  auto eth1
  iface eth1 inet static
  address 10.10.10.11
  netmask 255.255.255.0
  # On VM, no gateway because it is set by the DHCP on the NAT interface
  # gateway 10.10.10.1
  ```
2. Modify /etc/hosts to configure the name resolution:

  ```
  # controller
  10.10.10.11       controller

  # network
  10.10.10.21       network

  # compute1
  10.10.10.31       compute1
  ```  
  Comment any other line.
3. Reboot to activate changes.

**Configure NTP**

Install the NTP service:  
  `apt-get install ntp`

**Basic prerequisites**

1. Install _python-software-properties_ package to ease repository management:  
  `apt-get install python-software-properties`
2. Enable the OpenStack repository (seems not necessary on Ubuntu 14.04 since Icehouse is the default version):  
  `add-apt-repository cloud-archive:icehouse`
3. Upgrade the packages on the system:  
  `apt-get update && apt-get dist-upgrade`

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
  3. MariaDB misplaces by default the mysqld.sock location, so modify /etc/mysql/my.cnf:
  
    ```
    [client]
    socket = /tmp/mysqld.sock
    ```
    ```
    [mysqld_safe]
    socket = /tmp/mysqld.sock
    ```
    ```
    [mysqld]
    socket = /tmp/mysqld.sock
    ```
4. Restart the database service:  
  `service mysql restart`
5. Secure the database service:  
  `mysql_secure_installation`

**Messaging server**  
We use RabbitMQ as message broker service.

1. Install the package:  
  `apt-get install rabbitmq-server`
2. Configure the message broker service (replace the guest password):  
  `rabbitmqctl change_password guest RABBIT_PASS`  
  Replacing _RABBIT_PASS_ with a suitable password.

**Note:** During development, we use the guest account that is automatically created by RabbitMQ, but for production, we should create a unique user with suitable password.  
Then, we would have to configure the _rabbit_userid_ and _rabbit_password_ keys accordingly in the configuration files of each OpenStack service that use the message broker.
