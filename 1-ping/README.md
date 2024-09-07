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
