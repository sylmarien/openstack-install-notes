# OpenStack install notes

## Summary

Install notes on the installation procedures of the different components of OpenStack in the context of the deployment at the University of Luxembourg.  
Note that this is **not** a user documentation, it is destined to system administrators that would have to operate on our deployment or to people who would like to replicate our deployment.
Each file contains notes regarding either  a specific component of OpenStack or a node of the Architecture.  
The OpenStackNotes.md file is a bit appart from the others, since it contains general informations on the project (notably the architecture).

## Files description

#### [Overview.md](Overview.md)

Overview of the installation.

### Nodes and Virtual Machines

#### [Controller.md](Controller.md)

This file contains notes on the Controller node of the architecture.

##### [HorizonVM.md](HorizonVM.md)

This file contains notes on the Horizon VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Horizon VM.

##### [CoreVM.md](CoreVM.md)

This file contains notes on the Horizon VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Core VM.

##### [StoreVM.md](StoreVM.md)

This file contains notes on the Horizon VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Store VM.

##### [DatabaseVM.md](DatabaseVM.md)

This file contains notes on the Horizon VM of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Database VM.

#### [Compute.md](Compute.md)

This file contains notes on the Compute node of the architecture. Among others, the purpose of this document is to take track of the OpenStack components, and each modules of these components specifically, that are installed on the Compute node.

### Services

#### [Keystone.md](Keystone.md)

This file contains notes on the install procedure for the Keystone module of OpenStack.

#### [Glance.md](Glance.md)

This file contains notes on the install procedure for the Glance module of OpenStack.

#### [Nova.md](Nova.md)

This file contains notes on the install procedure for the Nova module of OpenStack.

#### [Neutron.md](Neutron.md)

This file contains notes on the install procedure for the Neutron module of OpenStack.

#### [Horizon.md](Horizon.md)

This file contains notes on the install procedure for the Horizon module of OpenStack.

#### [Cinder.md](Cinder.md)

This file contains notes on the install procedure for the Cinder module of OpenStack.

### LCSB - eTRIKS specific

#### [eTRIKS specific settings](etriks.md)

This file contains configuration details specific to eTRIKS.

### Issues

#### [Issues](issues.md)

This file tracks issues encountered as well as the solution applied to solve them.
