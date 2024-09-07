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
# -n1 is used to ping only once, there are no limits by default.
ping -n1 google.com
```

Please note that `ping` can be used to test the TCP/IP configuration on a host by using a loopback address (commonly known as `localhost`):

```bash
ping -n1 127.0.0.1
```

Keep in mind that using the loopback address does not create a packet that traverses the network.

##
