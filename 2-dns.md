# Step 2: DNS Is Only For Humans

## Table of Contents

<!--toc:start-->

- [Name Resolution Through a DNS Server](#name-resolution-through-a-dns-server)
- [The Configured DNS Server](#the-configured-dns-server)
- [Recursive Resolutions](#recursive-resolutions)
- [Summary](#summary)
<!--toc:end-->

Let's check our ping to `google.com` again. Throughout the analysis, I will frequently refer to it:

```bash
ping -c1 google.com

# PING google.com (172.217.169.206): 56 data bytes
# 64 bytes from 172.217.169.206: icmp_seq=0 ttl=56 time=26.257 ms
#
# --- google.com ping statistics ---
# 1 packets transmitted, 1 packets received, 0.0% packet loss
# round-trip min/avg/max/stddev = 26.257/26.257/26.257/nan ms
```

In this example, there is a subtle thing going on during the execution.
If you check carefully, there is an _IP address_ next to `google.com`, which is defined as `172.217.169.206`. It seems like we are pinging to that IP address, not `google.com`. Or are we?

Here comes the first "aha!" moment of the networking.

IP address is the main concept that is used in networking to facilitate communication between devices.
Internet Protocol sits on L3 of the TCP/IP model, and is the basis of how we communicate between devices in today's world.
The domain names we use - _google.com, youtube.com, github.com, you name it_ - are just there for user convenience.
The L3 devices we use for networking only understands IP addresses, they basically have no idea what `google.com` means.
By "mapping" an IP address with a domain name, it provides us many advantages.
Here is a couple of them:

- Users of a domain can reference it by using its domain name from now on (_google.com_), which is a lot easier compared to memorizing its IP address (_172.217.169.206_).
- It abstracts away the IP address from the end user, which makes it a lot easier to administer networking devices.

So going back to the example, `ping` actually does what it needs to do, it sends an ICMP packet to the IP address `172.217.169.206`, because it doesn't know what `google.com` is.
However, how does it know that the IP address of `google.com` is actually `172.217.169.206`?

This is where _DNS servers_ join the adventure.

This action of **resolving an IP address from a domain name** is called _name resolution_ and this is primarily handled by DNS servers.
DNS servers do this by storing records that matches domain names with IP addresses.

Essentially, what `ping` does is when it sees a domain name to connect (e.g `google.com`), first it sends a DNS query to a DNS server to ask for the IP address of the host.
DNS server does its name resolution on `google.com` and then responds to `ping` by attaching the IP address of `google.com` to its DNS response.
Once `ping` gets the IP address from a DNS server, then it attempts to connect that IP address as shown in the example.

So, how does this "name resolution" actually work?

## <a id='name-resolution-through-a-dns-server'/> Name Resolution Through a DNS Server

Lets talk about DNS a little bit.
DNS is a **L4 (Application Layer) Service** that allows us to find the IP address of a requested resource.
Based on the implementation, it can leverage both of the **L4 (Transport layer) protocols TCP/UDP** and it's **well-defined port number is 53**.

---

PS: Please note that DNS is not only used for name resolution - it's just one of the features.

---

So now we know that to resolve `google.com`, we need to send a DNS query to a DNS server via TCP/UDP by specifically using the port 53.
But which DNS server should we send the query?

## <a id='the-configured-dns-server'/> The Configured DNS Server

Here comes another important point about DNS, there are _tons_ of DNS servers around the world, and there is a specific order of execution happening once a DNS query is sent from a host.

The first place to look for DNS records is actually the host itself.
On Unix/Linux, there is a special file called `hosts` that can hold the DNS records.
If the requested IP address can be found on the `hosts` file, the host actually does not send a DNS query to the network itself.

Let's check the `hosts` file on the test host:

```bash
cat /etc/hosts

##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
# 127.0.0.1	        localhost
# 255.255.255.255	broadcasthost
# ::1               localhost
```

As we can see, there are no records pointing to `google.com`, instead we have the following:

- The host name `localhost` pointing the IP address `127.0.0.1`, which is the loopback address we frequently use during development or troubleshooting.
- The host name `localhost` pointing the IP address `::1`. This is actually a _IPv6_ address, and it is a whole another world compared to _IPv4_ (or simply, IP).
- The host name `broadcasthost` point to the IP address `255.255.255.255`. This is an alias IP address is used for broadcast messages, which will be explained in further chapters.

As a side info, `hosts` file is considered legacy and it is no longer the primary way of resolving domain names in today's networking.
Therefore, in order to resolve `google.com`, the host actually sends a DNS query to a DNS server, but which one?

There are different ways which you can use to locate your configured DNS server. Here is a couple of them:

1 - On Unix/Linux, there is a nice command called `scutil` that can show you the name resolver configuration you have on your host (by using the `--dns` flag):

```bash
scutil --dns

# DNS configuration
#
# resolver #1
#   nameserver[0] : 192.168.1.1
#   if_index : 15 (en0)
#   flags    : Request A records
#   reach    : 0x00020002 (Reachable,Directly Reachable Address)
#
# DNS configuration (for scoped queries)
#
# resolver #1
#   nameserver[0] : 192.168.1.1
#   if_index : 15 (en0)
#   flags    : Scoped, Request A records
#   reach    : 0x00020002 (Reachable,Directly Reachable Address)
```

As you can see, the DNS server can be reached on `192.168.1.1`.

2 - Remember the wireless interface `en0`? Well, we can use another command called `ipconfig` to view the interface's IP configurations. I'm only highlighting the relevant place on the output:

```bash
ipconfig getsummary en0

# ...
# router (ip_mult): {192.168.1.1}
# domain_name_server (ip_mult): {192.168.1.1}
# ...
```

Again, based on the output, we can see that the domain name server can be reached from `192.168.1.1`.
In the WAP section, I told that the example WAP was configured as a gateway (router) and a DNS server. That conclusion was based on the output of `ipconfig` command above.

Now that we found our DNS server, now it's time to do the actual name resolution.

## <a id='recursive-resolution'/> Recursive Resolutions

As I mentioned above, there are a _ton_ of DNS servers around the world.
So we can make a guess that our DNS server located at `192.168.1.1` probably does not know every single IP address out on the Internet.
Nonetheless, it's responsibility is to provide IP address to any host in the test network.

So, if our DNS server does not know every IP address, how does it retrieve the IP address of `google.com`, which is `172.217.169.206`?
Here comes the next important thing about DNS, it is the fact that **a DNS server does its name resolution recursively**.

So, what does recursion mean in this context?

It means that if a DNS server cannot provide name resolution for a domain name, it goes and searches other DNS servers around the world to provide that information!
But does a DNS server randomly send DNS queries to other servers?
As you probably guessed, it does not.
There is a specific order of execution happening during this recursive operation.

Here is a rundown of how recursion works for `google.com`:

1 - The name lookup actually starts from the end of a domain name. So, to understand where `google.com` is, first our test DNS server needs to find where `.com` lives. So it sends a DNS query to the _root DNS servers_ to find the location which holds information about `.com` records. The root DNS servers are globally well-known servers. For more information, you can check [this](https://www.iana.org/domains/root/servers) link.

2 - One of the root servers from the list responds to our DNS server with a list of locations that holds `.com` records.

3 - At this point, our DNS server recursively send another query - this time to the `.com` servers - until it finds the location that holds `google.com`.

4 - One of the `.com` nameservers respond to our query by listing which servers hold `google.com` information.

5 - Finally, our DNS server again recursively send another query - this time to the servers holding `google.com` - until it finally gets the IP address.

6 - Once it gets the IP address, it is sent back to the `ping`.

As you can see, based on the domain name, there are quite a few steps to perform just to get the IP address of a domain.
Since domain name resolution happens pretty frequently on the Internet, this process actually creates an overhead in each and every connection.
Therefore, when a new domain name resolution hits to DNS servers, they actually cache the response to improve the performance of the resolution.
So, if we send the same query again, this time we get the response from the cache, but the location of the cache may vary based on the TTL values.

Therefore, it is important to understand that when there is an update regarding a DNS record, the changes of that record may take some time to propagate to all other DNS servers around the world. This is called _DNS propagation_.

Before continuing the analysis, I just want to showcase another command that we frequently use for name resolution, which is called `dig`.
If we just want to see the IP address of a domain, we can just use `dig` like this:

```bash
dig google.com

# <<>> DiG 9.10.6 <<>> google.com
# ;; global options: +cmd
# ;; Got answer:
# ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25494
# ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
#
# ;; OPT PSEUDOSECTION:
# ; EDNS: version: 0, flags:; udp: 1232
# ;; QUESTION SECTION:
# ;google.com.			IN	A
#
# ;; ANSWER SECTION:
# google.com.		248	IN	A	172.217.169.206

# ;; Query time: 11 msec
# ;; SERVER: 192.168.1.1#53(192.168.1.1)
# ;; MSG SIZE  rcvd: 55
```

As you can see, `dig` returns us an A record, which is used to point a domain to a IP address.

If we want to see the recursive resolution part, we can use `dig` like this:

```bash
dig google.com +trace

# ; <<>> DiG 9.10.6 <<>> google.com +trace
# ;; global options: +cmd
# .			435650	IN	NS	a.root-servers.net.
# .			435650	IN	NS	d.root-servers.net.
# .			435650	IN	NS	h.root-servers.net.
# .			435650	IN	NS	b.root-servers.net.
# .			435650	IN	NS	c.root-servers.net.
# .			435650	IN	NS	g.root-servers.net.
# .			435650	IN	NS	j.root-servers.net.
# .			435650	IN	NS	l.root-servers.net.
# .			435650	IN	NS	m.root-servers.net.
# .			435650	IN	NS	e.root-servers.net.
# .			435650	IN	NS	f.root-servers.net.
# .			435650	IN	NS	k.root-servers.net.
# .			435650	IN	NS	i.root-servers.net.
# ;; Received 1097 bytes from 192.168.1.1#53(192.168.1.1) in 11 ms
#
# com.			172800	IN	NS	a.gtld-servers.net.
# com.			172800	IN	NS	b.gtld-servers.net.
# com.			172800	IN	NS	c.gtld-servers.net.
# com.			172800	IN	NS	d.gtld-servers.net.
# com.			172800	IN	NS	e.gtld-servers.net.
# com.			172800	IN	NS	f.gtld-servers.net.
# com.			172800	IN	NS	g.gtld-servers.net.
# com.			172800	IN	NS	h.gtld-servers.net.
# com.			172800	IN	NS	i.gtld-servers.net.
# com.			172800	IN	NS	j.gtld-servers.net.
# com.			172800	IN	NS	k.gtld-servers.net.
# com.			172800	IN	NS	l.gtld-servers.net.
# com.			172800	IN	NS	m.gtld-servers.net.
# com.			86400	IN	DS	19718 13 2 8ACBB0CD28F41250A80A491389424D341522D946B0DA0C0291F2D3D7 71D7805A
# ;; Received 1170 bytes from 198.97.190.53#53(h.root-servers.net) in 43 ms
#
# google.com.		172800	IN	NS	ns2.google.com.
# google.com.		172800	IN	NS	ns1.google.com.
# google.com.		172800	IN	NS	ns3.google.com.
# google.com.		172800	IN	NS	ns4.google.com.
# ;; Received 644 bytes from 192.48.79.30#53(j.gtld-servers.net) in 140 ms
#
# google.com.		300	IN	A	172.217.169.206
# ;; Received 55 bytes from 216.239.32.10#53(ns1.google.com) in 53 ms
```

In this trace output, we first see NS (nameserver) records, which are records that point a server that may hold the information.
`dig` uses these servers to search for `google.com`.
Starting from the root servers, `dig` is eventually pointed to the DNS servers of Google (`ns1.google.com` through `ns4.google.com`), and then one of those servers responds with the A record at the end, which contains the IP address.

As we can see from this output `+trace` option is useful to troubleshoot any DNS issues, or to verify the name resolution process.

---

PS: There are actually quite a bit of record types (A, AAAA, PTR, NS, MX, CNAME, ...), but for our analysis, only NS and A records are examined.

---

## <a id='summary'/> Summary

Here is what has discussed in this part of the analysis:

- The process of mapping `google.com` to `172.217.169.206` during `ping`.
- The importance of DNS, it's place on the TCP/IP layers.
- The name resolution process which is used pretty much anywhere on the Internet.

Now that we have the IP address on our hands and we actually saw where that address came from, we can proceed with our analysis.

Trust me, the further we proceed the better it gets.

[Previous Chapter](./1-the-local-network.md) | [Next Chapter](./3-ip-configuration.md) | [Main README](./README.md)
