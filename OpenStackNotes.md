# OpenStack notes

### About install method:

- Stackforge seems deprecated (and not updated for months), so we should avoid this method.
- The puppetlabs-openstack module seems to be maintained but isn't exactly up-to-date (not tracking Juno yet).
- Since OpenStack is packaged on Ubuntu (and Juno is already available by this mean), I recommend to the package repository as the source for the installation. This require to write our own Puppet manifests to automatize the retrieval of the modules (with maybe a differenciation based on the node type, see below for more information).

### Architecture (DRAFT):
Services Virtualised High Availability architecture.

**2 Controller nodes** with (one VM per component on each node):

- Neutron  
- Horizon  
- Keystone  
- MySQL or PostgreSQL  
- RabbitMQ  
- Nova Cloud Controller  
- (More ? Cinder ? Glance ? rsyslog ?)

**4 Compute nodes** with:

- Hypervisor (KVM, certainly)  
- Nova (exact modules to be defined)  
- Neutron (some modules at least certainly)

Additional notes:

- What specific modules from these components have to be installed there ?    
- Storage on a separate machine or on the Controller nodes ? (Cinder, Glance)
- **Everything run on Ubuntu 14.04.1 Server**  
- Virtual networks and networks have yet to be defined.
- Base idea for the architecture: http://www.ubuntu.com/cloud/openstack/reference-architecture

![Alt OpenStack architecture](http://assets.ubuntu.com/sites/ubuntu/1211/u/img/cloud/ubuntu-openstack/reference-architecture/image-servicesvmhigh-medium.png)

### Diverse notes
Hypervisor : KVM  
iSCSI ? (Is it the additional component in the servers ? How to use it ? (finding in the doc))  
RabbitMQ needs the line "127.0.0.1  ubuntu" in /etc/hosts to work properly even though the doc ask us to comment it. Would it be a problem later to let that uncommented ?  
MariaDB misplaces by default the mysqld.sock location, so change the "sock" lines (in /etc/mysql/my.conf) to /tmp/mysqld.sock
