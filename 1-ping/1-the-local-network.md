# Step 1: The Local Network

## Table Of Contents

<!--toc:start-->

- [NIC (Network Interface Card)](#nic-network-interface-card)
- [WAP (Wireless Access Point)](#wap-wireless-access-point)
- [Summary](#summary)
<!--toc:end-->

Before diving into what is happening on the network itself, we have to define the "network" first.
This analysis was written on a computer that was connected to a regular home network.

In this "test" network, there are 2 main devices that is responsible from the communication:

- NIC (Network Interface Card)
- WAP (Wireless Access Point)

## <a id='nic-network-interface-card' /> NIC (Network Interface Card)

A NIC is a piece of hardware that connects a device to a network. NICs are mostly found in personal computers, servers and routers.

In this example, the test computer uses its integrated NIC to connect to the wireless home network. As a result, there is an interface on the computer that we can analyze by using `ifconfig`:

```bash
# If your host do not have an Ethernet port, en0 is probably your Wireless interface
# If your host do have an Ethernet port, en1 is probably your Wireless interface
ifconfig en0

# en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
# 	options=400<CHANNEL_IO>
# 	ether xx:xx:xx:xx:xx:xx
# 	inet6 fe80::899:c1a1:7014:1413%en0 prefixlen 64 secured scopeid 0xf
# 	inet 192.168.1.101 netmask 0xffffff00 broadcast 192.168.1.255
# 	nd6 options=201<PERFORMNUD,DAD>
# 	media: autoselect
# 	status: active
```

In our test, `en0` interface of the host is used for any communication through the test network.

The output of `ifconfig en0` tells us many things but for the scope of the analysis, we will focus on `ether`, `inet`, `broadcast` fields in the further chapters.

---

PS: The MAC address of the host (named `ether` in the output) is masked for security purposes.

---

## <a id='wap-wireless-access-point' /> WAP (Wireless Access Point)

A WAP is a device that allows wireless devices to connect to a wired network.
This device is frequently used in home networks (also called modems).

Other than connecting wireless devices, it can also have other functions based on how it is configured.
The WAP on the example (given by the ISP) is used as a gateway (router) and a DNS server.

---

PS: Remember that in a wireless network the main way to communicate is by _sending analog radio signals_. These signals are modulated from digital signals in L1 (Physical Layer), and demodulated once they are transmitted to another station. This is why the name "modem" can also be used for WAP's.

---

## <a id='summary' /> Summary

Okay, so to recap:

- The "test" network we have uses 2 devices: A computer and a WAP.
- The WAP is also configured as a gateway (router) and a DNS server.

Now that we defined what our "network" is, we can proceed with the analysis.
As we progress through, the terms "gateway" and "DNS server" will be more clear.

[Next Chapter](./2-dns.md) | [Main README](./README.md)
