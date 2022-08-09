---
title: pfSense Routing with SR-IOV and Proxmox
slug: pfsense-sriov
date_published: 2021-07-04T18:01:31.000Z
date_updated: 2021-07-04T18:01:31.000Z
tags: Linux, Networking, Virtualization
excerpt: Setting up SR-IOV and using VF's on a Proxmox Host
---

### Introduction

Single Root I/O Virtualization (SR-IOV) is a technology that was developed in order to split up physical PCI devices into multiple separate PCI devices. In a virtualization stack, it allows us to remove the VMM layer of virtualization and address hardware directly. [Scott's Weblog explains this in further detail. ](https://blog.scottlowe.org/2009/12/02/what-is-sr-iov/)
![](__GHOST_URL__/content/images/2021/07/single_port_nic.png)Source: https://doc.dpdk.org/guides/nics/intel_vf.html
One of the most common examples of this technology is present in NIC's. Intel has put this technology in lots of there equipment, but one of the most common chip-sets for Homelabbers is the [i350 chipset](https://ark.intel.com/content/www/us/en/ark/products/84805/intel-ethernet-server-adapter-i350-t4v2.html). After doing my research online, I came to the conclusion that this was the most appropriate chip-set for pfSense. My specific variant of this chip-set is the T4V2 version. A 4 port gigabit NIC going for about £50 on [eBay](https://www.ebay.co.uk/sch/i.html?_nkw=i350-t4&amp;_sacat=0). 

SR-IOV allows me to split up my WAN connection into "Virtual Functions" which reduces the overhead on the Hypervisor due to not having to process traffic through the virtual switch. This means that latency-sensitive applications could benefit greatly alongside more CPU cycles being available to the host.

---

## Creating Virtual Functions

I didn't find much documentation on this available. However, after a couple of bug reports and Proxmox threads I had managed to piece it together. Here are the requirements:

- SR-IOV enabled in the BIOS. I found my NIC has a dedicated section with the amount of VF's I could define.
- Intel VT-D / AMD-Vi enabled
- Access to GRUB's configuration file
- Access to module configurations

### Editing Grub

On Proxmox, the path for the grub configuration file can be found at /etc/default/grub. Under the `GRUB_CMDLINE_LINUX_DEFAULT` parameters I appended the following: 

`amd_iommu=on iommu=pt pci=assign-busses` (Replace AMD with Intel for Intel CPU's)

These flags will split the PCI devices up into IOMMU groups and assign numbers from the kernel as opposed to the firmware. I found that Virtual Functions could not be created without the assign-busses flag. [This thread](https://forum.proxmox.com/threads/enabling-sr-iov-for-intel-nic-x550-t2-on-proxmox-6.56677/) is how I found out. 

**************************************************************************

**Warning: pci=assign-busses will rename any interfaces you have, please ensure you have console access to amend your network conneciton.**

**************************************************************************
![](__GHOST_URL__/content/images/2021/07/grub.png)My Grub configuration with the changes highlighted in red. 
After adding this to my grub config file, I ran `update-grub`. My previous PCI pass through devices had changed identifiers so I made sure to update these also,

### Enabling the VFIO modules

We need to add the VFIO modules which will allow us to use the VF's as a stub so that they do not act as network interfaces on the host. This is a common technique to keep a PCI device free from the host in order to pass through to a guest machine.

Put `vfio`, `vfio_iommu_type1`, `vfio_pci` and `vfio_virqfd` into `/etc/modules`
![](__GHOST_URL__/content/images/2021/07/modules.png)A preview of my modules
After this, we can update our initramfs image with `update-initramfs -u -k all` and reboot the host with `shutdown -r now`. 

---

## Enabling the functions 

Credit to Sandbo on the Proxmox Forum for a [detailed guide](https://forum.proxmox.com/threads/enabling-sr-iov-for-intel-nic-x550-t2-on-proxmox-6.56677/) on this. These gave me the right pointers in order to enable it for my system.

To begin with, we will need to get the device name of our NIC. This can be found with the command `ls /sys/class/net` as it will list all available interfaces. Hopefully this is enough for you, but the command `ip address` or `ip link` may also help to indicate which NIC is what. For me, my i350 was under the name enp6s0f0.
![](__GHOST_URL__/content/images/2021/07/enp6s0f0.png)My interface with the `ip link` command
Now that we have our NIC interface, we can finally execute the following. Make sure to replace N with the amount of interfaces you want and <IF> with the interface name we found above.

`echo N > /sys/class/net/<IF>/device/sriov_numvfs`

In my case, the command is:

`echo 6 > /sys/class/net/enp6s0f0/device/sriov_numvfs`

This command will hopefully succeed, creating as many virtual interfaces as you'd like. A great way to see these interfaces is by using the `ip link` comamnd:
![](__GHOST_URL__/content/images/2021/07/iplinksriov.png)The output of ip link showing my VF's.
---

## Making the functions persistent

[Sandbo made a perfect systemd daemon for this](https://forum.proxmox.com/threads/enabling-sr-iov-for-intel-nic-x550-t2-on-proxmox-6.56677/). Credits to him.

Create the file `/etc/systemd/system/sriov.service` and input the following, amending to follow the previously used command:

[Unit]

Description=Script to enable SR-IOV on boot

[Service]

Type=oneshot

ExecStart=/usr/bin/bash -c '/usr/bin/echo N > /sys/class/net//device/sriov_numvfs'

[Install]

WantedBy=multi-user.target

**If you intend to use pfSense with a virtual function, you will need to manually set the MAC address of one due to broken support in FreeBSD:**

`ExecStart=/usr/bin/bash -c '/usr/bin/ip link set <IF> vf <NUM> mac 02:00:00:00:00:00'`

This script will run while the server is starting up, alleviating the need for manual VF creation. 

---

## Passing to a Virtual Machine

Now that our Virtual Functions have been stubbed and are working correctly, we can pass them through to our guest machines. These machines will more than likely have the driver already, however BSD guests may face issues. I found that OpenBSD did not support VF's.

On the web interface for Proxmox, select a virtual machine on the sidebar and click on the hardware section. Next, click:

**`Add > PCI Device > Virtual Function X`**
![](__GHOST_URL__/content/images/2021/07/proxmox1.png)A list of PCI devices we can pass through. Select a Virtual Function
You may notice on the side each VF has a different IOMMU group.  This can be a easy way to note down which guest uses what, or alternatively the PCI ID of the VF.

That's all. It's a relatively simple process, however getting here may have been a few headaches. In the future, all you need to do is just assign PCI devices!

---

## Monitoring the interface

### On the guest

Once you start the guest, it will more then likely have already configured itself and be online within your network. In my case, it has received an IP from DHCP:
![](__GHOST_URL__/content/images/2021/07/rockysrv.png)The VF is represented by ens16
If we view information about the PCI device (your ID will be different):
![](__GHOST_URL__/content/images/2021/07/lspcirocky.png)Information on the ethernet via passthrough
We can see that it is detected and has a driver in use. 

### On the host

If we view out interfaces with the `ip link` command, we can see that there has been a change in the VF MAC addresses:
![](__GHOST_URL__/content/images/2021/07/sriovonhost.png)VF 3 has a random MAC address
This shows us that the interface has been successfully configured for pass through as the driver as given it a valid MAC. However, this may not always be the case on UNIX derived guests and may need manually setting. This can be done with the following command:

`ip link set <interface> vf <number> mac 02:00:00:00:00:00`

We can also view the interface with commands such as `nload` or `iftop` by finding the interface name associated with the VF. In my case, this VF carried the name `fwln101i0`
![](__GHOST_URL__/content/images/2021/07/nload.png)The interface on nload
---

## pfSense quirks

pfSense can work lovely with Virtual Functions, however, as mentioned above BSD guests do not tend to treat these interfaces well. In my experience, they either do not assign MAC addresses automatically or they do not have the drivers at all. 

To fix this, I amended the following to my systemd script:

`ExecStart=/usr/bin/bash -c '/usr/bin/ip link set enp6s0f0 vf 0 mac 02:00:00:00:00:00'`

This will assign the MAC address 02:00:00:00:00 to the first function on system boot. This function will then be passed through to my pfSense guest.

After this, the guest should recognise the function and work correctly. I have not met any unexpected issues by using this MAC address fix however I cannot guarantee the same for you.
![](__GHOST_URL__/content/images/2021/07/pfsenseinterfaces.png)Shows up under the igb0 interface
---

## Final Thoughts

As a consumer, this technology is not something which I can benefit from as much as someone from the enterprise may. None the less, exploring this route of Virtualization is opening doors for further projects. Youtuber [Level1Techs](https://www.youtube.com/channel/UCOWcZ6Wicl-1N34H0zZe38w) has explained in great detail about how SR-IOV can open up single GPU virtualization for multiple guests which is exponentially more exciting. 

If you have a chipset laying about which supports this technology, it's worth a play around to grasp the potential of it. 
