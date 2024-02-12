**This is another Easy level THM box named 'Brute It' (I wonder what kind of attacks we will be learning about 😉)! Let's dive in!**

# Enumeration
As always, we are going to start with a default nmap scan to get the lay of the land.

![nmap scan](https://github.com/NTHSec/CTF-Writeups/assets/150489159/dabca070-5961-46c1-ab1f-3c14b30e7931)

We see we have port 22, and port 80 open. Let's start by checking out the website on port 80.

![website](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f895203c-ceca-485e-abc9-21519a9edf7c)

Hmmm... it looks like a default apache page. When we don't have any leads on a website, it's a good idea to use a tool like GoBuster to find hidden directories. 

![gobuster](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f0e8efd4-fd4e-4c1d-a606-dc6aedc098fe)



# Info Gathering

Let's check out the admin page we found!

![admin page](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d4c2b9b6-bf7e-45aa-a558-cc3995fc20da)

Looks pretty simple, but we really have no other leads. I wonder what else we can grab from this website. By pressing  `CTRL + U`, we can examine the source code of the website.

![source code](https://github.com/NTHSec/CTF-Writeups/assets/150489159/9ec3f0b6-aaa1-4ba7-9020-b3edf255ed2c)

Look at that! What a nice comment, telling us the username is admin!


# Exploitation
Based on the information we have, we can brute force the login page with the username ‘admin’, but how?

The first thought that came to my mind was to use hydra, which is a common tool for password-cracking.

To use hydra, we need to first identify a couple of things. Firstly, we need to capture a login attempt with Burp Suite. This will allow us to see how the server is dealing with logins.

![burp intercept](https://github.com/NTHSec/CTF-Writeups/assets/150489159/58aee07e-b40b-47c3-b74c-c1d6e98a5f11)

We see that this login page is using POST requests. This is useful to us because we now know we can use hydra’s `http-post-form` functionality to brute force this login page.


## Hydra 
Let's break down our command:
```
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP_ADDRESS> http-post-form "/admin/:user=^USER^&pass=^PASS^:Username or password invalid"
```
- **`hydra`**: This is the start command of using hydra
- **`-l admin`**: This specifies the username to use during the password-cracking process. In this case, it's set to "admin".
- **`-P /usr/share/wordlists/rockyou.txt`**: Here, we're specifying the path to a text file containing a list of potential passwords. In this example, it's using the "rockyou.txt" wordlist, which is a commonly used wordlist for password cracking.
- **`<IP_ADDRESS>`**: This is where we'd put the actual IP address of the target system we're trying to brute force.
- **`http-post-form`**: This indicates the type of form-based authentication you're trying to crack. It's specifically for HTTP POST forms, which are commonly used in web applications for login pages.
- **`"/admin/:user=^USER^&pass=^PASS^:Username or password invalid"`**: This part is specifying the login form parameters and the error message to check for. This is why we had to intercept the burp request, we are using the **path, body, and error message** we got from burp. It's telling Hydra how to interact with the login form. In this example, we're saying that the login form is located at the "/admin/" subdirectory and has parameters for the username ("user") and password ("pass"). We are also specifying the error message that indicates an invalid login attempt.
    - Notice how I replaced my initial test parameters (’admin’, ‘password’) with `^USER^` and `^PASS^` . This tells hydra that we are going to use the username we specified with the **`-l`** option. We then alter the password value and change it to ^PASS^ so hydra knows to fuzz this value with passwords from our password list. Also, make sure you put semi-colons(:) in-between your arguments in your command (path, body, error message)!

In simpler terms, this command is essentially telling Hydra to attempt to log in as the username "admin" using a list of passwords from the "rockyou.txt" file, targeting a specific IP address, and using a specific form on a web page. It will keep trying different passwords until it either finds the correct one or exhausts the list of passwords.

Phew! That was a lot! Now back to the fun stuff! Let's run this command and cross our fingers...

![hydra command](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b4801a6d-6c41-4b40-a97f-95818f2815d8)

Nice! We have our password for the login page!

After entering this credential into admin website, we pull up this page, where we find our first flag:

![website flag](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f3b7cbf3-7f2d-4866-8ca9-692b218669e2)

The website also gives us john's RSA private key. This is something we can use to SSH into the server.

When we click on the hyperlink, it brings us to the private key! Let’s now use this to ssh into the server as the user 'john'.

![rsa key](https://github.com/NTHSec/CTF-Writeups/assets/150489159/cf5bce52-e08b-43db-b66b-6a46e02c2763)

## SSH
First we need to create an id_rsa file, putting the key we just found inside of it. I used `nano` as my text editor of choice.

![id_rsa file](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f4b7202a-409e-438f-b3c3-316cc693ee93)

Let’s give our id_rsa file permissions of 600 and try and ssh in!

*Note:* using **`chmod 600`** on an **`id_rsa`** file ensures that only you, the owner of the file, have read and write access to it, thereby enhancing the security of your SSH key pair. In this case, we want the SSH server to believe we are john, the owner of the id_rsa file.

![id_rsa password](https://github.com/NTHSec/CTF-Writeups/assets/150489159/4463f974-5b17-468d-996a-57b1db0a3301)

hmmm… seems like it’s asking for a password for our key. Looks like we might need to use brute force again to get this passphrase!

## Cracking the RSA key
This may sound intimidating at first, but we can crack RSA keys super easily in two steps:
1. Make our RSA key into a readable format for john using `ssh2john` (the extension doesn’t matter)
   
   ![ssh2john](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b801c97c-c8ca-4579-8d18-b97abe84dd86)

2. Crack it with any wordlist of your choosing! I used rockyou.
   
   ![id_rsa cracked](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c3252797-59ea-4b91-aca2-e4bb8d5c2334)

We have our RSA key password! Let's try to SSH in now with our new credentials.

![john ssh](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2a89da91-29df-41ad-8669-b2f7672bca10)

We're now SSH'd in as john! Let's see what we can do.

# Lateral Movement / Privilege Escalation

Our first step is to get our hard earned user flag:
![user flag](https://github.com/NTHSec/CTF-Writeups/assets/150489159/72c9ecbd-7f65-445a-b63a-5506bc6b51e7)

Now let’s try and run the bread and butter of PrivEsc, **`sudo -l`**

![sudo -l](https://github.com/NTHSec/CTF-Writeups/assets/150489159/a6e7772c-fe3b-428e-9b96-76cc83e8b515)

Looks like this is gonna be an easy box to root because 'cat' is a binary file! This means we’re most likely going to be able to use GTFObins to escalate to root!

Here is our [GTFObins](https://gtfobins.github.io/gtfobins/cat/#sudo) Sudo command!

![gtfobins](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f0c84ff4-9dd4-4e7b-84a4-4f5698b387d6)

This command wont give us root directly because the ‘cat’ binary is used to read files. This is helpful to us though, as now we can read sensitive files such at the /etc/shadow file.

*Note: The `/etc/shadow file` is a crucial component of user authentication and security on Linux and Unix-based systems. You can find sensitive data such as hashes and passwords when reading the file.*

![etc shadow](https://github.com/NTHSec/CTF-Writeups/assets/150489159/caa37378-66c8-4760-8bd6-a93c3d997282)

Looks like we have a hash for our root user! I think we all know what we can do next...

## Cracking the root hash with john

In our situation, we've got a two-step process: unshadowing and cracking. Unshadowing kicks things off by merging the /etc/passwd file with the /etc/shadow hash. This combo gives our buddy John the context he needs to make sense of what's coming his way.

1. Getting the /etc/passwd line:

![passwd](https://github.com/NTHSec/CTF-Writeups/assets/150489159/18fd18ca-bbbd-41b7-9ac9-533242e9cace)


2. Getting the shadow line:

![shadow line](https://github.com/NTHSec/CTF-Writeups/assets/150489159/27b7712e-924c-45ba-9a61-b72543fcd5fa)

Now we have our two files ready to go into the `unshadow` command!

![files to go](https://github.com/NTHSec/CTF-Writeups/assets/150489159/febe546c-1d74-4659-98d7-4c66f69d0d50)

Let's use the `unshadow` command on our two files and output them into a new file called `unshadowed.txt`.

![unshadow](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f3b9598c-4312-435d-925d-688190911ff4)

Now that we have our hash in a formate john can read, let's see john take a crack at it!

![shadow cracked](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ac3da11f-d84b-4042-b932-9d3fd2986f1d)

Awesome! There is our root password!

Now all we need to do is use the `su` command to switch to the root user, enter our password and now we have root!

![root](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8df67883-4afd-452e-9ef6-e3e6183ec9a8)


*Ending note: This box was one of my favorites I've done so far! I learned a ton about brute forcing using both hydra and john in this box! Before this, I never knew you could brute force RSA keys! I hope that you shared my learning, and you enjoyed the box as much as I did!*
