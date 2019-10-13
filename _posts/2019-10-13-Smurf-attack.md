---
title: "Performing Smurf Attack To Overwhelm Target Network"
date: 2019-10-13
toc: true
toc_label: "Table of Contents"
categories:
  - blog
tags:
  - Network Security
  - Smurf Attack
  - ICMP Spoofing
  - Scapy
---

# Introduction
In my previous [post][dns], we had a look at spoofing DNS responses which allowed us to redirect users to a web server running on our machine. In this post, we will look at spoofing ICMP Echo requests which causes a flooding of the target network. 

# ICMP
Internet Control Message Protocol (ICMP) is a protocol used primarly to report errors (source quench, unreachable destination, etc) in sending IP packets. An ICMP packet has the following packet format.

![pwned](../../assets/images/icmp_format.png)

The two codes relevant to the Smurf attack are:

1. 8 - for echo message, and
2. 0 - for echo reply message

The above messages are commonly known as the ping messages, which are used to test connectivity issues and obtain statistics about the connections of a network. When a host receives an ICMP echo request, it sends back an ICMP echo reply packet to the sender. The sender uses the replies to compute metrics such as the round trip time.  

# Smurf Attack
Well, what if an attacker can control who sends the ICMP echo replies and to whom? Then, the target receives each ICMP echo response from the hosts whom the attacker asks to send the echo replies. If the number of responses are much higher than what the target can handle, it will be overwhelmed and results in a DoS. This is the idea behind the Smurf attack. 

We ask each host on the network to send an ICMP reply to the target by spoofing an ICMP request with the destination as the broadcast address. Each host, on receiving the request, send back a reply to the target. This is an amplification of the traffic, i.e., for one packet sent by the attacker, the target receives *n* packets where *n* is the number of hosts that respond.

The following code can be utilized to start a smurf attack.

```python
pkt = IP(src = g_target_ip, dst = g_broadcast_ip)/ICMP()
while True:
  resp = sr1(pkt, verbose = False)
```
That's it. The above 3 lines of code are all it takes to simulate a Smurf attack. But do note that the success of this attack depends on the number of hosts that respond to the target. If the ICMP response traffic is minute and within what the target can handle, then there will not be any noticable drop in performance.   

# Results
I am running the DNS-hijacker and the web server on a Raspberry PI 3B+ with the target as my iPhone.

![dns-hijacker](../../assets/videos/dns-hijacker.gif)

Sending a request from my iPhone once the DNS hijacker and the web server are started,  results in the following.

![pwned](../../assets/videos/pwned.gif)

The full script is available on my github [page][page].

## Disclaimer
The attacks mentioned on any posts on my blog are to tested at your own risk. Please do not use them without the permission of the target host. I will not be responsible for any damage caused by running any of the scripts available on my website.  

[dns]: https://fsec404.github.io/blog/DNS-hijacking/#results
[page]: https://github.com/venkat-abhi/Smurf-Attack