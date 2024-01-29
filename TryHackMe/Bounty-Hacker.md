## Welcome to my first TryHackMe box documentation! Bounty Hacker is a really straightforward box, which makes it great for beginners! Let's jump straight in! 

# Basic Enumeration

As always, we are going to start with a basic nmap scan to get the lay of the land.

![basic nmap](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c90c53e4-9486-4fcc-9259-0251ba1d8212)


We see we have 3 open ports: FTP (21), SSH (22), and HTTP (80). Let's use a more advanced nmap scan to get additional details about each of these ports.

![detailed nmap](https://github.com/NTHSec/CTF-Writeups/assets/150489159/69e7ffc5-2e9b-48a3-9222-cb5ef653da92)

As shown, FTP anonymous login is _enabled_. Make sure to take note of that, as we might need it later on.


# Web Enumeration

We know that we have an open HTTP port, which usually means there is a website running. Let's check it out!

![Website](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c7c3bfd6-1e12-4486-9edb-8b8d07867d1a)

There isn't really any information here that will help us on the CTF, but it is a cool storyline! 


## Web Enumeration With GoBuster

Since the basic website doesn't have anything of use for us, lets see if we can find hidden subdirectories using GoBuster.

![Gobuster](https://github.com/NTHSec/CTF-Writeups/assets/150489159/0260bffc-3730-42af-8791-6d78c15c1518)

We have an Images subdirectory, lets see what happens when we navigate to that page.

![Image subdirectory](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b295a4fa-75be-4c4f-8db2-ef0fe6edfded)

We can see the image file that's on the website, but outside of that there isn't anything interesting unfortunately. Let's go back to what we learned from the nmap scan and gather some information about the target.


# Info Gathering

We know from the nmap scan that Anonymous login is available for the FTP server, so let's login as such.

![ftp login](https://github.com/NTHSec/CTF-Writeups/assets/150489159/4c2b36a1-efad-4d6d-bf40-f258d6b36233)

You might notice that when you try to run commands like `ls`, you could get a response which says something along the lines of "Entering Extended Passive Mode". To fix this, simple type `passive` into your FTP prompt and you should be able to run commands now!

![turning passive mode off](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c23e097e-9f64-4af3-9090-a8963d1d9247)

We see that we have two files here, `task.txt`, and `locks.txt`. To extract these to our machine, we need to use the `get` functionality in FTP.

![extracting files](https://github.com/NTHSec/CTF-Writeups/assets/150489159/92cb9fa6-82c4-4942-aa87-fc13065b3da0)

Now that we have the files on our machine, lets see what information they contain.

![files extracted to our system](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7fea5569-c680-4173-9c89-7fafa805d405)

task.txt:

![Task](https://github.com/NTHSec/CTF-Writeups/assets/150489159/cf244bf4-2e9f-44d2-b021-3b228339a1c2)



locks.txt:

![locks](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e991f1ef-f122-4fc4-abf6-9bb75af15b47)

So we have a message about "Vicious" from someone named "lin", and what looks like a list of passwords. Going back to our nmap scan again, we had a SSH port open! Maybe we can try to use these credentials to brute force our way into the ssh server!


# Exploitation

I am using a technique called “password spraying” to test every password for both Vicious and lin. Password spraying is a form of brute force attack where I systematically test a list of passwords (stored in `locks.txt`) against multiple user accounts. Specifically, in this scenario, I'm applying the passwords from `locks.txt` to attempt SSH logins for the users "Vicious" and "lin". 

To do this, I am using a metasploit module, but I’m sure there are other tools out there. If you get confused, refer to this [article](https://www.geeksforgeeks.org/how-to-brute-force-ssh-in-kali-linux/).

*Note: If you're using the metasploit module, make sure to set these parameters:*
- set RHOSTS 192.168.XX.XX (vulnerable machine address) 
- set PASS_FILE /root/Desktop/passwords.txt (path of the passwords text file)
- set USER_FILE /root/Desktop/usernames.txt (path of the usernames text file)
- set VERBOSE true (it will show the exactly matched combination of username and password)


So first, let's test the user `Vicious` with our `locks.txt` file as a password list.

![Vicious password spray](https://github.com/NTHSec/CTF-Writeups/assets/150489159/73775070-c9e0-4c88-b200-659d85ff11e1)

Hmmm, no luck. Let's try with the user `lin` now.

![lin password spray](https://github.com/NTHSec/CTF-Writeups/assets/150489159/62b05fdf-ea52-43da-b497-6f5879e520d8)

Nice! We have our SSH credentials! Now let's see what we can do with them.


# Lateral Movement & Privilege Escalation 

First let's SSH in with lin's credentials.

![lin ssh](https://github.com/NTHSec/CTF-Writeups/assets/150489159/1ad4a13a-8185-48e7-9750-e875341d6578)

A quick `ls` shows us that we have found our user flag!

![user flag](https://github.com/NTHSec/CTF-Writeups/assets/150489159/510d688f-f5ed-4b58-9cef-a25939427c0f)

Now it's time to see if we can escalate our privileges and pwn this box!

## Privilege escalation

The first thing to do when you're trying to PrivEsc is to run `sudo -l`. This will show you all the commands the user (in our case, lin) can run as root.

![sudo -l](https://github.com/NTHSec/CTF-Writeups/assets/150489159/1e47761a-9d62-44ba-ba81-824d659657f9)


`tar` is a default linux binary, which means it’s probably exploitable via GTFOBins! 

![gtfo bins](https://github.com/NTHSec/CTF-Writeups/assets/150489159/a1b32b98-6c69-4af0-9638-decea46fd3f5)


We were right! Now let's try out this one liner and see if it works!

![root flag](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ece594df-3659-4e01-9e1e-7224aa8fc7d7)


Indeed it does! We have our root flag.


**Hope you enjoyed this box as much as I did! Most importantly, I hope you learned something along the way!**

Happy Hacking everyone!




