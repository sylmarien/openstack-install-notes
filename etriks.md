# eTRIKS specific settings

## Overview

This document tracks the various information specific to eTRIKS such as the access configuration and the details of the running services.

## Access configuration

### Stored SSH keys

(on the access VM: whose and why)

### Network settings

(IPs, ports, DNS, aliases, etc for external access)

### OpenStack tenants

(details of the existing tenant(s) and their rights/quotas)

## Virtual Machines details

(Running VMs, flavor and software details for each of them, "public" IPs + name resolution if applicable)

Test flavors for Bio (b) related VMs (flavor name, auto-generate ID, memory in MB, root disk in GB, #VCPUs):

         nova flavor-create b1.tiny      auto    1024    10      1
         nova flavor-create b1.standard  auto    2048    10      1
         nova flavor-create b1.medium    auto    4096    20      2
         nova flavor-create b1.2xmedium  auto    8192    40      4
         nova flavor-create b1.4xmedium  auto    16384   80      8
         nova flavor-create b1.large     auto    40960   100     10
         nova flavor-create b1.xlarge    auto    81920   200     20
         nova flavor-create b2.standard  auto    16180   10      1
         nova flavor-create b2.medium    auto    32358   20      2
         nova flavor-create b2.2xmedium  auto    64716   40      4
         nova flavor-create b2.4xmedium  auto    129434  80      8
         nova flavor-create b2.large     auto    161792  100     10
         nova flavor-create b2.xlarge    auto    315392  200     20
