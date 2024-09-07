# Study 1: `ping` Over a Remote Network

This study contains an analysis of `ping`, a tool that is commonly used to test connections to hosts.
The study will focus on pinging `google.com`, which is a commonly known remote network.

## What is `ping`?

Like it is mentioned above, `ping` is a Unix tool that is used to test a connection to a host.

One of the important things about `ping` is that it uses Internel Control Management Protocol (ICMP) packets to test the connection.
This protocol lives on L3 of the _TCP/IP_ model (Network/Internet Layer), which allows our analyze to focus purely on the connection, unlike L7 protocols such as `HTTP/HTTPS` or `FTP/SFTP/TFTP`.
Since ICMP lives on L3, it does not use TCP or UDP to function. It is encapsulated in a IPv4 packet.

ICMP is mainly used to understand the `state` of the transmitted packet. Based on the status of the remote location, ICMP can return messages like:

- Time-to-live Exceeded (TTL)
- Destination Unreachable
- Request Timed Out

Using `ping` is pretty straightforward. Here is an example call that we will analyze:

```bash
# -c1 is used to ping only once, there are no limits by default on Unix.
ping -c1 google.com
```

Please note that `ping` can be used to test the TCP/IP configuration on a host by using a loopback address (commonly known as `localhost`):

```bash
ping -c1 127.0.0.1
```

Keep in mind that using the loopback address does not create a packet that traverses the network.

## The Tip of the Iceberg

First, let's see the case study in action.
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
- We can see the TTL of the ICMP packet coming back from `google.com`,
- We have 1 packet transmitted with 0% packet loss,
- There are also additional metrics regarding the round-trip duration.

Based on the output we can say that the operation is pretty straightforward, we send a packet to `google.com`, and `google.com` sends us a packet back.
Therefore we can say that we can successfully connect to it.

Is this all what's happening though?
As you probably guessed, there is A LOT that happens behind the scenes.

So if you are ready for an adventure, let us begin.

## Step 1: The Local Network

Before diving into what is happening on the network itself, we have to define the "network" first.
This analysis was written on a computer that was connected to a regular home network.

In this home network, there are 2 main devices that is responsible from communication:

- NIC (Network Interface Card)
- WAP (Wireless Access Point)

### NIC (Network Interface Card)

A NIC is a piece of hardware that connects a device to a network. The device can be anything; computers, switches, routers, you name it.

In this example, the computer uses its integrated NIC to connect to the wireless home network. As a result, there is an interface on the computer that we can analyze by using `ifconfig`:

```bash
# If your host do not have an Ethernet port, en0 is probably your Wireless interface
# If your host do have an Ethernet port, en1 is probably your Wireless interface
ifconfig en0

# en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
# 	options=400<CHANNEL_IO>
# 	ether 6c:77:67:dd:d2:a6
# 	inet6 fe80::899:c1a1:7014:1413%en0 prefixlen 64 secured scopeid 0xf
# 	inet 192.168.1.101 netmask 0xffffff00 broadcast 192.168.1.255
# 	nd6 options=201<PERFORMNUD,DAD>
# 	media: autoselect
# 	status: active
```

`en0` interface of the computer is used for any communication through the network.

The output of `ifconfig en0` tells us many things but for the scope of the analysis, we will focus on `ether`, `inet`, `broadcast` fields in the further sections.

### WAP (Wireless Access Point)

A WAP is a device that allows wireless devices to connect to a wired network.
This device is frequently used in home networks (also called modems).

Other than connecting wireless devices, it can also have other functions based on how it is configured.
The WAP I currently use (given by the ISP) also acts as a gateway (router) and a DNS server.

_PS: Remember that in a wireless network the main way to communicate is by sending analog radio signals. These signals are modulated from digital signals in L1 (Physical Layer), and demodulated once they are transmitted to another station. This is why the name "modem" can also be used for WAP's._

<hr>

Okay, so to recap:

- The home network that is used has 2 devices: A computer and a WAP.
- The WAP is also configured as a gateway (router) and a DNS server.

Now that we defined what our "network" is, we can proceed with the analysis.
As we progress through the analysis, the terms "gateway" and "DNS server" will be explained in further sections.
