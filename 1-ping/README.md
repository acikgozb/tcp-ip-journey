# Study 1: `ping` Over a Remote Network

This study contains an analysis of `ping`, a tool that is commonly used to test connections to hosts.
It focuses on pinging `google.com`, which is a commonly known remote network.

## What is `ping`?

Like it is mentioned above, `ping` is a Unix tool that is used to test a connection to a host.

One of the important things about `ping` is that it uses _Internet Control Management Protocol_ (ICMP) packets to test the connection.
This protocol lives on L3 of the _TCP/IP_ model (Network/Internet Layer), which allows our analysis to focus purely on the connection, unlike L7 protocols such as _HTTP/HTTPS_ or _FTP/SFTP/TFTP_.
Since ICMP lives on L3, it does not use TCP or UDP to function. It is encapsulated in a IPv4 packet.

ICMP is mainly used to understand the `state` of the transmitted packet. Based on the status of the remote location, ICMP can return messages like:

- `Time-to-live Exceeded (TTL)`
- `Destination Unreachable`
- `Request Timed Out`

Using `ping` is pretty straightforward. Here is an example call that will be analyzed:

```bash
# -c1 is used to ping only once, there are no limits by default on Unix.
ping -c1 google.com
```

Please note that `ping` can be also used to test the TCP/IP configuration on a host's NIC by using a loopback address (commonly known as `localhost`):

```bash
ping -c1 127.0.0.1
```

Keep in mind that using the loopback address does not create a packet that traverses the network.

## The Tip of the Iceberg

First, let's see the case in action.
Here is what we see if we `ping` `google.com`:

```bash
ping -c1 google.com

# PING google.com (172.217.169.206): 56 data bytes
# 64 bytes from 172.217.169.206: icmp_seq=0 ttl=56 time=26.257 ms
#
# --- google.com ping statistics ---
# 1 packets transmitted, 1 packets received, 0.0% packet loss
# round-trip min/avg/max/stddev = 26.257/26.257/26.257/nan ms
```

Here is a summary of the output above:

- We successfully pinged `google.com` once,
- We can see the TTL of the ICMP packet coming back from an IP address,
- We have 1 packet transmitted with 0% packet loss,
- There are also additional metrics regarding the round-trip duration.

Based on the output we can say that the operation is pretty straightforward, we send a packet to `google.com`, and an IP address sends us a packet back.
Therefore we can say that we can successfully connect to it.

Is this all what's happening though?
As you probably guessed, there is A LOT that happens behind the scenes.

So if you are ready for an adventure, let us begin.

## Step 1: The Local Network

Before diving into what is happening on the network itself, we have to define the "network" first.
This analysis was written on a computer that was connected to a regular home network.

In this "test" network, there are 2 main devices that is responsible from the communication:

- NIC (Network Interface Card)
- WAP (Wireless Access Point)

### NIC (Network Interface Card)

A NIC is a piece of hardware that connects a device to a network. The device can be anything; computers, switches, routers, you name it.

In this example, the test computer uses its integrated NIC to connect to the wireless home network. As a result, there is an interface on the computer that we can analyze by using `ifconfig`:

```bash
# If your host do not have an Ethernet port, en0 is probably your Wireless interface
# If your host do have an Ethernet port, en1 is probably your Wireless interface
ifconfig en0

# en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
# 	options=400<CHANNEL_IO>
# 	ether 6c:xx:xx:xx:xx:a6
# 	inet6 fe80::899:c1a1:7014:1413%en0 prefixlen 64 secured scopeid 0xf
# 	inet 192.168.1.101 netmask 0xffffff00 broadcast 192.168.1.255
# 	nd6 options=201<PERFORMNUD,DAD>
# 	media: autoselect
# 	status: active
```

In our test, `en0` interface of the computer is used for any communication through the test network.

The output of `ifconfig en0` tells us many things but for the scope of the analysis, we will focus on `ether`, `inet`, `broadcast` fields in the further sections.

PS: The MAC address of the computer (named `ether` in the output) is masked for security purposes.

### WAP (Wireless Access Point)

A WAP is a device that allows wireless devices to connect to a wired network.
This device is frequently used in home networks (also called modems).

Other than connecting wireless devices, it can also have other functions based on how it is configured.
The WAP on the example is used (given by the ISP) as a gateway (router) and a DNS server.

_PS: Remember that in a wireless network the main way to communicate is by sending analog radio signals. These signals are modulated from digital signals in L1 (Physical Layer), and demodulated once they are transmitted to another station. This is why the name "modem" can also be used for WAP's._

### Summary

Okay, so to recap:

- The "test" network we have uses 2 devices: A computer and a WAP.
- The WAP is also configured as a gateway (router) and a DNS server.

Now that we defined what our "network" is, we can proceed with the analysis.
As we progress through, the terms "gateway" and "DNS server" will be more clear.

## Step 2: DNS Is Only For Humans

Lets check our ping to `google.com` again. Throughout the analysis, I will frequently refer to it:

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
If you check carefully, there is an _IP address_ next to `google.com`, which is defined as `172.217.169.206`. It seems like we are pinging to that IP address, not `google.com`.

Or are we?

Here comes the first "aha!" moment of the networking.

IP address is the main concept that is used in networking to facilitate communication between devices.
Internet Protocol sits on L3 of the TCP/IP model, and is the basis of the Internet itself.
The domain names we use - _google.com, youtube.com, github.com, you name it_ - are just there for user convenience. The L3 devices we use for networking only understands IP addresses, they basically have no idea what `google.com` means.
By "tagging" an IP address with a domain name, it provides us two things:

- Users of a domain can reference it by using its domain name from now on (_google.com_), which is a lot easier to remember compared to memorizing its IP address (_172.217.169.206_).
- By abstracting away the IP address, it creates a way of updating IP addresses freely, without disrupting users. Of course this scenario is not as easy as it sounds.

So going back to the example, `ping` actually does what it needs to do, it sends an ICMP packet to the IP address `172.217.169.206`, because it doesn't know what `google.com` is.
However, how does it know that the IP address of `google.com` is actually `172.217.169.206`?

This is where _DNS servers_ join the adventure.

This action of **resolving an IP address from a domain name** is called _name resolution_ and this is primarily handled by DNS servers.
DNS servers do this by storing records that matches domain names with IP addresses.

Essentially, what `ping` does is when it sees a domain name to connect (e.g `google.com`), first it sends a DNS query to a DNS server to ask for the IP address of the host.
DNS server does its name resolution on `google.com` and then responds to `ping` by attaching the IP address of `google.com` to its DNS response.
Once `ping` gets the IP address from a DNS server, then it attempts to connect that IP address as shown in the example.

So, how does this "name resolution" actually work?

### Name Resolution Through a DNS Server

Lets talk about DNS a little bit.
DNS is a **L7 (Application Layer) Service** that allows us to find the IP address of a requested resource.
Based on the implementation, it can leverage both of the **L4 (Transport layer) protocols TCP/UDP** and it's **well-defined port number is 53**.

So now we know that to resolve `google.com`, we need to send a DNS query to a DNS server via TCP/UDP by specifically using the port 53.
But which DNS server should we send the query?

#### Step 1: The Configured DNS Server

Here comes another important point about DNS, there are _tons_ of DNS servers around the world, and there is a specific order of execution happening once a DNS query is sent from a host.

The first place to look for DNS records is actually the computer itself.
On Unix/Linux, there is a special file called `hosts` that can hold the DNS records.
If the requested IP address can be found on the `hosts` file, the host actually does not send a DNS query segment to the network itself.

Let's check the `hosts` file on the example computer:

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
- The host name `broadcasthost` point to the IP address `255.255.255.255`. This IP address is used for broadcast messages, which will be explained in further sections.

As a side info, `hosts` file is considered legacy and it is no longer the primary way of resolving domain names on the Internet.
Therefore, in order to resolve `google.com`, the computer actually sends a DNS query to a DNS server, but which one?

There are different ways which you can use to locate your configured DNS server. Here is a couple of them:

1 - On Unix/Linux, there is a nice command called `scutil` that can show you the name resolver configuration you have on your computer (by using the `--dns` flag):

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
# server_identifier (ip): 192.168.1.1
# subnet_mask (ip): 255.255.255.0
# router (ip_mult): {192.168.1.1}
# domain_name_server (ip_mult): {192.168.1.1}
# ...
```

Again, based on the output, we can see that the domain name server can be reached from `192.168.1.1`.
In the WAP section, I told that the example WAP was configured as a gateway (router) and a DNS server. That conclusion was based on the output of `ipconfig` command above.

Now that we found our DNS server, now it's time to do the actual name resolution.

#### Step 2: Recursive Resolutions

As I mentioned above, there are a _ton_ of DNS servers around the world.
So we can make a guess that our DNS server located at `192.168.1.1` probably does not know every single IP address out on the Internet.
Nonetheless, it's responsibility is to provide IP address to any host in the test network, which is `192.168.1.0`.

So, if our DNS server does not know every IP address, how does it retrieve the IP address of `google.com`, which is `172.217.169.206`?
Here comes the next important thing about DNS, it is the fact that **a DNS server does its name resolution recursively**.

So, what does recursion mean in this context?

It means that if a DNS server cannot provide name resolution for a domain name, it goes and searches other DNS servers around the world to provide that information!
But does a DNS server randomly send DNS queries to other servers? As you probably guessed, it does not. There is a specific order of execution happening during this recursive operation.

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
Starting from the root servers, `dig` is eventually pointed to the DNS servers of Google (`ns1` through `ns4`), and then one of those servers respond with the A record at the end, which contains the IP address.

As we can see from this output `+trace` option is useful to troubleshoot any DNS issues, or to verify the name resolution process.

PS: There are actually quite a bit of record types (A, AAAA, PTR, NS, MX, CNAME, ...), but for our analysis, only NS and A records are examined.

### Summary

Here is what has discussed in this part of the analysis:

- The process of mapping `google.com` to `172.217.169.206` during our example `ping`.
- The importance of DNS, it's place on the TCP/IP layers.
- The name resolution process which is used pretty much anywhere on the Internet.

Now that we have the IP address on our hands and we actually saw where that address came from, we can proceed with our analysis.

Trust me, the further we proceed the better it gets.

## Step 3: Where Does the IP Belong?

Again - let's take a look at our example `ping`:

```bash
ping -c1 google.com

# PING google.com (172.217.169.206): 56 data bytes
# 64 bytes from 172.217.169.206: icmp_seq=0 ttl=56 time=26.257 ms
#
# --- google.com ping statistics ---
# 1 packets transmitted, 1 packets received, 0.0% packet loss
# round-trip min/avg/max/stddev = 26.257/26.257/26.257/nan ms
```

We know from our discussions that the IP address we see (`172.217.169.206`) comes from our DNS server.
We also know that our `ping` successfully transmits a packet to `google.com`.

Here comes the next question - is the IP address `172.217.169.206` points to a remote host or is it actually a host on our test network?
In other words, does `ping` actually goes out on the Internet or does it stay in the local network?
Since the destination is `google.com`, we can guess the answer - it is a remote host.
But can we see this clearly? How can we make sure that an IP address points a local or remote network?

And here comes another vital point of TCP/IP - the "network" and "host" bits of an IP address.

### The Parts of an IP Address

An IP address holds 32 bits - or 4 bytes, and by toggling these bits we create the decimal representation.
Since each decimal part of an IP address is a byte, this means that the possible values of a decimal part can be between 0 and 255.
Here is a visual representation of these ranges:

```bash
    192.    168.    1.      1
#  0-255   0-255  0-255   0-255
```

In previous sections, we saw the NIC configuration of the test host, and we know that the configuration contains an IP address associated with the active interface `en0`.
That IP address is the starting point of our journey, in other words, it is the _source IP address_ that is used to sent an ICMP packet by `ping`.

Let's see that IP address again. This time, I will use `ipconfig` in a different way to specifically get the IP address:

```bash
ipconfig getifaddr en0
# 192.168.1.101
```

From the output, we can see that our test host has the IP address `192.168.1.101`.
This is a quite good start, but there is a missing piece that is really, really important.
Our test host simply does not float on the Internet, it is specifically connected to a local network. So that network **has to have an IP address** as well.
Remember, the thing that we are trying to understand is this:

Given that there are 2 IP addresses `192.168.1.101` and `172.207.169.206`, do these addresses belong to the same network or not?

In order to answer this question, we need to find our local network's IP address. If `172.207.169.206` is an address that is _inside_ our local network, we can say that it belongs to it. If not, then we can safely say that it is an address that lives on a remote network.

And here's the news - we can "guess" that they are on a different network, but there is a wonderful way to verify it.
Before jumping straight to the answer though, let's take a detour first.

### The History of IP

This detour is important because we need to see what happened in the past to understand the "why" of the networking concepts we frequently use in today's networking.
At first, a _classful_ system is designed for IP to specify network addresses and available host addresses for each network.
What I mean by _classful_ is that certain ranges of IP addresses were tied into a specific class.
Here is the list of classes:

- Class A: `0.0.0.0` through `127.255.255.255`
- Class B: `128.0.0.0` through `191.255.255.255`
- Class C: `192.0.0.0` through `223.255.255.255`
- Class D: `224.0.0.0` through `239.255.255.255`
- Class E: `240.0.0.0` through `254.255.255.255`

So for example, if we see an IP address like `10.5.2.240`, based on the classful system above, we could say that it is a Class A address and it's network address is `10.0.0.0`.

This system had worked well for a while but unfortunately due to the fact that there is a maximum limit of IP addresses we can use (around 4.3 billion, even less when we subtract Class D and E, which are reserved IP pools), it started to cause some trouble:

- In Class A and B networks, there is an abundance of IP addresses available to hosts, and most of these addresses were wasted.
- On the other hand, in Class C networks, the available IP address for hosts were too small so most companies could not fully benefit from it.

Essentially, this system was using available IP address pool very inefficiently, so eventually it gave its place to another new concept, which is called _subnetting_.

---

I want to point out that there are certain methods that are used in today's networking to counter the limitations of IPv4, and subnetting is one of those.
However, the fact still remains - we have already run out of the available IP address pool.

The thing that actually solves this limitation is _IPv6_, but that is not the scope of this analysis.

---

So, let's see what subnetting is and how it helps to answer our question.

### Subnetting

Before you continue reading, try to not think about the classful system we discussed above.
The classful system is mentioned simply because we need to understand "why" there is a thing called subnetting.

So, the problem is pretty obvious by now - we need a more efficient way to use IP. This is the primary thing that subnetting solves.

How?

The logic behind subnetting is simple: Subnetting allows us to take a network, and create smaller networks from it.
By optimizing subnetting, we can achieve efficient IP addressing throughout our networks.

---

PS: Efficiency in this context means that a network is utilizing as much of it's available host addresses as possible.

---

If we are creating smaller sub-networks from one big network, that means that **some the host bits of the big network is used as sub-network addresses**.
So, if we see an IP address of `192.168.1.1`, this IP address can mean two different things:

- The IP address can be the network address itself.
- The IP address can be a host that lives in a network.

Therefore, if subnetting is used, then we need to have a way to discover the actual network address of a given IP.
And for this very reason, a _subnet mask_ is usually associated with a given IP address. So if you are given an IP address to analyze, most of the time you are provided with the subnet mask as well, like this:

- `192.168.1.1 255.255.255.0`
- `192.168.1.1/24`

Both of the notations above mean exactly the same thing. The 2nd notation is commonly known as _Classless Inter-Domain Routing (CIDR)_.
So, let's see if our test host has its subnet mask configured.
Remember how we checked the NIC configuration by both `ipconfig` and `ifconfig`?
We can use those programs to find out our subnet mask as well.

Here is our subnet mask with `ipconfig`, with its decimal representation:

```bash
ipconfig getsummary en0

# ...
# server_identifier (ip): 192.168.1.1
# lease_time (uint32): 0xe10
# subnet_mask (ip): 255.255.255.0
# router (ip_mult): {192.168.1.1}
# domain_name_server (ip_mult): {192.168.1.1}
# ...
```

And here is with `ifconfig`, with its hexadecimal representation (netmask):

```bash
ifconfig en0

# ...
# inet 192.168.1.101 netmask 0xffffff00 broadcast 192.168.1.255
# ...
```

As you can see, both commands output the same subnet mask, which is `255.255.255.0`.
So, now that we have our subnet mask, let's find our network address.
There are a couple of different ways to calculate the network address, and they all end up more or less the same.
So here is one way of doing it:

1 - The byte representation of 255.255.255.0 is as follows:

```txt
11111111   11111111    11111111    00000000
```

In here, the bits that are turned on (1) indicate the network address, and the bits that are turned off (0) indicate host addresses.
By looking at this, we can say that for the IP address `192.168.1.101`, we are using 3 bytes for our network. Since we are using 3 bytes, the actual subnetting happens on the third byte, not the first or the second. That byte is the one we should focus.

2 - In the third byte, we use all the bits to determine the subnets.
If all bits are used for subnets, that means there are no bits available for hosts in the third byte.
Therefore, the size of each subnet would be 2^(_available host bit count_) = 1.
This is commonly referred as the _block size_ of a subnet.

3 - After determining the block size, we can start to see the subnets more clearly.
Starting from zero, if we increment the third byte by the block size up until the last possible value (255), we get our subnets.
So for the IP address `192.168.1.101` and the subnet mask `255.255.255.0`, here are the subnets:

- 192.168.0.0
- 192.168.1.0
- 192.168.2.0
- 192.168.3.0
- ....
- 192.168.255.0

4 - Once subnets are more clear, then we take a look at the given IP address and find which subnet it belongs to.
So for our test host and its subnet mask, we can see that the address of the test network is `192.168.1.0`.

---

PS: The subnet mask that is used in the analysis was one of the simpler ones. In reality, we may encounter with masks that are not so easy to figure out.
However, the calculation for those would be exactly the same.

---

With the subnet mask, we can determine another important type of an address, which is called the _broadcast_ address of a network.
The concept of broadcasting will be discussed in later sections, therefore it is skipped in here.

Subnetting is one of the key concepts that is frequently used in today's networks to optimize a given network.
If you want to practice the calculations, you can visit this [website](https://subnetipv4.com/).

Now that we found out that our local network address is `192.168.1.0`, we can answer the question.
The question was:

_Given that there are 2 IP addresses `192.168.1.101` and `172.207.169.206`, do these addresses belong to the same network or not?_

With the help of our subnet mask, we can see that these IP addresses truly belong to different networks.
For our test host to communicate with `172.207.169.206`, it really needs to go over the Internet.

## Summary

In this section of the analysis, we found out:

- How to determine our network addresses,
- Whether a given IP lives in the network or not.

Now that we know `ping` needs to send the ICMP packet to the remote network where `google.com` lives.
So the ICMP packet needs to travel through the Internet. How does this happen?

Like I said, it's getting better and better on each section.
