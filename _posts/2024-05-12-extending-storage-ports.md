---
layout: post
title:  "Extending a motherboard's storage ports"
date:   2024-05-12 00:00:00 +0100
categories: hardware storage
---

_DISCLAIMER: the information here might not be completely correct. Please, if you find any error open an [issue](https://github.com/mkmenta/mkmenta.github.io/issues). Thanks!_

These notes are about adding more storage drive ports to a motherboard. It can be useful for building a NAS to serve data for AI training or other applications. However, the first step should be figuring out your speed, capacity, and data integrity requirements. For example, I have found that training a computer vision deep learning model in an RTX 3090 in bfloat16 can take as low as 20MB/s random read to have the GPU at the maximum. If the model was trained with pre-processed low-resolution images/text, had more parameters to compute, or was in a higher precision data type, probably the speed requirements could have been even lower.

## The basics
### Connection types
These are the current connection types for storage that I have found. In **bold** are the most common ones.

| Connection        | Controller           | Max speed | Description/Notes                                                                                                                    |
|-------------------|----------------------|----------:|--------------------------------------------------------------------------------------------------------------------------------------|
| **SATA**          | **SATA III (3.x)**   |  600 MB/s | Very common. But the SATA controller is a bottleneck in speed for new drives and its connector is _unnecessarily_ big.               |
| mSATA (mini-SATA) | SATA III (3.x)       |  600 MB/s | Deprecated by M.2 NVMe                                                                                                               |
| M.2 (B+M Key)     | SATA III (3.x)       |  600 MB/s | Deprecated by M.2 NVMe                                                                                                               |
| **SAS**           | **SAS-5**            |  5.6 GB/s | SAS is an upgrade of SATA common in enterprise systems. It is   retrocompatible with SATA connectors.                                |
| U.2               | NVMe via PCIe x4     |   15 TB/s | Possibly deprecated by M.2 connector, which is much smaller. It tried to   be an upgrade for SATA.                                   |
| **M.2 (M Key)**   | **NVMe via PCIe x4** |   15 TB/s | The max speed is taken from a PCIe x4 5.0 connection and shows how big is   the bandwith. It should even be increased in the future. |

Note that previous versions of SATA, SAS, and PCIe x4 exist that provide slower speeds. You can find their bandwidths easily on Wikipedia.

### Disk types
- Hard Disk Drives (HDD) can provide a lot of GB per € but their speed is limited: while they can achieve up to ~150MB/s if read/written sequentially*, when accessing them randomly they drop fast to ~2MB/s.
- Solid-state drives (SSD) can provide huge random read/write speeds easily achieving 1-2GB/s. But they often are limited by the connection type (unless it is M.2 NVMe, see the Connection types section).

*NOTE: In artificial intelligence (AI) training, datasets subdivided (sharded) in `.tar` files exploit the sequential read speed of HDDs to get high-performance cheaper (check [webdataset](https://github.com/webdataset/webdataset?tab=readme-ov-file#the-webdataset-format)).

## Adding M.2 NVMe ports
M.2 (M Key) NVMe ports use, in essence, a PCIe x4 connection (four lanes). This means that they can be easily connected with a PCIe x4 to M.2 adapter.

However, if you want to use a single PCIe connector to connect multiple M.2 NVMe drives (e.g. a PCIe x16 to connect four M.2 NVMe drives), things get more complicated. In this case, there are two options.

### Option 1
If the motherboard supports PCIe bifurcation it should be easy to do with a cheap adapter (and the correct BIOS configuration). PCIe bifurcation means that the motherboard can use a PCIe x16 connection as four x4x4x4x4 connections, for example. It is important to note that it should support the specific bifurcation you need: you can't connect four M.2 NVMe drives with an adapter if it only supports x16 -> x8x8. If the motherboard does not support bifurcation, it will only recognize one of the devices (it will basically use the x8 or x16 as an x4). 

Knowing if the motherboard supports bifurcation is often not easy (especially if it is not an enterprise-grade motherboard). [Here](https://www.reddit.com/r/Amd/comments/14bnqh3/guide_about_how_to_check_pcie_bifurcation_support/) you can find a method (is not as hard as it seems at first sight).

### Option 2
If the motherboard does not support bifurcation, the adapter should. But these adapters are way more expensive (at this point you might consider buying a used/refurbished enterprise server). [Here](https://forums.servethehome.com/index.php?threads/multi-nvme-m-2-u-2-adapters-that-do-not-require-bifurcation.31172/) is some more info about it.

## Adding SATA ports
### What NOT to do: SATA splitters
There are cheap (10-20€) SATA splitters/controllers/multipliers that go into PCIe. However, they are not recommended for various reasons (see [here](https://www.reddit.com/r/truenas/comments/10qt9fw/hba_vs_pci_based_sata_expansion_for_truenas_yes_i/), [here](https://www.reddit.com/r/zfs/comments/9hs0ij/why_get_a_hba_controller_when_the_interal_sata/), and [here](https://www.truenas.com/community/resources/multiply-your-problems-with-sata-port-multipliers-and-cheap-sata-controllers.177/)):
- They often have compatibility issues.
- They are often not prepared to provide access to multiple disks in parallel (and RAID systems normally do this).
- They have cheap components that can break under high loads (like a RAID system, as said above).

### What to do: HBA/RAID SAS cards
The enterprise-grade solution that can be below 50€ (used) is a Host Bus Adapter (HBA) card with SAS ports (usually Mini SAS a.k.a SFF-8087). Then you can connect several SATA drives to each of these ports just with a cable, e.g. SFF-8087 to four SATA ports cable. As simple as that.

![Example of a HBA card with SAS ports](/static/images/hba-sas-card.jpg)

Even more, these SAS ports often have very high throughputs. So, in the case of needing more ports, you can add a **SAS expander** (splits a SAS port into several SAS ports) that also connects to PCIe, and using the same cables you can easily scale to hundreds of SATA ports.

More information about these HBA cards:
- The most common ones are from the LSI brand (now bought by Broadcom), so they can be found searching for example "LSI 9211-8i". But there are also cards from Dell, Intel, etc. (look for "Dell HBA SAS card").
- For the LSI cards, the `-8i`, `-16e` etc. at the end of the name refers to the number of ports. `i` means that the ports are internal (probably what you look for), and `e` means external. The number means the amount of SATA ports that are provided (directly using SFF-8087 to 4xSATA cables). For example: `-8i` means that it has two internal SAS SFF-8087 ports that can provide 8 SATA ports.
- These cards can do hardware RAID but usually, you will want to do the RAID using software (like TrueNAS). For this reason, the card will need to be flashed into IT mode. So that it just passes the drives to the system. Otherwise, you will need to flash the card yourself.
