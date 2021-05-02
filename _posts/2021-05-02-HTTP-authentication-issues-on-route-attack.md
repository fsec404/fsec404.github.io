---
title: "A Look at a Simple On-Route HTTP Content Manipulation Attack"
date: 2021-05-02
toc: true
toc_label: "Table of Contents"
categories:
  - blog
tags:
  - Network Security
  - HTTP
---

# Introduction
I was recently looking into Transport Layer Security (TLS) and came across an explanation of [secure web browsing][computerphile_url] by Dr. Richard Mortier on the YouTube channel Computerphile. In the video, Dr. Mortier talks about how the lack of authentication of servers can allow the nodes between the client and the server to inject their own content in place of the user-requested data. In this post, we will look at a demonstration of such an attack.

# On-path HTTP Attack
There are numerous ways an attacker can force the client to themselves a part of the route to the webserver. In this post, we assume that the client is connected to a proxy server under the attacker's control. This situation can be the case of many corporate networks where clients are allowed to fetch content through proxy servers only.

I have written a simple HTML website that displays the following page. 

![Original-Site](../../assets/images/http-onpath/original_site.png)

To inject my own content, I have written the following simple script using mitmproxy's APIs. 

```python
from mitmproxy import http

def request(flow: http.HTTPFlow) -> None:
  if "foodblog.com:8000/images/cookie" in flow.request.pretty_url:
    fd = open('images/salad.jpg', mode='rb')
    replacement_img = fd.read()

    flow.response = http.HTTPResponse.make(
      200,
      replacement_img,
      {
          "Content-Type": "image/jpg",
          "Content-Disposition": "inline"
      }
    )
```

All the above script does is respond to any request for `/images/cookie` with an image of a salad. The following video shows how the attack looks to the client before and after the attack.

{% include video id="1EB4Q2RBTwHLBO4Zo-c8baoeboRnU8Fsc" provider="google-drive" %}

We can clearly see that image of the cookie is being replaced with an image of a salad. In this manner, any content can be easily replaced, including the entire web page, as the attacker has complete control of deciding what responses are sent back to the client. 

[computerphile_url]: https://youtu.be/E_wX40fQwEA
