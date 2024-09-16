# Step 4: The Journey of a Packet - Part 1 (Host)

## Table of Contents

<!--toc:start-->

- [Gateway](#gateway)
- [MAC Address](#mac-address)
- [Summary](#summary)
<!--toc:end-->

Here is a recap of the things we have discovered so far:

- Our `ping` uses a DNS server to understand the IP address of `google.com`,
- By analyzing the configured subnet mask `255.255.255.0` of our host `192.168.1.101`, we understood that the destination IP address `172.217.169.206` lives on a remote network.
- So the ICMP packet created by `ping` needs to go over the Internet to reach that IP address.

In this chapter, we will talk about the first part that our ICMP packet goes through in its journey.

Like I mentioned before, it's obvious at this point that the packet will travel through the Internet.
However, right now the packet lives on our test host, so let's see what happens on the host first.

To understand what happens on our test host, we can take advantage of the TCP/IP layers and of course - the protocols.
To prevent the confusion, I'll skip the part where name resolution happens.
If you are good to go, let's dive into what happens in our host:

1 - `ping` creates an ICMP Echo packet on our host to test the connection.
ICMP is a L3 protocol, so the journey starts on L3.
The IP address of the test host is used as the source IP address, and the IP address `ping` gets from the configured DNS server is used as the destination IP address.

2 - ICMP hands this packet to IP, which is again a L3 protocol. The important thing in here is that the Protocol header value of the packet is ICMP, to let IP know which protocol to hand it over in the destination host.

3 - IP checks the source and destination address (the process we did in the previous chapter), and determines whether the destination address lives on local or a remote network.

Here comes the next part we will discuss:

4 - When IP figures out that the destination address is on a remote network, it decides to forward the packet to the **gateway** of the local network from our host.

So, what is a gateway?

## <a id='gateway'/> Gateway

A gateway is a L3 networking device (commonly known as router) that connects 2 or more distinct networks.
In this sense, you can think that it is really similar to a switch (a L2 device).
However, there are key features that makes routers a _lot_ different than switches, here a list of some differences:

**Routers are responsible from the logical path of a packet, whereas switches are responsible from physical path of a packet.**

The main way of communication for routers is through IP addresses. For switches, MAC addresses are used instead.

**Routers by default, separate broadcast domains of two networks.**

A broadcast domain is a network segment where a host can send a specific message (called _broadcast message_) to reach all the network devices of that segment.
If the domains are separated, that means a host in one broadcast domain cannot send a broadcast message to another host (or device) in a different broadcast domain.

**Switches create separate collision domains for all hosts that are connected to it, but by default it does not create separate broadcast domains.**

A collision domain is a domain that enables the possibility of the messages coming from different hosts to _collide_, resulting in loss of transmission.
Increasing the size of the collision domain increases the possibility of packet loss.
There are prevention methods that are used to solve this problem, but the fact still remains.
In order to not take a huge detour, these prevention methods are skipped in this analysis.

**Routers are way more flexible and configurable than switches.**

As seen in our test network, routers can act as DNS servers and may contain functionalities of many different network devices (Firewall, NAT, etc.).
Switches, in comparison, are a lot more limited when it comes to flexibility.

There are many more differences but to keep you in the context I'll not dive too deep into them.

So, you can think of the gateway as the main entry/exit point of a network.
If there is a packet that is destined for a remote network, it goes through the gateway of the source network.
Likewise, when a packet arrives from a remote network, it has to go through the gateway first.

Now that we understand gateways a little bit, we can continue with the journey:

5 - IP checks the configuration on our test host to find the gateway's IP address.
Remember, we can use `ipconfig` to see the IP address of the gateway (router):

```bash
ipconfig getsummary en0

# ...
# router (ip_mult): {192.168.1.1}
# ...
```

6 - Now that IP knows the gateway address, it can forward the ICMP packet to it.
However, there is another thing unknown at this point, which is essential for IP and pretty much all Ethernet communcations to know before moving on with the transmission.

## <a id='mac-address'/> MAC Address

Up until this point, we mainly dealt with IP addresses, because we were analyzing the _logical path_ of the ICMP packet.
However, there is also a _physical path_ of a packet that it actually needs to travel through.
And for the physical path, instead of looking at IP addresses, we need to look at MAC addresses.

A MAC address is a unique hexadecimal value that is determined by the manufacturer of a device.
It is the primary concept that is used to facilitate communication in L2 (Data Link layer).

So, in order for our host to forward the packet to the gateway, it needs to know the gateway's MAC address.
This MAC address will be used by L2 later on to initiate the transmission.
IP itself cannot figure out the MAC address, but there is a protocol that is used just for this purpose which is called _Address Resolution Protocol (ARP)_.
Essentially, by using ARP our host can ask the question below to the network:

**Hey, I need the MAC address of `192.168.1.1`, please let me know when you get this message.**

Now, ARP is a L3 protocol, and same rules are applied to this protocol as well:

1 - It needs a source IP, which is `192.168.1.1`.
2 - It needs a source MAC address, which is the host's MAC address.
2 - It needs a destination IP, since it is a broadcast message, ARP puts the _broadcast address_ of the subnet, which is `192.168.1.255`.

---

PS: The broadcast address of a subnet is _the last available address of a subnet_, and it does not have to be 255.
It depends on the given subnet mask.

---

However, An ARP packet needs to travel through the physical path just as any other packet.
We still do not have the MAC address for the destination though, so what is used for that?

The answer lies in the fact that ARP sends a _broadcast message_ to the whole network.
And here comes another important thing about broadcasts - which is the fact that the MAC address of a broadcast message is `FF:FF:FF:FF:FF:FF`.

When this MAC address is used in a packet, it indicates that this is a broadcast message so eventually the message gets send to every single device on our test network.
The are two outcomes when an ARP broadcast message is sent:

- When a device with a different IP address gets the message, it simply drops it.
- When the target device gets the message, it adds its MAC address to the message and sends it back to the host.

If you think about it, this operation has to be done in each and every time to communicate with another device on the network, so it adds overhead, just like how DNS does.
Therefore, just like how DNS servers have caches - ARP has a _table_, which is the first place it checks for MAC addresses.
If it cannot find the MAC address on the table, then it proceeds with the broadcast message.

You can check your own ARP table by using `arp` tool we have on Unix/Linux:

```bash
arp -a

# ...
# ? (192.168.1.1) at xx:xx:xx:xx:xx:xx on en0 ifscope [ethernet]
# ...
```

---

PS: The actual MAC address of the gateway is masked for security purposes.

---

If you want to delete the whole table, you can use:

```bash
arp -da
```

Now that we know about ARP and MAC addresses, we can continue with our journey:

7 - ARP is used on the host to get the MAC address of the gateway.
When the MAC address is retrieved, IP adds the address as destination MAC address to the packet. The source MAC address is the address of the host itself.

8 - The packet is handed down to L2. L2 checks the MAC addresses, and then creates _frames_ from the packet.
There is a small detail that we can talk about, and that is the `EtherType` header of the frame. At the destination, the packet needs to be handed to IP to process, therefore the protocol is put into this field on each frame.

9 - Other than specifying EtherType, L2 also runs a Cyclic Redundancy Check (CRC) on each frame and puts the output into the frame's Frame Check Sequence (FCS) field.
This field is used while collecting the frames on the destination device. The L2 of the destination device also runs a CRC and compares it with the FCS field. If both values are the same, it means the data integrity is preserved.

10 - L2 hands the frames to L1 (Physical layer) and then bits are sent through the physical medium. The physical medium of our test network is wireless, so analog radio signals are created from digital signals and those signals are transmitted to be picked up by the gateway.

## <a id='summary'/> Summary

To sum up, here is what has discussed on this chapter:

- We discussed what a _router_ (gateway) is and how it is used in networking.
- We discussed some of the differences between a _switch_ and a _router_.
- We discussed about another important protocol called ARP, and talked about how MAC addresses are used to facilitate communcation between devices.
- We discussed what happens in `ping` in terms of TCP/IP layers, starting from L3 to L1.

And with this chapter, we have completed our analysis of the first part of the journey - the path that our packet takes from our host to the gateway of our test network.

Now, let's proceed with the analysis and see what happens when the signals hit the gateway.

[Previous Chapter](./4-network-of-ip.md) | [Next Chapter](./6-journey-on-gateway.md) | [Main README](./README.md)
