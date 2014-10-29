# OpenStack install notes

## Summary

Install notes on the installation procedures of the different components of OpenStack in the context of the deployment at the University of Luxembourg.  
Note that this is **not** a user documentation, it is destined to system administrators that would have to operate on our deployment or to people who would like to replicate our deployment.
Each file contains notes regarding either  a specific component of OpenStack or a node of the Architecture.  
The OpenStackNotes.md file is a bit appart from the others, since it contains general informations on the project (notably the architecture).

***Important note: This repository is a Work In Progress. Therefore, some files may be listed below but not be actually present in the repository. They will be as soon as the deployment of the corresponding component attains a state from which reliable information can be taken.  
Moreover, the presence of a file does not mean that it is in a final/release state. Information may be incomplete or contain some errors and therefore may change in later versions.***

## Files description

#### [OpenStackNotes.md](https://github.com/sylmarien/openstack-install-notes/blob/master/OpenStackNotes.md "OpenStackNotes.md")

This file contains diverses notes and comments on the installation as a whole. Its main purpose is to describe the architecture and some specificity it could have.  
During the development period it also contains reminders on topics to dig further and reflexions I'm having that may (or may not) make it to the implementation/release.

#### Controller.md

This file contains notes on the Controller node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Controller node.

#### Compute.md

This file contains notes on the Compute node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Compute node.

#### Storage.md (Exists ? To be decided)

This file contains notes on the Storage node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Storage node.

#### [Keystone.md](https://github.com/sylmarien/openstack-install-notes/blob/master/keystone.md "Keystone.md")

This file contains notes on the install procedure for the Keystone module of OpenStack.