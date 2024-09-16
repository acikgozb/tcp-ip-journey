# Step 4: Where Does the IP Belong?

## Table of Contents

<!--toc:start-->

- [The Parts of an IP Address](#the-parts-of-an-ip-address)
- [The History of IP](#the-history-of-ip)
- [Subnetting](#subnetting)
- [Summary](#summary)
<!--toc:end-->

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

We know from our discussions that the IP address we see (`172.217.169.206`) comes from our DNS server and it is configured via DHCP.
We also know that our `ping` successfully transmits a packet to `google.com`.

Here comes the next question - is the IP address `172.217.169.206` points to a remote host or is it actually a host on our test network?
In other words, does `ping` actually goes out on the Internet or does it stay in the local network?
Since the destination is `google.com`, we can guess the answer - it is a remote host.
But can we see this clearly?
How can we make sure that an IP address points a local or remote network?

And here comes another vital point of TCP/IP - the "network" and "host" bits of an IP address.

## <a id='the-parts-of-an-ip-address'/> The Parts of an IP Address

An IP address holds 32 bits - or 4 bytes, and by toggling these bits we create the decimal representation.
Since each decimal part of an IP address is a byte, this means that the possible values of a decimal part can be between 0 and 255.
Here is a visual representation of these ranges:

```bash
    192.    168.    1.      1
#  0-255   0-255  0-255   0-255
```

In previous chapters, we saw the NIC configuration of the test host, and we know that the configuration contains an IP address associated with the active interface `en0`.
That IP address is the starting point of our journey, in other words, it is the _source IP address_ that is used to sent an ICMP packet by `ping`.

Let's see that IP address again.
This time, I will use `ipconfig` in a different way to specifically get the IP address:

```bash
ipconfig getifaddr en0
# 192.168.1.101
```

From the output, we can see that our test host has the IP address `192.168.1.101`.
This is a quite good start, but there is a missing piece that is really, really important.
Our test host simply does not float on the Internet, it is specifically connected to a local network. So that network **has to have an IP address** as well.
Remember, the thing that we are trying to understand is this:

Given that there are 2 IP addresses `192.168.1.101` and `172.217.169.206`, do these addresses belong to the same network or not?

In order to answer this question, we need to find our local network's IP address.
If `172.217.169.206` is an address that is _inside_ our local network, we can say that it belongs to it.
If not, then we can safely say that it is an address that lives on a remote network.

And here's the news - we can "guess" that they are on a different network, but there is a wonderful way to verify it.
Before jumping straight to the answer though, let's take a detour first.

## <a id='the-history-of-ip'/> The History of IP

This detour is important because we need to see what happened in the past to understand the "why" of the networking concepts we frequently use in today's networking.
At first, a _classful_ system is designed for IP to specify network addresses and available host addresses for each network.
What I mean by _classful_ is that certain ranges of IP addresses were tied into classes.
Here is the list of them:

- Class A: `0.0.0.0` through `127.255.255.255`
- Class B: `128.0.0.0` through `191.255.255.255`
- Class C: `192.0.0.0` through `223.255.255.255`
- Class D: `224.0.0.0` through `239.255.255.255`
- Class E: `240.0.0.0` through `254.255.255.255`

So for example, if we see an IP address like `10.5.2.240`, based on the classful system above, we could say that it is a Class A address and it's network address is `10.0.0.0`.

This system had worked well for a while but unfortunately due to the fact that there is a maximum limit of IP addresses we can use (around 4.3 billion, even less when we subtract Class D and E, which are reserved IP pools), it started to cause some trouble:

- In Class A and B networks, there is an abundance of IP addresses available to hosts, and most of these addresses were wasted.
- On the other hand, in Class C networks, the available IP address for hosts were too few so most companies could not fully benefit from them.

Essentially, this system was using available IP address pool very inefficiently, so eventually it gave its place to another new concept, which is called _subnetting_.

---

PS: I want to point out that there are certain methods that are used in today's networking to counter the limitations of IPv4, and subnetting is one of those.
However, the fact still remains - we have already run out of the available IP address pool.

The thing that actually solves this limitation is _IPv6_, but that is not the scope of this analysis.

---

So, let's see what subnetting is and how it helps to answer our question.

## <a id='subnetting'/> Subnetting

Before you continue reading, try to not think about the classful system we discussed above.
The classful system is mentioned simply because we need to understand "why" there is a thing called subnetting.

So, the problem is pretty obvious by now - we need a more efficient way to use IP addresses.
This is the primary thing that subnetting solves. But how?

The logic behind subnetting is simple: Subnetting allows us to take a network, and create _smaller networks_ from it.
By optimizing subnetting, we can achieve efficient IP addressing throughout our networks.

---

PS: Efficiency in this context means that a network is utilizing as much of it's available host addresses as possible.

---

If we are creating smaller sub-networks from one big network, that means that **some the host bits of the big network is used as sub-network addresses**.
So, if we see an IP address of `192.168.1.1`, this IP address can mean two different things:

- The IP address can be the network address itself.
- The IP address can be a host that lives in a network.

Therefore, if subnetting is used, then we need to have a way to discover the actual network address of a given IP.
And for this very reason, a _subnet mask_ is usually associated with a given IP address.
So if you are given an IP address to analyze, most of the time you are provided with the subnet mask as well, like this:

- `192.168.1.1 255.255.255.0`
- `192.168.1.1/24`

Both of the notations above mean exactly the same thing.
The 2nd notation is commonly known as _Classless Inter-Domain Routing (CIDR)_.
So, let's see if our test host has its subnet mask configured.
Remember how we checked the NIC configuration by both `ipconfig` and `ifconfig`?
We can use those tools to find out our subnet mask as well.

Here is our subnet mask with `ipconfig`, with its decimal representation:

```bash
ipconfig getsummary en0

# ...
# subnet_mask (ip): 255.255.255.0
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

In here, the bits that are turned on (1) indicate the _network address_, and the bits that are turned off (0) indicate _host addresses_.
By looking at this, we can say that for the IP address `192.168.1.101`, we are using 3 bytes for our network.
Since we are using 3 bytes, the actual subnetting happens on the third byte, not the first or the second.
That byte is the one we should focus.

2 - In the third byte, we use all the bits to determine the subnets.
If all bits are used for subnets, that means there are no bits available for hosts in the third byte.
Therefore, the size of each subnet would be 2^(_available host bit count_) = 1.
This is commonly referred as the _block size_ of a subnet.

3 - After determining the block size, we can start to see the subnets more clearly.
Starting from zero, if we increment the third byte by the block size up until the last possible value (255), we get our subnets.
So for the subnet mask `255.255.255.0`, here are the subnets:

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

_Given that there are 2 IP addresses `192.168.1.101` and `172.217.169.206`, do these addresses belong to the same network or not?_

With the help of our subnet mask, we can see that these IP addresses truly belong to different networks.
For our test host to communicate with `172.217.169.206`, it really needs to go over the Internet.

## <a id='summary'/> Summary

In this chapter of the analysis, we found out:

- How network addresses are determined,
- Whether a given IP lives in the network or not.

Now that we know `ping` needs to send the ICMP packet to the remote network where `google.com` lives.
So the ICMP packet needs to travel through the Internet. How does this happen?

Like I said, it's getting better and better on each chapter.

[Previous Chapter](./3-ip-configuration.md) | [Next Chapter](./5-journey-on-host.md) | [Main README](./README.md)
