# Hardware

## Overview

This document details the hardware of the various servers of the OpenStack infrastructure.

## Controller nodes and storage node

DELL R620
- 2x Xeon E5-2630Lv2 (2.4GHz, 6 cores, 15Mo cache)
- 32 GB RAM
- 1TB HDD (2x1TB RAID-1)
- 4x 1GbE, 2x 10GbE, 1x IB QDR (Mellanox ConnectX-3)


## Compute nodes

DELL R620
- 2x Xeon E5-2660v2 (2.2GHz, 10 cores, 25Mo cache)
- 320 GB RAM
- 1TB HDD (2x1TB RAID-1) + 2.4 TB (4x1.2TB RAID10)
- 4x 1GbE, 2x 10GbE, 1x IB QDR (Mellanox ConnectX-3)

## Storage backend

NetApp 5400 appliance, FC/iSCSI
60 x 4TB SAS : 240TB (raw)
