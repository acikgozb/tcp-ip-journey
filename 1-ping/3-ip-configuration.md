# Step 3: How Is an IP Configured?

From the previous chapter, we talked about the fact that IP is the main protocol we use to facilitate communications.
We also started to see this more clearly - our test host, the DNS server we analyzed, and the website we want to connect (`google.com`) - they all have IP addresses.
So I wouldn't be surprised if you wonder how these IP's are configured.
Therefore in this chapter, I would like to analyze the configuration of an IP address before moving on.

## Table of Contents

<!--toc:start-->

- [Static IP Configuration](#static-ip-configuration)
- [Dynamic Host Configuration Protocol (DHCP)](#dynamic-host-configuration-protocol-dhcp)
- [Our Host Configuration](#our-host-configuration)
- [DHCP Lifecycle](#dhcp-lifecycle)
  - [DHCP Discover](#dhcp-discover)
  - [DHCP Offer](#dhcp-offer)
  - [DHCP Request](#dhcp-request)
  - [DHCP Ack](#dhcp-ack)
- [Summary](#summary)
<!--toc:end-->

## <a id='static-ip-configuration' /> Static IP Configuration

The very first way that we can configure our networking devices is called _static IP configuration_.
We will see the word "static" throughout our analysis, and it mainly means _configuring something manually_.
In this context, if static configuration is used, that means an administrator manually assigns IP addresses to every single networking device on their network.

As you can imagine - depending on the scale of the network - the maintenance cost may quickly ramp up.
The maintenance cost is not just the initial setup of a device, any changes later on would require additional manual work as well.
However, configuring IP addresses statically is beneficial if a device should has a certain IP address, and has no need for a change later on.

Static IP configuration is not the only way we have.
A different way of configuring IP was developed later on and it is much, much efficient than configuring our devices statically.

## <a id='dynamic-host-configuration-protocol-dhcp' /> Dynamic Host Configuration Protocol (DHCP)

The most common way of IP configuration in today's world is through a protocol called _Dynamic Host Configuration Protocol (DHCP)_.
Regarding TCP/IP layers, this protocol sits on L4 (Application layer) and facilitates configuration via a _client-server_ communication.

---

PS: A client-server network is a type of network where a client (host) asks for resources from a server.
The security and authentication of the resources are server's repsonsibility.
There is another type of network called _peer-to-peer_, and for that network everyone shares the same resources to each other.
The security and authentication of a resource depends on each peer.

---

DHCP is different from static IP configuration based on the fact that a DHCP server uses a _pre-configured IP address pool_ to distribute the addresses to the devices (clients) on a network.
By using DHCP, an administator only needs to configure the available IP pool for a DHCP server.
They do not need to configure each host by hand, therefore the maintenance cost of this approach is a lot more manageable than static IP configuration.
There are a couple of more advantages of using DHCP:

- Each IP that is given by a DHCP server has a _lease time_ when a client receives it. This allows DHCP server to use the IP addresses more than once for different devices.
- Some clients can _reserve a specific IP_ when their lease period ends. This is called _MAC reservation_, since the reservation is determined based on the MAC address of the client.

Now that we talked the differences between DHCP and static configuration, let's see how it works in action.

## <a id='our-host-configuration' /> Our Host Configuration

We know that to use DHCP, we need to have a DHCP server and a DHCP client.
Can we see this from our test host's IP configuration?

Remember that we have an interface `en0` which we use to connect to our test network:

```bash
ifconfig en0

# en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
# 	options=400<CHANNEL_IO>
# 	ether xx:xx:xx:xx:xx:xx
# 	inet6 fe80::899:c1a1:7014:1413%en0 prefixlen 64 secured scopeid 0xf
# 	inet 192.168.1.101 netmask 0xffffff00 broadcast 192.168.1.255
# 	nd6 options=201<PERFORMNUD,DAD>
# 	media: autoselect
# 	status: active
```

We can see from the output that our host has an IP address `192.168.1.101`. I for sure did not configure this IP address.
So we need to find:

- How this IP was configured,
- If it is DHCP, the DHCP server which provided this IP address,
- The lease time, if there is one.

And here is the good news - we can find answers for all these questions with `ipconfig`, which we already used in DNS chapter.

1 - Let's see how the IP is configured first. I'm only highlighting the necessary places on the output.

```bash
ipconfig getsummary en0

# ...
# IPv4 : <array> {
#   0 : <dictionary> {
#     Addresses : <array> {
#       0 : 192.168.1.101
#     }
#     ChildServiceID : LINKLOCAL-en0
#     ConfigMethod : DHCP
# ...
```

As you can see from the output, the `ConfigMethod` is set to DHCP. This means our test host essentially has a DHCP client built in!

2 - Let's find the DHCP server. Here is the corresponding configuration in our output:

```bash
ipconfig getsummary en0

# ...
# ciaddr = 192.168.1.101
# yiaddr = 192.168.1.101
# siaddr = 192.168.1.1
# giaddr = 0.0.0.0
# chaddr = 6c:7e:67:d7:d5:a6
# ...
# Options:
# Options count is 11
# dhcp_message_type (uint8): ACK 0x5
# server_identifier (ip): 192.168.1.1
# ...
```

There are actually 2 places in our output that we can see the DHCP server's IP address.
The first one is the field `siaddr`, which basically means the _server IP address_ that our host got its configuration from.
The second one is the field `server_identifier (ip)`, which is a field that exists on the options of the DHCP ACK message.

In previous chapters, we discovered how our test WAP was configured as a gateway and a DNS server.
From this output, we can see that our WAP `192.168.1.1` was also configured as a DHCP server as well!

3 - All that is left is the lease time, if there is any. Let's see the relevant place:

```bash
ipconfig getsummary en0

# ...
# IPv4 : <array> {
#   0 : <dictionary> {
#     Addresses : <array> {
#       0 : 192.168.1.101
#     }
#     ChildServiceID : LINKLOCAL-en0
#     ConfigMethod : DHCP
#     DHCP : <dictionary> {
#       LeaseExpirationTime : 09/15/2024 16:06:11
#       LeaseStartTime : 09/15/2024 15:06:12
# ...
```

The lease time can be seen from our IP configuration values.
Since there is an expiration time, our test host needs to send a _DHCP request_ to renew it's IP configuration before the configuration expires.

Now that we know our test host was configured via DHCP, let's dive a little bit deeper.

## <a id='dhcp-lifecycle' /> DHCP Lifecycle

We can use `ipconfig` to see the configuration values, but right now we can't see _how_ the whole thing happened.
Therefore, in this section I want to talk about the communication between a DHCP client and a DHCP server.
There are a couple of specific DHCP messages that are used between the client and server to get the IP configuration.
Let's briefly talk about the main ones.

### <a id='dhcp-discover' /> DHCP Discover

The very first step of this lifecycle starts from the DHCP client.
At this point, the client does not know anything about the DHCP server, so it tries to search for one.
Therefore, the client _broadcasts_ a DHCP Discover message to the whole network to find a DHCP server.

---

PS: The term _broadcast_ will be explained in detail in Chapter 5.
For now, we can say that a broadcast message is a type of message that gets sent to every host on a network.

---

### <a id='dhcp-offer' /> DHCP Offer

The lifecycle continues with a DHCP Discover message reaching to a DHCP server.
At this point, a DHCP server understands that a DHCP client tries to find a server.
Therefore, the server that gets the DHCP Discover message _reserves_ an IP address to that client and responds with a DHCP Offer message.

The important thing in here is that the broadcasted DHCP Discover message may reach more than one DHCP server.
Therefore, the client _may receive multiple DHCP Offer messages_ at the end.

### <a id='dhcp-request' /> DHCP Request

Once the client get a DHCP Offer message from a server, it accepts that offer and sends a DHCP Request to the server to confirm the IP address.
If there are multiple DHCP Offer messages, the client _chooses one_ and sends a DHCP Request message to only that server.
The way the client chooses an offer is based on _first come first serve_.

### <a id='dhcp-ack' /> DHCP Ack

The last main message type is called DCHP Ack, which is what the DHCP server returns to the client after the client confirms the offer.
This message contains the configuration we see in `ipconfig`.
We already saw this message back when we are looking at our host's configuration:

```bash
ipconfig getsummary en0

# ...
# dhcp_message_type (uint8): ACK 0x5
# server_identifier (ip): 192.168.1.1
# ...
```

And by getting a DHCP Ack from a server, the client gets configured _until the lease time expires_.
I have to say there are more types of DHCP messages. Since these four are the main ones, I wanted to explain these.

All of these messaging between a client and a server happens via _User Datagram Protocol (UDP)_ and via **well-defined ports 67 for the server and 68 for the client**.
UDP sits on L3 (Transport Layer) of TCP/IP, and it is a _connectionless_ communication between two hosts.

## <a id='summary' /> Summary

Here is a wrap up of this chapter:

- We talked about the ways we can use to configure IP addresses of our hosts,
- We analyzed how our test host was configured via DHCP,
- We briefly talked about the main message types of DHCP to dive a little bit deeper into the client-server communication.

We started to uncover the secrets behind the IP addresses, but there is still much to talk about them.
Let's see where the road takes us.

[Previous Chapter](./2-dns.md) | [Next Chapter](./4-network-of-ip.md) | [Main README](./README.md)
