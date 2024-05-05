# Cybros CTF
This is the first ever CTF I've ever made, so it's interesting doing a writeup on it, but here we are!

## Enumeration

### Nmap Scan:
![Nmap](https://github.com/NTHSec/CTF-Writeups/assets/150489159/426cfc1a-0cc6-4243-9811-8681f9b3173d)

In our initial nmap scan we only have 2 ports open - 21 (FTP), and 22 (SSH). Notice the flags I added to my command:
- `-sC` to run default scripts
- `-sV` to enumerate versions
- `-Pn` to avoid the firewall blocking nmap's probes.

We also see that in our FTP server **Anonymous Login** is allowed. This is interesting, and typically a red flag in CTFs, so it will be our next path of attack.

## Information Gathering

### FTP Server
Knowing that we can login to FTP as **Anonymous**, let's see what info we can gather from the server.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2ea259f4-a7ec-4120-85cc-aacf4bb989e3)

*Note: Make sure to run the `passive` command so that your commands run without issues.*

A quick `ls` shows us that there is only one file in this directory called **message.txt**. We can use the `get` command to download this file onto our local machine.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7e5f7539-6d2a-429d-86d4-80d0f4b582e5)

We can `cat` out the file and see that it is a message for someone named **Cybro3**. Something about an SSH key and how it is hidden on the server.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2f26eb2e-0b8e-4603-98d3-c962d74829a5)

When I think of hidden files, the first thing I think of is running `ls -a`, which stands for _all_. If we go back to our FTP server and run that command, we see that we have a hidden directory called **supersecretfolder**!

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/25b7bc70-3957-414b-9cbd-edb637751415)

Navigating to that direcotry, we find a file called **id_rsa**, which is the SSH private key we were looking for! We can use the same `get` command seen earlier to download this file onto our local machine.

## Initial Foothold

### Giving permission to the SSH key
We can't use the newly obtained SSH key just yet. We have to give it the correct permissions in order to trick the SSH server into thinking the key is ours. To do this, we simply have to give the id_rsa file owner permissions. The command for this is `chmod 600 id_rsa`.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f6c12781-c96f-4db1-b91a-42d2d308bcae)

### Logging into the SSH server
Now that the key has the correct permissions, we're ready to SSH into the server! We know that a possible username is **cybro3**, since that's who the message in the FTP server was addessed to. All we have to do is add the key-login syntax to our default SSH command and we're good to go! To do this, our command looks like `ssh cybro3@{IP} -i id_rsa`.

Notice the `-i id_rsa` - this where the magic happens. Here, we're telling the SSH server, "Use this specific private key ('id_rsa') as the token to gain entry." This is why we had to give the key 600 permissions earlier. We are telling the server that we (as **cybro3**), have ownership of the key, therefore we can authenticate with it.

We can also grab the first flag!
![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2b4e06b4-f852-44e1-a923-a25c2907ad6e)


## Lateral Movement

### Finding the hidden backup file

In the home directory of **cybro3**, I ran a quick `ls -a` and noticed that there was a .backup directory. This may look like a normal directory at first, but it isn't! Navigating to that directory we find a backup file that contains the password hashes for **cybro3** and **haik**.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/926657f2-d96c-4509-9b47-1dcd34a0eb34)

### Cracking the Hashes
To crack the hashes, I elected to use hashcat, but using john is perfectly fine too. I know by looking at these hashes that they are SHA256, but we can double check with a hash identifier tool like [this one](https://hashes.com/en/tools/hash_identifier) 

I put the hashes into a separate file called **hashes.txt**, and used hashcat in autodetect mode.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ad5f1b63-bffe-488c-ac1b-a91e7e4af3c7)

We're going to have to specify a mode, and I am going to choose 1400. Running the command again with `-m 1400` and the `rockyou.txt` wordlist, the hashes crack instantly and we get our passwords!

*Note: The full command I used here is `hashcat -m 1400 hashes.txt /usr/share/wordlist/rockyou.txt`*

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/524dafbd-3690-4510-9f53-8eaa9c6b2d6d)

Now that we have **haik's** password, we can use the `su` command to switch to him. Navigating to **haik's** home directory, we obtain flag 2!

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/54edbd98-2fbb-4c44-9dbc-288485bd3152)


## Privilege Escalation

Now that we are in a higher level user, let's try to run `sudo -l`, which will tell us all of the commands that the user **haik** can run on the machine as root.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/9deb0231-82ed-4df4-bbee-e57598c76e77)

It looks like we can run one command, that being the `cat` command. The `cat` command is really fundamental in UNIX systems and doesn't seem too powerful at first, but we can do a lot with this! Since we can now read files we typically can't read, my first thought gravitates to the `/etc/shadow` file, which contains hashed passwords for every user on the system.

We can combine using the `unshadow` command in tangent with **john the ripper** to crack the root hash! To read a more detailed walkthrough of this step, refer to this [article](https://erev0s.com/blog/cracking-etcshadow-john/).

If we run the command `sudo cat /etc/shadow` we get all of the hashes, but most importantly we get the entry for the root user, who is our goal.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d42f652c-a242-4034-b294-a9a01312a134)

We can also output the `/etc/passwd` file and put both of the root entries into files respectively. 

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/17ebcba9-eb42-4fde-8453-ddca7de2b6e3)

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ad9ddfb3-4c16-4cde-9f6d-c0c08ebf24f9)

After we have the root entries in **passwd.txt** and **shadow.txt**, we can run it through the `unshadow` command and output it to a file called **unshadow.txt**, which will essentially combine the two files and make it readable for john so we can crack the hash.

Running john with the **rockyou.txt** wordlist, we can obtain the cracked hash within seconds!

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/8bf7dc1e-a321-4cc9-9a35-e26781d5e2e1)

Finally, we can use `su root` to switch to the root user and grab the root flag.

![image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/30d42e07-21e0-46f3-8f4e-e723792649b6)

Hope you enjoyed the CTF, thank you so much for playing!


