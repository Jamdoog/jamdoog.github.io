---
title: QEMU Guest Agent on pfSense
slug: guest-agent-on-pfsense
date_published: 2022-05-26T06:45:20.000Z
date_updated: 2022-05-26T06:47:22.000Z
tags: [Networking, Linux, UNIX, FreeBSD, Proxmox, Virtual Machine]
excerpt: A quick look at installing the qemu-guest-agent package on pfSense (FreeBSD).
---

![](/assets/images/guest-agent-on-pfsense/github.png)

A piece of software I always found fascinating about hypervisors was the concept of a guest agent. The first example I recall was file sharing from a VMWare Host to Guest on Workstation which seemed genius, removing the need for either USB or Network approaches to traditional file sharing.

Forward to now, my virtualization needs have since expanded to a completely seperate home server with Proxmox as a chosen Hypervisor. Proxmox is a **Free and Open Source** software suite for managing KVM/QEMU virtualization in addition to legacy LXC containers.

This is great and the qemu-guest-agent project is brilliant, however it has always posed 1 problem: A lacking UNIX port. [It was to great suprise to recently learn that their has been a port development over the past few years that has reached maturity in the latest FreeBSD version. ](https://github.com/aborche/qemu-guest-agent)

This finally meant all of my guest's had proper shutdown capability, no more hanging on the occasional host reboot. 🥳

---

## Install

Installation of the agent, while not point and click, is relatively easy.

[*I should link this thread for reference to the instructions.*](https://forum.netgate.com/topic/162083/pfsense-vm-on-proxmox-qemu-agent-installation)*[Someone also made a auto-installer script.](https://github.com/Weehooey/pfSense-scripts/blob/main/install-qemu-guest-agent.sh)*

To begin with, open up your console. **We need to access the shell so type in 8**.
![](/assets/images/guest-agent-on-pfsense/console.png)
Then to install the QEMU Guest Agent package, it is 1 simple command:

`pkg install qemu-guest-agent`. 

After this, rc.local needs to be modified:

`echo 'qemu_guest_agent_enable="YES"' >> /etc/rc.local.conf`

`echo 'qemu_guest_agent_flags="-d -v -l /var/log/qemu-ga.log"' >> /etc/rc.local.conf`

Finally, we need to start the service on boot:

`echo "#!/bin/sh" >> /usr/local/etc/rc.d/qemu-agent.sh`

`echo "sleep 3" >> /usr/local/etc/rc.d/qemu-agent.sh`

`echo "service qemu-guest-agent-start" >> /usr/local/etc/rc.d/qemu-agent.sh`
![](/assets/images/guest-agent-on-pfsense/script.png)
Finally, ensure the agent is enabled on Proxmox itself.
![](/assets/images/guest-agent-on-pfsense/enable_guest.png)
After this, you will have to hard stop and start the VM. If you are like me and rely on it for a network connection, open up a root shell on the host and type:

`qm stop <vmid>; qm start <vmid>`. This will stop and start it, regaining network connectivity when it's up again. 

---

## Result!

The open source community has blessed us one again. 

The easiest way of testing if the agent is functional is by viewing the summary of the VM. It should display all of the IP's of the VM.
![](/assets/images/guest-agent-on-pfsense/ip_show.png)
