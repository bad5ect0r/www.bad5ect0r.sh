+++
title = "Stored XSS Leads to Plaintext Password Disclosure"
date = "2020-05-17T00:00:00+11:00"
author = "bad5ect0r"
authorTwitter = "bad5ect0r" #do not include @
cover = ""
tags = ["vdp", "webapp", "xss"]
description = "Uploading a malicious HTML file to the web application to get XSS and decode a sensitive cookie."
showFullContent = false
+++

# Introduction

This post documents another vulnerability I found in the same company as the last post. I disclosed both vulnerabilities on the same day and was told that both vulnerabilities have been resolved. I was authorized to provide a partial disclosure, hence why I’m writing this post now.

# The Story

So once I had logged into the mobile app, I had a look around and noticed a profile image upload feature:

![Can upload a profile picture. Please ignore the missing image icon.](/stored-xss-leads-to-plaintext-password-disclosure/image-upload-on-app.png)

I used Burp to intercept the request sent when uploading a profile image and saw something like this:

![PATCH request used to upload profile image. This was taken after exploiting the issue. Originally the `ProfilePicture` parameter contained base64 encoded image data and the `ProfilePictureMIME` parameter contained `image/png`.](/stored-xss-leads-to-plaintext-password-disclosure/patch-request-example.png)

Basically they are taking the image data, converting it to Base64 and then sending it via a standard JSON OData API request.

The most important thing you should notice is that there is a `ProfilePictureMIME` parameter. This essentially dictates the file type that is being uploaded.

I tried changing that to `text/x-php` and `application/php` but they were not allowed. Surprisingly, `text/html` was allowed. I then created a basic HTML file that pops an alert:

```html
<!DOCTYPE html>
<html>
    <body>
        <h1>Test</h1>
    </body>
    <script>
        alert(1);
    </script>
</html>
```

I converted this to Base64 and replaced the value of the `ProfilePicture` parameter in the request with my base64 encoded HTML file. I set the `ProfilePictureMIME` to `text/html` and sent the request. When I used my browser to check the raw file to my image, it actually rendered the HTML! Unfortunately I do not have screenshot of this.

So then I thought of a practical attack scenario. I realized that there was no authentication required to access the raw profile image files from their server. I then realized that the domain hosting these profile images was also the same domain as the one hosting their web application:

Link to image file: https://something.redacted.com/res/img/usermeta//551/USER_7025dffcf32e4097bebe7b530f9f1a5d.png?ts=1584857339
Link to web application: https://something.redacted.com/login/

Using the credentials I found before, I logged into the web application and tried visiting the link to the HTML profile image I uploaded. I could access it.

So then I examined any cookies I could access using JavaScript. One of these was called `AUTHH`. It was base64 encoded, so i decoded it using CyberChef and realized that it was the same value as the Authorization header! Since they use HTTP Basic Authentication, the credentials are in plaintext!

```
Cookie: AUTHH=QmFzaWMgWm1GclpUcG1ZV3RsY0dGemN3PT0=
```

I immediately uploaded the following HTML file:

```html
<html>
    <head>
        <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>
        <script src="https://cdn.jsdelivr.net/npm/js-cookie@rc/dist/js.cookie.min.js"></script>
    </head>
    <body>
        <h1>Password Display by bad5ect0r</h1>
        <p>Your username is: <span id="uname"></span></p>
        <p>Your password is: <span id="pass"></span></p>
    </body>
    <script>
        $(document).ready(function () {
            const AUTHH = Cookies.get('AUTHH');
            const unb64 = atob(AUTHH);
            const basic = unb64.split(' ');
            const uname_pass = atob(basic[1]).split(':');
            const user = uname_pass[0];
            const pass = uname_pass[1];
            $('#uname').html(user);
            $('#pass').html(pass);
        });
    </script>
</html>
```

This basically displays the user’s username and password by decoding their AUTHH cookie. A real attacker would just forward this information to their server after getting a user to click on the link to their malicious profile image.

I reloaded the link while still authenticated to the web application, and it worked like a charm!

![Viewing that link would disclose your username and password.](/stored-xss-leads-to-plaintext-password-disclosure/password-display.png)

I immediately sent out another email to the company alerting them of this new vulnerability and while there was some initial confusion on the severity of this bug, they were able to prioritize getting it fixed.

I look forward to finding more bugs on this company’s platform and hope that one day they move to a rewards based system to encourage repeat hackers.

# Takeaway

If you can’t upload a web shell, try the next best thing, an HTML file to get stored XSS.

# Disclosure Timeline

| Date | Details |
|------|---------|
| 21/03/2020 | Issue was reported to the company. |
| 25/03/2020 | Follow up. |
| 27/03/2020 | Acknowledged by the company. |
| 03/04/2020 | Issues were fixed. |
| 15/05/2020 | Partial disclosure was authorized. |