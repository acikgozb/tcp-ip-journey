# Step 7: Here Comes the ICMP Response

Let's sum up where are we at in this analysis:

- The ICMP packet created by `ping` is sent to the local gateway from our test host.
- The local gateway chose the most appropriate path to `google.com`, and the packet was transmitted along that path.

Now, remember that the last line of our `traceroute` output was our destination IP address.
So, we can say that the _digital signals_ are transmitted to our destination host successfully.

---

PS: Here is a reminder - the physical path of a packet is the responsiblity of L1 (Physical) and L2 (Data Link), thus the destination host first needs to extract the packet from digital signals.

---

So, at this point, here are steps from L1 to L3 on our destination host:

- L1 takes the digital signals and forwards it to L2.
- L2 extracts frames from digital signals, and it also runs a CRC (Cyclic Redundancy Check) to see whether the data integrity is preserved.
- L2 sees that the EtherType header value is IP, so it forwards to IP at L3.
- IP extracts the packet from frames, and checks the destination IP address.
- Once the destination address matches with our destination host's IP address, IP checks the Protocol field and sees that ICMP is responsible from creating this packet.
- IP forwards the packet to ICMP.

As you can see, these steps are the more or less the same steps our gateway used when it received an analog signal from our host.

Here is the part that is different - once ICMP gets the packet from our source host, it sees that it is a Echo packet, and prepares an Echo response packet to answer our host.
At this point, we are back to L3 on our destination host.
This time, the source IP address of the ICMP Echo response packet is actually _our destination address_ `172.217.169.206`, because this time the destination will transmit a packet back to our test host.
But what about the destination IP address?

Remember that our host address `192.168.1.101` is a private IP address, and it is not routable.
To make our packet routable, our gateway translated the private IP address into a public IP address by utilizing NAT.
When we look at the perspective from the destination host, it only knows a packet that was transmitted from that public IP address.
Therefore, the destination host uses the exact same public IP address as it's destination IP address.

To understand the differences clearly, here is a visual representation of the packets we use:

| Packet Type        | Created By      | Source IP Address                | Destination IP Address           |
| ------------------ | --------------- | -------------------------------- | -------------------------------- |
| ICMP Echo Request  | `192.168.1.101` | the translated public IP address | `172.217.169.206`                |
| ICMP Echo Response | `google.com`    | `172.217.169.206`                | the translated public IP address |

From here until the response packet hits our gateway - the steps are pretty much the same.

[Previous Chapter](./6-journey-on-hops.md) | [Next Chapter](./8-journey-ends-here.md) | [Main README](./README.md)
