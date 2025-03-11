# Step 9: The Journey Ends Here

Unfortunately, we are coming to the end of our analysis, and this chapter is not as exciting as previous ones due to the fact that the same concepts are used on our way back.
So, let's quickly take a look at what happens when the ICMP response hits our gateway:

- The process between L1 and L3 of our gateway is again the same - the digital signals are eventually converted into a packet that L3 can process.
- L3 IP checks the destination IP address of the response and re-translates it back to our host's private IP address by looking on its _translation table_.
- Once the private IP address is determined, our gateway understands that it is a host that lives on the same network - so it uses either its own ARP table or sends an ARP broadcast message to get the MAC address of our host.
- With the MAC address in hand, the same steps are taken for L3 -> L1 on our gateway, and the analog radio signals are sent for out host at the end.

And here is the last part on our host:

- Analog radio signals are picked up by L1, demodularized into digital signals, and forwarded to L2.
- L2 extracts frames from signals, runs it's own CRC to check the data integrity.
- L2 hands the frames over to L3 IP, and the packet is extracted from frames.
- IP checks the destination address and understands that the packet belongs to the host itself.
- IP hands over to ICMP (by looking at the Protocol header value) and ICMP understands that this is an Echo response coming from `google.com`.
- At the end, `ping` reads the ICMP response and outputs it to stdout.

The ICMP response we are talking about is the very first output we looked at in this analysis:

```bash
ping -c1 google.com

# PING google.com (172.217.169.206): 56 data bytes
# 64 bytes from 172.217.169.206: icmp_seq=0 ttl=56 time=26.257 ms
#
# --- google.com ping statistics ---
# 1 packets transmitted, 1 packets received, 0.0% packet loss
# round-trip min/avg/max/stddev = 26.257/26.257/26.257/nan ms
```

Like I said, if you read the previous chapters there is nothing new on this one, but we had to take a look at the response as well to see the whole picture.

[Previous Chapter](./8-icmp-response.md) | [Main README](./README.md)
