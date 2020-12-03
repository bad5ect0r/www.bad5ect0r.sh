+++
title = "UTCTF 2020 - Chatt With Bratt"
date = "2020-03-09T00:00:00+11:00"
author = "bad5ect0r"
authorTwitter = "bad5ect0r" #do not include @
cover = "/utctf-2020-chatt-with-bratt/cover.png"
tags = ["ctf", "utctf", "xss"]
keywords = ["xsshunter", "blind-xss"]
description = "Send XSSHunter payload to a chat bot to read their cookie."
showFullContent = false
+++

The application sets two cookies:

* `chat_id`: UUID used to identify the chat log currently in use.
* `secret`: The one that potentially holds the flag on Bratt’s side.

![Cookies being set](/utctf-2020-chatt-with-bratt/cookies.png)

From this, we can assume that our goal is to get Bratt’s secret cookie.

My first thought was to exploit XSS. A quick smoke test on the application confirmed that it was vulnerable to XSS:

{{< figure src="/utctf-2020-chatt-with-bratt/xss-test.png" caption="&lt;img src=x onerror=alert(1)&gt;" >}}

Now that I knew that triggering XSS was very simple, I tried using an XSS hunter payload and a little while later, we see that Bratt got hooked:

![Got a hit on XSS hunter](/utctf-2020-chatt-with-bratt/xss-hunter-hit.png)

![Got the Bratt's secret cookie](/utctf-2020-chatt-with-bratt/xss-hunter-cookies.png)

The flag was: `utflag{95debad95cfb106081f33ceadc36bf9c}`