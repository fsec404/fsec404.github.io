---
title: "An Introduction to HTTP Parameter Pollution"
date: 2019-08-22
toc: true
toc_label: "Table of Contents"
categories:
  - blog
tags:
  - Web Security
  - HTTP Parameter Pollution
  - Python
---

# Introduction 

Ever seen how your search query is sent to Google? Well, it is sent as a GET request which looks the like the following, if we type in `fsec404` as our query.

``` 
https://www.google.com/search?ei=YrxrXYLDONG1rQHctJ7wBg&q=fsec404&oq=fsec404&gs_l=psy-ab.3..35i39l2j0i13l8.21652.22581..23256...0.0..0.128.806.0j7......0....1..gws-wiz.......0i131j0j0i273j0i67j0i10.Z20sHSUQM2Q&ved=0ahUKEwiC_LHa0q_kAhXRWisKHVyaB24Q4dUDCAo&uact=5
```

We can see that the browser sends a request to `google.com/search` with the parameters filled in automatically. Looking at the whole URL, we can see that the query is sent through the `q` parameter. So, the above URL can be rewritten as

```
https://www.google.com/search?q=fsec404
```

![q-param](../../assets/images/hpp/1.png)

Well, what happens if we change the above URL to `https://www.google.com/search?q=fsec404&q=phreak`?
If we type in the URL in our browser, we see the following.

![q-param](../../assets/images/hpp/2.png)

We see that Google concatenates the two query strings (`fsec404` and `phreak`) and searches for the resulting string. But not all web servers behave in this manner. Let's see how an apache web server handles multiple parameter with the same name by running the following php file.

```php
<html>
<head>
  <title>Search Test</title>
</head>
<body>
  <?php
    if (isset($_GET["q"]) && !empty($_GET["q"])) {
      echo "You have searched for " . $_GET["q"] . "!";
      exit(0);
    }
  ?>
  <form method=get action="">
    Input: <input name="q">
  </form>
</body>
</html>
```

The resulting page is shown in the following.

![q-param](../../assets/images/hpp/3.png)

If we enter in `fsec404`, we get the following.

![q-param](../../assets/images/hpp/4.png)

Now, like before, if we change the URL to `http://osp/websec/search.php?q=fsec404&q=phreak`, we get the following result. 

![q-param](../../assets/images/hpp/5.png)

We can see that only the second instance of `q` is being echoed onto the page. You might wonder what the problem is. Well, consider the scenario where a banking application sends a request, such as the following.

```
insecure-bank/transfer.php?from=ac1&to=ac2&value=100
```

If we change the above to 

```
insecure-bank/transfer.php?from=ac1&to=ac2&value=100&to=attackers-ac
```

When the bank processes the request, the money, instead of being transferred to ac2, is sent to the attacker's account. 

# Parameter Pollution in Social Share Buttons
Another place where HTTP parameter pollution is prevalent is the share buttons on most websites. When we click on the facebook share button, a request is sent that looks like

```
https://www.facebook.com/sharer/sharer.php?u=example.html
```

The `u` parameter specifies the URL of the page we want to share. If we change the original URL to

```
https://example.html?&u=https://fsec404.github.io
```

and click on the facebook share button, if the website does not sanitize the URL, then the following request will be sent to facebook.

```
https://www.facebook.com/sharer/sharer.php?u=example.html&u=https://fsec404.github.io
```

Since facebook's web servers use the last value of the supplied parameter, `fsec404.github.io` is shared instead of `example.html`.

# Examples of HPP in Social Buttons That I have Found 
Testing as many sites as possible over a weekend, I was able to find HPP in 9 websites. One notable site was PayPal. The `stories` directory was vulnerable to HPP and can be seen below.

![q-param](../../assets/images/hpp/pay1.png)
![q-param](../../assets/images/hpp/pay2.png)

I have notified PayPal on Hackerone but was informed that it wasn't a true security threat. Well, I guess I need to look for different bugs.

To automate this process, I have also written a simple script that uses a brute-force approach by checking each link on the user-supplied domain to find instances of parameter pollution. The script is available on my Github [page][page]. Note that it doesn't find links dynamically inserted by javascript. To account for this, I have tried using requests_html package but the package was too slow and kept throwing many errors. If you guys have any suggestions, please let me know. That's it for this post.  


[page]: https://github.com/venkat-abhi/URL-Share-HTTP-Parameter-Pollution-Tester