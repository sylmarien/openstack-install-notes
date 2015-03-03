# Horizon

## Overview

This file contains notes on the install procedure for the Keystone module of OpenStack.
Important note: This document currently describe a very generic configuration, it should be adapted to describe the configuration that we will actually deploy. (Will be done as soon as we know more about this configuration)

## Install on the Horizon VM

As of now, I follow the instructions of the documentation at this page:  
http://docs.openstack.org/icehouse/install-guide/install/apt/content/install_dashboard.html

#### List of installed modules
- apache2
- memcached
- libapache2-mod-wsgi
- openstack-dashboard

#### Install steps
1. Install the packages:  
  `apt-get install apache2 memcached libapache2-mod-wsgi openstack-dashboard`
2. Remove _openstack-dashboard-ubuntu-theme_ package, because this theme prevents translations, several menus as well as the network map from rendering correctly.  
  `apt-get remove --purge openstack-dashboard-ubuntu-theme`
3. Modify `/etc/openstack-dashboard/local_settings.py` to:
  1. Define the hostname of the Identity service (in this example, controller):  
    `OPENSTACK_HOST = "core"`
  2. Set the addresses from which you want to enable the access to the dashboard:  
    `ALLOWED_HOSTS = ['*']`  
    In a production environment, it would be intersting to allow only IP from the University maybe ? (what about home working ?). Note: if you want to specify multiple location, separate them with a `,` (example: `['localhost','workstation']` )
  3. Modify the value of `CACHES['default']['LOCATION']` to match the ones set in /etc/memcached.conf:
    
    ```
    CACHES = {
    'default': {
    'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
    'LOCATION' : '127.0.0.1:11211'
    }
    }
    ```  
    **Notes:**  
    - The address and port must match the ones set in /etc/memcached.conf. If you change the memcached settings, you must restart the Apache web server for the changes to take effect.
    - You can use options other than memcached option for session storage. Set the session back-end through the SESSION_ENGINE option.
  4. (Optionnal) Change the timezone (DON'T: currently, only working with UTC so far):  
    `TIME_ZONE = "UTC"`  
    To find you timezone, follow this [link](http://en.wikipedia.org/wiki/List_of_tz_database_time_zones).
4. Restart the services:
  
  ```
  service apache2 restart
  service memcached restart
  ```
5. You know can access the dashboard at http://horizon/horizon (from a machine that can resolve the _horizon_ name) and login with any of the users you created in the Identity service.
