# Step 6: The Journey of a Packet - Part 3 (Hops)

## Table of Contents

<!--toc:start-->

- [The Determined Path](#the-determined-path)
- [`traceroute`](#traceroute)
- [Hops](#hops)
- [Summary](#summary)
<!--toc:end-->

We left the journey at our gateway, which picked up the analog radio signals from our host and eventually extracted the given packet.
It checked its own routing table to determine the best path for the given destination IP address.
Once it determined the best path for that packet, it proceeded with the transmission.

However, by determining the path, it brought us some questions along with it:

- Can we see the path that `ping` takes?
- Is it a path that straight goes to the remote host, or is it something else?

So let's try to answer these questions.

## <a id='the-determined-path'/> The Determined Path

Like it is mentioned in previous chapter, our gateway chooses a path by checking its routing table.
The routing table is mostly updated by _dynamic routing protocols_ with the help of _neighboring routers_.
Therefore, we can say that the routing table of our gateway has:

- Updates coming from **other gateways**,
- Records associated with what is connected to **its own interfaces**,
- Records that are **statically configured** (not every record has to be dynamic).

Based on this information, we can say that:

- If the destination IP address belongs to one of the networks that is connected to its interface, then it is a path that _goes straight to the destination network_.
- If the destination IP address belongs to either static or dynamic records, then we can't say for certain that path goes straight to the destination network.

To fully answer our first question, we can utilize a perfect tool called `traceroute`.

## <a id='traceroute'/> `traceroute`

`traceroute` is a tool similar to `ping` that uses ICMP packets to understand the path of a packet.
`traceroute` specifically uses _UDP_ and the _TTL_ attribute of ICMP to record the location of a packet.
ICMP's "TTL Exceeded" message is listened by UDP probes and when TTL value expires on a gateway, an error message is directly returned to `traceroute`.
It then understands that at that specific gateway the TTL is exceeded, therefore it records the gateway, increases the value of TTL by 1 and then transmits another ICMP packet, and this process is executed recursively until the packet is reached to a host.

Time to see the tool in action.
First, let's use it to see the path to our gateway from our local network.
Remember, the IP address of our gateway is `192.168.1.1`:

```bash
traceroute 192.168.1.1

# traceroute to 192.168.1.1 (192.168.1.1), 64 hops max, 52 byte packets
# 1  192.168.1.1 (192.168.1.1)  6.853 ms  2.236 ms  2.844 ms
```

As you probably guessed, since our gateway lives on the local test network, the given path does not have any "intermediate" gateways.
The packet straight goes to our gateway.
This was an easy one.

Let's see what happens when we use `traceroute` for `google.com`. Remember, our destination IP address is `172.217.169.206`:

```bash
traceroute 172.217.169.206

# traceroute to 172.217.169.206 (172.217.169.206), 64 hops max, 52 byte packets
# 1  192.168.1.1 (192.168.1.1)  6.853 ms  2.236 ms  2.844 ms
# 2  172.17.1.111  5.099 ms  8.908 ms  4.802 ms
# 3  * 31.223.39.161  12.556 ms  *
# 4  159.146.100.66  10.083 ms  5.614 ms  5.205 ms
# 5  31.223.39.157  8.217 ms  *  23.588 ms
# 6  95.70.195.41  18.809 ms  19.506 ms  19.353 ms
# 7  193.192.105.114  14.427 ms  17.518 ms  14.277 ms
# 8  192.192.105.113  17.601 ms  14.923 ms  2.844 ms
# 9  * * *
# 10 142.251.52.82 22.145 ms
#    142.250.212.22 14.922 ms
#    142.251.236.33 15.840 ms
# 11  108.170.236.33  17.587 ms
#     192.178.108.44 18.025 ms 15.529 ms
# 12  172.217.169.206  16.848 ms 14.655 ms
```

Wow, that's a lot of IP addresses right there.
There are a couple of interesting things on the output above, so let's analyze this path.

---

PS: The output of `traceroute` shows the path of its own ICMP packet.
This does not mean that every single connection to `google.com` will go through the same path (including the `ping` that we have been analysing).
Remember, gateways may choose different paths from their routing tables based on the incoming updates and their AD values. Thus, if you try `traceroute`, you may see a different path.

---

## <a id='hops'/> Hops

If we check the output, the very first thing we notice is the amount of IP addresses we have compared to the first example.
The first IP address is our gateway's IP address, and the last one is the destination IP address.

The IP addresses in between are commonly called _hops_ and each of them represent a _gateway along the path_.
Basically, our ICMP packet hops betweeen different gateways around the world until it reaches its destination host.

Another interesting thing on our output is the asterisks.
If you check the output again, we have asterisks for our 9th gateway.
This can mean two things:

- That gateway may have been configured to not throw ICMP "TTL Exceeded" errors.
- The returning ICMP error packets from 9th gateway may have too little TTL values that makes it not reach back to our host.

If we had the administration of that gateway, we would be able to understand the exact reason.

---

PS: I would really recommend you to check the man pages for `traceroute`, it has a couple of examples and detailed explanations of them.

---

The last interesting thing on our output is really subtle - it is the fact that our third gateway has _an private IP address_ associated with it.
We just talked about private IP addresses being non-routable - so how is this possible?

To understand this, let's take a closer look at how `traceroute` prints a gateway:

- Based on the hops, `traceroute` sets the TTL values of ICMP packets (e.g TTL=1 for first hop, 2 for second, and so on),
- When TTL is expired on a certain router, it returns an ICMP error packet back to the source host, **assigning its IP address as source address**.
- If the error packet manages to reach back to our host, `traceroute` sees that an error occured from a new location, thus it extracts the source IP address from that error packet and prints it with other metrics.

Now, the reason why we see a private IP address is because the ICMP error packet actually comes from **a private gateway in a remote network**, and since it does not have a public IP address, it puts the only address it has - a private one.

---

PS: A private gateway in this context means a gateway that is responsible from connecting private networks via its routing table.

---

This is actually expected when it comes to gateways managed by ISPs.
ISPs have huge networks and if every gateway in those networks had a public IP address, we would run out of available IP addresses way sooner.

This is why the third gateway is pretty interesting, it basically tells us that **one of the possible paths that goes to google.com contains a private network**.

## <a id='summary'/> Summary

In this chapter, we added one new tool called `traceroute` to our analysis to understand the path of our ICMP packet starting from our gateway to the destination itself.
Remember, these were our questions:

- Can we see the path that `ping` takes?
- Is it a path that straight goes to the remote host, or is it something else?

Based on the output from `traceroute`, we can say that:

- We can see a possible path that goes to `google.com`, but the actual path that `ping` takes may vary based on the performance of the different paths at that time.
- The path that goes to `google.com` consists of many different gateways around the world - it's not a path that goes straight without any gateways in between.

We almost completed the first part of our analysis.
Now comes the second part - the ICMP Echo response which is generated at the destination host.

[Previous Chapter](./6-journey-on-gateway.md) | [Next Chapter](./8-icmp-response.md) | [Main README](./README.md)
