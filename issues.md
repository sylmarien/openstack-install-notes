# Issues

## Overview

This file tracks issues encountered as well as the solution applied to solve them.

## Hardware issues

## Software issues

- Known issue with Horizon: When listing the instances, a message shows up stating "Error: Unable to connect to Neutron.". However, everything works without any problem and this message, or any related one, is nowhere to be found in any of the various services logs.
- Sometime the nova quota_usage for a tenant can be different from the actual usage of the tenant. This is usually due to OpenStack not handling correctly some cases when an error occurs. The solution in this case is to modify the database manually. In particular, to reset the nova quota_usages to 0 for a given user, on `pyro-database`:

        mysql -u root -p
        use nova;
        select * from quota_usages;
        update quota_usages set in_use='0' where project_id='<my project id>';
Where <my project id> is the id of the tenant for which you want to reset the nova quota usages.