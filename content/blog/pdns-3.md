+++
title = 'PowerDNS part 3: setting up the secondary'
date = 2025-05-27T22:25:39+01:00
draft = false
summary = "Manually setting up a second nameserver"
+++

It has been a short while since I last said I'd add a secondary server to my virtual PowerDNS setup.
Time to do it :)

## Possible options

The documentation explains three ways to set up secondaries:

- Native replication - the database backend (e.g. MySQL) handles replication. This is the default.

- Primary/secondary - servers are explicitly set up to be primaries and secondaries. Each zone has to be added
to each server.

- Autoprimary/autosecondary - secondaries set up any domain they're notified about, as long as the notification comes
from a configured autoprimary.

Native replication isn't an option - I use SQLite which has no replication features. And autoprimary is a very nice sounding feature, but I'd like to try
manually first. So a manual primary/secondary setup it is.

## Setup

First turn on the primary feature in `pdns.conf`:

```txt
primary=yes 
```

Then change the zone (`an3`) to be a primary zone on that server:

```sh
pdns:~$ doas pdnsutil set-kind an3 primary # The docs say 'MASTER'. But 'PRIMARY', 'primary' and 'master' also work
pdns:~$ doas pdnsutil list-all-zones primary
an3
```

And before starting on the secondary, add proper records (`A` and `NS`) to make sure the primary knows about the secondary:

```sh
pdns:~$ doas pdnsutil add-record an3 ns2 A 192.168.122.3
New rrset:
ns2.an3. 3600 IN A 192.168.122.3
pdns:~$ doas pdnsutil add-record an3 '@' NS ns2.an3
New rrset:
an3. 3600 IN NS ns1.an3
an3. 3600 IN NS ns2.an3
```

On the secondary server add `secondary=yes` to `pdns.conf`.
Then restart PowerDNS and add the zone with `pdnsutil create-secondary-zone <zone> <primary IP>`:

```sh
pdns2:~$ doas nano /etc/pdns/pdns.conf
pdns2:~$ doas rc-service pdns restart
 * Stopping PowerDNS (default) ...                                                     [ ok ]
 * Starting PowerDNS (default) ...                                                     [ ok ]
pdns2:~$ doas pdnsutil create-secondary-zone an3 192.168.122.2
Creating secondary zone 'an3', with primaries '192.168.122.2'
```

Now I have two happy DNS servers, right?

```sh
$ dig ns2.an3 @192.168.122.3 +noall +comments
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 6943
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
```

I don't. The status `REFUSED` means my secondary isn't even serving the zone.

### Fixing failed AXFRs (zone transfers)

The first fix to try is notifying the secondary with `pdns_control notify <zone>` or `pdns_control notify-host <zone> <secondary IP>`, in case it
hasn't attempted to fetch anything yet. Or run `pdns_control retrieve <zone>` on the secondary. But none of these worked.

Let's check the primary's logs:

```txt
Jun  4 17:41:35 pdns daemon.warn pdns[2679]: AXFR-out zone 'an3', client '192.168.122.3:36603', transfer initiated
Jun  4 17:41:35 pdns daemon.warn pdns[2679]: AXFR-out zone 'an3', client '192.168.122.3:36603', denied: client IP has no permission
Jun  4 17:41:35 pdns daemon.warn pdns[2679]: AXFR-out zone 'an3', client '192.168.122.3:36603', failed: client may not request AXFR
```

My mistake was not allowing zone transfers from other servers on the primary. The fix is to put this in `pdns.conf` and restart PowerDNS:

```txt
allow-axfr-ips=192.168.122.3
```

Then notify the secondary again or wait for it to retry, and everything should work.

```sh
$ dig an3 any @192.168.122.3 +nocomments

; <<>> DiG 9.18.33 <<>> an3 any @192.168.122.3 +nocomments
;; global options: +cmd
;an3.                           IN      ANY
an3.                    3600    IN      NS      ns1.an3.
an3.                    3600    IN      NS      ns2.an3.
an3.                    3600    IN      SOA     ns1.an3. hostmaster.an3. 3 10800 3600 604800 3600
ns2.an3.                3600    IN      A       192.168.122.3
ns1.an3.                3600    IN      A       192.168.122.2
;; Query time: 2 msec
;; SERVER: 192.168.122.3#53(192.168.122.3) (TCP)
;; WHEN: Wed Jun 04 19:08:46 BST 2025
;; MSG SIZE  rcvd: 147
```

Great!

## Making dnsmasq use the server

I use libvirt for virtualisation, which uses dnsmasq as a DHCP and DNS server. dnsmasq is only set to use the _primary_ for domains on
`.an3`. So if the primary goes down for some reason, DNS will break for the whole virtual network:

```sh
$ sudo virsh destroy pdns
Domain 'pdns' destroyed
$ dig version.bind ch txt @192.168.122.1 +short
"dnsmasq-2.90"
$ dig ns1.an3 @192.168.122.1 +short
;; communications error to 192.168.122.1#53: timed out
;; communications error to 192.168.122.1#53: timed out
;; communications error to 192.168.122.1#53: timed out

; <<>> DiG 9.18.33 <<>> ns1.an3 @192.168.122.1 +short
;; global options: +cmd
;; no servers could be reached
```

That doesn't scream redundancy or reliability, which is the point of having more than one server.

Fortunately, it's easy to add another server in the libvirt network configuration:

```xml
<dns>
  <forwarder domain="an3" addr="192.168.122.2"/>
  <forwarder domain="an3" addr="192.168.122.3"/>
</dns>
```

After restarting the network, DNS resolution works fine:

```sh
$ dig ns1.an3 @192.168.122.1 +short
;; communications error to 192.168.122.1#53: timed out
192.168.122.2
$ dig ns1.an3 @192.168.122.1 +short
192.168.122.2
```

Although there is a timeout on the very first DNS query for `an3` (since I took the primary down), queries after work fine.

## Next steps

After this, I'll start automating setup of PowerDNS itself and _also_ automate making DNS records when I run an Ansible playbook
(for example, the playbook for `service` would make `A` records for `service.an3`).

I could also try VRRP with Keepalived, so multiple servers can use one IP address. It's overkill here but still fun.

Thanks for reading!
