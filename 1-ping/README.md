# Study 1: Pinging Over a Remote Network

This study contains an analysis of `ping`, a tool that is commonly used to test connections to hosts.
It focuses on pinging `google.com`, which is a well-known website.
The goal is to explore how it works and briefly mention the concepts that are used.

First of all, I would like to thank you for checking it out.
I tried to make it as intriguing as possible without going too much into details.
I was curious about the way `ping` works whilst studying networking for the past 1.5 months, thus I wanted to write an analysis about it to reinforce my understanding.
Since it is a tool that is frequently used, I thought it would be a nice topic to discuss.

## First, a Disclaimer

Before starting the analysis, I want to point out a couple of things:

- The notes provided in here are just my understanding of several networking concepts.
  There may be some misunderstandings, and if that is the case I would really like to have a feedback from you to correct it, if you have time.

- There were times where I had to explain a specific networking concept briefly to not steer too much away from the main topic.
  Therefore the concepts that are explained in this study are actually much more deep than what they appear to be.

- No AI was harmed during the process.

Alright, let's see where this study takes us.

## Table of Contents

<!--toc:start-->

- [What is `ping`?](#what-is-ping)
- [The Tip of the Iceberg](#the-tip-of-the-iceberg)
- [Chapters](#chapters)
- [Conclusion](#conclusion)

<!--toc:end-->

## <a id='what-is-ping'/> What is `ping`?

Like it is mentioned above, `ping` is a Unix tool that is used to test a connection to a host.

One of the important things about `ping` is that it uses _Internet Control Management Protocol_ (ICMP) packets to test the connection.
This protocol lives on L3 of the _TCP/IP_ model (Network/Internet Layer), which allows our analysis to focus purely on the connection, unlike L4 protocols such as _HTTP/HTTPS_ or _FTP_.
Since ICMP lives on L3, it does not use _TCP_ or _UDP_ to function. It is encapsulated in a _IPv4_ packet.

ICMP is mainly used to understand the _state_ of the transmitted packet. Based on the status of the remote location, ICMP can return messages like:

- `Time-to-live Exceeded (TTL)`
- `Destination Unreachable`
- `Request Timed Out`

---

PS: ICMP being on L3 does not make `ping` a L3 application. `ping` works at L4 (Application Layer) of TCP/IP model.

---

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

## <a id='the-tip-of-the-iceberg'/> The Tip of the Iceberg

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

## <a id='chapters'/> Chapters

Since the analysis is a bit long, I divided it into chapters - you can find them below:

- [Step 1: The Local Network](./1-the-local-network.md)
- [Step 2: DNS Is Only For Humans](./2-dns.md)
- [Step 3: How Is an IP Configured?](./3-ip-configuration.md)
- [Step 4: Where Does an IP Belong?](./4-network-of-ip.md)
- [Step 5: The Journey of a Packet - Part 1 (Host)](./5-journey-on-host.md)
- [Step 6: The Journey of a Packet - Part 2 (Gateway)](./6-journey-on-gateway.md)
- [Step 7: The Journey of a Packet - Part 3 (Hops)](./7-journey-on-hops.md)
- [Step 8: Here Comes the ICMP Response](./8-icmp-response.md)
- [Step 9: The Journey Ends Here](./9-journey-ends-here.md)

## <a id='conclusion'/> Conclusion

Now you will never be able to see `ping` like before!
With that said, we can conclude our analysis.
It was a quite fun ride to gather all these topics in one place.
I would like to encourage you to try all the commands that are used in here.
Also to see some of the protocols with your own eyes, I would recommend installing [Wireshark](https://www.wireshark.org/) to analyze your own network.
It is a popular package analyzer tool and it is what I have been using throughout my own networking journey.
