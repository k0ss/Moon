---
layout: post
title:  "FirstClass 0day Release (Part 1)"
date:   2011-04-05
excerpt: "It all started when I learned that you could create a settings file for someone by having them click on a specially formatted FCP:// URI link."
tag:
- firstclass
- opentext
- email
- logic
- fcp
comments: true
---
# Intro
Hello all,and welcome to my blog.  I am new to the full-disclosure world, though I have spent a lot of time finding exploits in the past.  First of all, thank you very much for reading my first article. I hope you find it informative, and if you have any questions, feel free to leave me a comment.

Today I am releasing an exploit I found three months ago in the First Class email client created by the company OpenText.  This exploit allows remote code to be executed against anyone using the FirstClass system, with a very small amount of user interaction.  The code would be executed with the same privileges as the user who launched FirstClass.  It could be tweaked so that it affects all users on Windows, but then it could have broken the exploit if the current user didn’t have administrator privileges.

# About the Bug
It all started when I learned that you could create a settings file for someone by having them click on a specially formatted `FCP://` URI link.  It would look like `FCP://user:password@fc.server.tld;settings.fc` and would produce a `settings.fc` file in the application's `settings` folder that looked like this in a hex editor:

![FC Settings File]({{ site.url }}/assets/img/blog0_fcfile.jpg)

### Unsanitized User Input in File
From that, we can see that we can set a username and password, both of which would be written into the settings file.  The password is not put into the settings file in plain-text for obvious security reasons, but the username and server name is.

### Arbitrary File Name
We can also see that the server name is ended with a semicolon, and what follows is the desired name of the settings file.  I played around with this part the most, and found that directory traversal is possible, so something like `;../settings.fc` would escape the current folder used to store the settings files, and put it in the folder above it.  I tried to change the name of the file to `settings.exe` as a test.  I had no luck since FirstClass checks to make sure that the file name ends in `.FC` and if not, it appends `.FC` to the file names for security and functionality purposes.

### File Name Filter Bypass
While experimenting with different file names, I tried making increasingly large settings file names, and noticed that with a certain length, the `.FC` extension that FirstClass tries to append was not being appended!  So next, I gave the settings file a name just long enough to fit the filename along with the extension `.exe`, and then I appended `.fc` to the end of it to satisfy FirstClass and its extension check.  To my excitement, the `.exe` file extension stayed, so I now could control the extension/type of file created by FirstClass.

So now that I knew that the username and server name are stored in plain-text within the settings file, along with how I could set the file extension to whatever I wanted and use directory traversal to place it wherever I wanted, I tested creating an arbitrary file outside of the main settings directory.  Here is the link I used to trigger the vulnerability and write an executable file to another directory:

```
FCP://%22%0d%0acalc%0d%0a@fc.server.tld;..\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.bat
```

This just pops the calculator up, and it isn't very stealthy as you will get a large command prompt which opens before the intended program (calc) opens.  I'll improve the exploit in the next post.

# Disclosure Process
Prior to deciding to publish the vulnerability and how someone can exploit it, I contacted OpenText twice via every email address they’ve given on their website, as well as some default security/bug email addresses that most companies have set up (security@[company_name], etc).  I gave them a week to respond each time, and they chose not to, so I've chosen to release it since this was pretty easy to find.
