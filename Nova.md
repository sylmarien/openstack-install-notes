# Nova

## Overview

This file contains notes on the install procedure for the Nova module of OpenStack.  
***Important note: This document currently describe a very generic configuration, it should be adapted to describe the configuration that we will actually deploy. (Will be done as soon as we know more about this configuration)***

## Install on the nodes

As of now, I follow the instructions of the documentation at this page:  
[Nova Install in the OpenStack documentation](http://docs.openstack.org/juno/install-guide/install/apt/content/ch_nova.html)

#### List of installed modules by node
**Controller node:**

- nova-api
- nova-cert
- nova-conductor
- nova-consoleauth
- nova-novncproxy
- nova-scheduler
- python-novaclient

**Compute node:**

- nova-compute
- sysfsutils

