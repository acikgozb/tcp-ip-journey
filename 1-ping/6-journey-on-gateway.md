# Step 5: The Journey of a Packet - Part 2 (Gateway)

## Table of Contents

<!--toc:start-->

- [Routing Table](#routing-table)
  - [Static Routing](#static-routing)
  - [Dynamic Routing](#dynamic-routing)
- [Public/Private IP Addresses](#public-private-ip-addresses)
- [Network Address Translation (NAT)](#network-address-translation-nat)
  - [Static NAT](#static-nat)
  - [Dynamic NAT](#dynamic-nat)
  - [Port Access Translation (PAT)](#port-access-translation-pat)
- [Summary](#summary)
<!--toc:end-->

Here is a quick rundown of what we have gone through in our analysis so far:

- `ping` uses an L3 ICMP packet to test the connection with a specific host.
- When a domain name is given to `ping`, it gets the IP address first via L4 DNS name resolution.
- After getting the IP address, L3 IP checks the source and destination address and decides to forward the packet to the gateway of the network.
- In order to physically move the packet to the gateway, L3 ARP is used to get the MAC address, then the packet is sent down to L2 to start the transmission.
- L2 creates frames from the packet, and forwards it to L1 (Physical) to transmit the frames via the configured physical medium (wireless).

---

PS: For DNS name resolution, the same process happens for L3 and below.
Since DNS is discussed before the layers, I decided explain the process in there.

---

Now, we are at the point where the analog radio signals are received by the gateway.
This chapter will cover exactly what happens on the gateway.

Gateways (routers) are responsible from _determining the best path to a remote network_, and they do this by analyzing the destination IP address.
However, our gateway just received the analog radio signals, so we are at L1.
We need to go our way up to L3 first, before analyzing how it determines the best possible path.
Basically, to get to L3, our gateway has to extract the packet from analog radio signals, and it does this by reversing the steps that were done on our host.
So, here is a quick rundown of these steps:

1 - L1 on gateway demodulates the received analog radio signals and creates digital signals from them.

2 - These signals are sent to L2, and L2 creates frames from the signals. In here, gateway runs the CRC for each frame, and compares it with the generated frame's FCS field. Remember, if both values match, that means the data integrity of the transmitted frames from our host are preserved.

3 - Once frames are generated on our gateway, L2 checks the EtherType values of frames, and sees that the main protocol that handles these frames on L3 is IP. So L2 forwards the frames to L3, and IP extracts the packet from the frames.

Now we are at the L3 in our test gateway. So, the very first thing that the gateway does is to check the destination address of the packet.
Remember that the packet created by `ping` contains `google.com`'s IP address: `172.217.169.206`.

Now that it has the destination address, it tries to _determine the most performant path_ for the packet.
And here comes another important concept that we encounter frequently - a gateway determines the path by looking at its _routing table_.

So, let's see what this _routing table_ is.

## <a id='routing-table'/> Routing Table

A routing table is the main tool that a gateway uses to determine the path of a packet.
In other words, if there is no possible path defined on the routing table, gateway drops the packet at that point.
It simply cannot figure out where to forward the packet, so it decides to not forward it altogether.

So, as we can see, it's pretty important to have our routing tables as up to date as possible.
There are a couple of ways to update the routing table, so let's take a small detour again to discuss them:

### <a id='static-routing'/> Static Routing

The simplest way we can update our routing tables is by through what is called _static routing_.
Static in this context means manually configuring all of our router's routing tables whenever we have a change on our network.
As you can imagine, static routing is simple but since it is manual, the amount of work we need to do may quickly evolve to something we cannot maintain - based on the number of routers and network changes we have.

Of course - if we have a static way of configuring our routing tables, we can guess we have a _dynamic_ way of configuring as well.

### <a id='dynamic-routing'/> Dynamic Routing

Dynamic routing is a way of updating our gateway configuration _automatically_ by using _dynamic routing protocols_ and neighboring gateways.
There are different branches of _routing protocols_ called _Interior Gateway Protocols (IGPs)_ and _Exterior Gateway Protocols (EGPs)_.

IGPs contain 3 different groups in it which are called _Distance Vector (DV)_, _Link State (LS)_, and Hybrid (mix of both).
In the EGP part, the main routing protocol that is used is _Border Gateway Protocol (BGP)_

---

PS: The protocols in each group would take a whole another analysis to cover, so I won't go further into details in here.

---

By using either static or dynamic routing, we update routing tables of our gateways.
However, I need to talk about one more point before moving on.
Remember how we said that the goal of our gateway is to _find the most performant path_ of the packet?
Unfortunately, having a list of networks in the routing table is not enough.
There needs to be an _order of preference_ for the entries in the routing table.
And that preference is done by what is called an _Administrative Distance (AD)_.

Basically, when our gateway updates it's table, it orders the entries based on the AD score of the provided path.
This AD score depends on many things:

- The exit interfaces of our gateway,
- Whether the route is statically or dynamically configured,
- If dynamic routing is used, the routing protocol that is used,
- The performance of the neighboring routers which provided the path (bandwidth, metrics, etc.)

Some routing protocols are known as _higher AD_ than other protocols, and when 2 different entries in a routing table has the same AD, our gateway checks the performance of the given routes.
If the perfomance of two routes are the same, our gateway simply _load balances_ between the routes.

Now that we have discussed how the routing table of our gateway is configured, we can continue with our analysis.

## <a id='public-private-ip-addresses'/> Public/Private IP Addresses

We are at the point where our gateway selects the most performant path from its routing table, and ready to forward the packet.
Before moving on though, it needs to do one more critical task.
Up until this point, we have analyzed lots of different aspects of IP, but one aspect was still missing.
Now is the perfect time to analyze it - and that is the fact that some IP addresses are _public_, and some of them are _private_.

The main difference between a public IP address and a private one is about routing - _private IP addresses are not routable, only public IP addresses are_.

Here is a list of _private IP address ranges_ that helps to understand whether our IP address is public or private:

| From          | To                | CIDR             |
| ------------- | ----------------- | ---------------- |
| `10.0.0.0`    | `10.255.255.255`  | `10.0.0.0/8`     |
| `172.16.0.0`  | `172.31.255.255`  | `172.16.0.0/12`  |
| `192.168.0.0` | `192.168.255.255` | `192.168.0.0/16` |

The question is - is the IP address of our test host `192.168.1.101` public and private?
If you check the table, you can see that the IP address of our host is actually a private IP address.

So, if our test `ping` can successfully test our connection to `google.com`, it means that our private IP address is somehow routed there.
Or is it?

And at this point, we uncover another important concept about routing, which is _Network Address Translation (NAT)_.

## <a id='network-address-translation-nat'/> Network Address Translation (NAT)

NAT is a concept that is used to _map private IP addresses to public IP addresses_, so that hosts with private IP addresses can send packets to the public, and vice versa.
Without NAT, we wouldn't be able to connect to a remote network from our home networks or private networks administrated by companies would have no access to public networks.

NAT also allows us to rely on private IP addresses, without it, we would have to configure public IP addresses for each and every host we have in our networks.
By doing so, we would quickly eat the available IP pool.
So, we can say that NAT is another concept like subnetting which helps us to cope with the limitations of IPv4.

If NAT is configured on a router, the _source IP address of the packet is changed with a public IP address on the router_ before forwarding the packet to the next router.
Likewise, when a remote packet comes to a router, the \*destination IP address of the packet is changed with the proper private IP address on the router, then forwards it to the host living in the private network.

NAT uses a _translation table_ to keep a track of the private-public IP address mappings.
There are 3 different types of NAT, so let's talk about them a little bit.

### <a id='static-nat'></> Static NAT

Static NAT is a concept that uses _manually configured public IP addresses for each host in the network_ to do the translation.
By manually configuring public IP addresses, the translation table ends up with specific public IP addresses for private IP addresses of configured hosts, so when a packet comes, NAT can easily translate the address before forwarding it to another router.

Like static routing, static NAT suffers the same problem - it has a maintenance cost.
Also, the amount of precious public IP addresses we use gets increased by the amount of private hosts in our network.

The next type of NAT is again - similar to routing, and that is called _Dynamic NAT_.

### <a id='dynamic-nat'></> Dynamic NAT

Dynamic NAT is a concept that uses _a configured public IP address pool_ to assign public addresses to each host in the network to do the translation.
Instead of configuring each host by ourselves, we provide a pool of available IP addresses to NAT, and NAT uses those addresses in its translation table.

This concept lowers the maintenance cost coming from static NAT, however the other problem still remains, we may still use a considerable amount of public IP addresses just for translation, which has an affect on the globally available IP address pool.

That is why - there is another type of NAT that was developed, which is called _PAT_.

### <a id='port-access-translation-pat'></> Port Access Translation (PAT)

PAT is a type of NAT that is commonly used in today's networks.
The address translation in PAT is done by source ports, instead of IP addresses themselves.
In this concept, there is only _one public IP address_ that is used for translation, but _the ports of the public IP address change based on the source ports of each host_.
By utilizing ports, even if NAT uses only one public IP address, it knows how to map back and forth.

Now we know how our private host sends packets: _our gateway uses NAT to make the address routable_.

## <a id='summary'/> Summary

We covered a lot in this part of the analysis - let's summarize the outcomes:

- Once analog signals hit our gateway, L1 demodularizes those signals back to digital, and forwards it to L2.
- L2 creates frames from digital signals, and it runs its own Cyclic Redundancy Check to ensure that data integrity is preserved.
- L2 checks the EtherType header of the frames and forwards it to appropriate L3 protocol - in this case IP.
- IP checks the destination IP address and understands that it is destined for a remote network.
- To determine the best possible path for the packet, gateway checks its routing table. This table is updated by either statically or via dynamic routing protocols.
- Once the path is determined, our gateway applies NAT to the private source IP address to make the packet routable.
- Finally, it forwards the packet to the selected path.

This is pretty much it, now the ICMP `ping` packet is officially out of our local network!
Is the journey completed though?
As you might guess, there is still quite a bit to analyze, so let's move on the third part of the journey.

[Previous Chapter](./5-journey-on-host.md) | [Next Chapter](./7-journey-on-hops.md) | [Main README](./README.md)
