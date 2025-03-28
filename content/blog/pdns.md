+++
title = 'Trying out PowerDNS'
date = 2025-03-19T12:19:00Z
draft = false
summary = 'An authoritative DNS server'
+++

In this post, I'm going to set up [PowerDNS Authoritative Server](https://doc.powerdns.com/authoritative/index.html)
on [Alpine Linux](https://alpinelinux.org/) for my virtual lab. Alpine Linux is a lightweight Linux distribution,
and PowerDNS is a widely known DNS server software.

It'll mainly be used to resolve `.an3` domains for the virtual network.

## Setup

First I need to install PowerDNS (do not forget `pdns-doc`, otherwise you won't have database schemas):

```sh
doas apk update
doas apk upgrade
doas apk add pdns pdns-doc
```

I'm using SQLite3 to store my data, so I need to initialise the database and make sure PowerDNS can access it:

```sh
doas mkdir -p /var/lib/powerdns
doas sqlite3 /var/lib/powerdns/pdns.sqlite3 < /usr/share/doc/pdns/schema.sqlite3.sql
doas chown -R pdns:pdns /var/lib/powerdns
```

And before starting the server, it needs to be configured in `/etc/pdns/pdns.conf` to use the database:

```ini
launch=gsqlite3
gsqlite3-database=/var/lib/powerdns/pdns.sqlite3
```

```sh
doas rc-service pdns start
```

To test if it works, try making a query. It should respond with `REFUSED` since there are no zones to serve yet.

```sh
$ dig +noall +comments @192.168.122.95
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 3113
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
```

You can also check the version over DNS!

```sh
$ dig +short version.bind ch txt @192.168.122.95
"PowerDNS Authoritative Server 4.9.2 (built Oct  3 2024 12:46:06 by buildozer@localhost)"
```

## My first zone

Time to create a zone. On the server, I used `pdnsutil` to create one named after me - alphanumeric3, with `ns1` as the primary.

```sh
pdns:~$ doas pdnsutil create-zone an3 ns1.an3
Creating empty zone 'an3'
Also adding one NS record
pdns:~$ doas pdnsutil list-zone an3
$ORIGIN .
an3     3600    IN      NS      ns1.an3
an3     3600    IN      SOA     a.misconfigured.dns.server.invalid hostmaster.an3 0 10800 3600 604800 3600
```

(**Note:** using made-up TLDs on your network runs the risk of ICANN _actually_ allowing said TLD to be registered.
If you want to do this full-time on your LAN, with the guaranteee of no conflicts, consider buying a domain or using a reserved name like [.home.arpa](https://datatracker.ietf.org/doc/html/rfc8375))

The zone is working!

```sh
$ dig +short an3 NS @192.168.122.95
ns1.an3.
```

However, there are two problems. 

The SOA (Start of Authority) record is wrong and says the primary is `a.misconfigured.dns.server.invalid`. If this was a nameserver on
the real internet, DNS resolvers wouldn't know where to make queries.

This is the default content and needs to be corrected with `pdnsutil replace-rrset`:

```sh
pdns:~$ doas pdnsutil replace-rrset an3 '' SOA 3600 "ns1.an3 hostmaster.an3 1 10800 3600 604800 3600"
Current records for an3 IN SOA will be replaced
New rrset:
an3. 3600 IN SOA ns1.an3 hostmaster.an3 1 10800 3600 604800 3600

```

You may notice I bumped the serial from `0` to `1`. I don't have any secondary nameservers (yet!), but if I did have any,
this would tell them that the zone has changed.

Also, `ns1.an3` points nowhere, so I need to make an A record for it:

```sh
pdns:~$ doas pdnsutil add-record an3 ns1 a 192.168.122.95
New rrset:
ns1.an3. 3600 IN A 192.168.122.95
```

Which works!

```sh
$ dig ns1.an3 @192.168.122.95 +short
192.168.122.95
```

## What's next?

So I set up PowerDNS Authoritative Server and am now the proud owner of `.an3`, at least if you're using my computer.

But that's not much. Next I plan to set up a secondary, automate the setup of secondaries, and use fun PowerDNS features
like [Lua records](https://doc.powerdns.com/authoritative/lua-records/index.html) and the [REST API](https://doc.powerdns.com/authoritative/http-api/index.html).

Until the next post, see you!
