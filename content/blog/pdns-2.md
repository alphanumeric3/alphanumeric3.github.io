+++
title = 'PowerDNS part 2: getting libvirt to use it'
date = 2025-03-30T20:56:21+01:00
summary = "How to use dnsmasq's `server` option in a libvirt network"
+++

As I detailed [recently](/blog/pdns.md), I have authoritative DNS running inside a virtual network.

But that doesn't mean other virtual machines will suddenly start using it for `.an3`! Thankfully in libvirt, dnsmasq is used for for DHCP and DNS by default. dnsmasq can forward queries for certain domains to specific servers - such as `*.an3` to my PowerDNS server. This way, VMs can automatically use `.an3`!

dnsmasq's configuration is very easy, so I looked at `/var/lib/libvirt/dnsmasq/irl-lab.conf`:

```txt
##WARNING:  THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BE
##OVERWRITTEN AND LOST.  Changes to this configuration should be made using:
##    virsh net-edit irl-lab
## or other application using the libvirt API.
##
## dnsmasq conf file created by libvirt
```

Okay. Looks like I'll have to set it in libvirt, or my configuration will be removed.


I found that this is all I need to add to the network configuration (as it said, through `virsh net-edit` or an app like `virt-manager`):

```xml
<dns>
    <forwarder domain="an3" addr="192.168.122.2"/>
</dns>
```

After restarting the network (can be done with `virsh net-destroy irl-lab && virsh net-start irl-lab`) and starting a test VM, it seems to work:

```sh
localhost:~# ping ns1
PING ns1 (192.168.122.2): 56 data bytes
64 bytes from 192.168.122.2: seq=0 ttl=64 time=0.637 ms
```

## What's next?

Still a secondary. I haven't deviated completely from my plan, this is just necessary setup!
