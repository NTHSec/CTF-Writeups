# tomghost

This box from THM presented multiple awesome vulnerabilities, including an LFI vulnerability on port 8009 which leaked SSH credentials, and weak encryption on a PGP key. Exploiting this known exploit, we can able to gain foothold access through SSH. Through this foothold, we find and crack a weakly encrypted PGP key to obtain additional credentials. Lateral movement was achieved through SSH access, and privilege escalation to root was accomplished using the zip binary with sudo permissions.

## Enumeration

### Nmap scan:

![60_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b431eaf6-f5fe-495d-acb9-6151246db15e)

Definitely some interesting ports here. The versions also are calling my attention. Let’s check out the Apache Tomcat website:

As expected, it is simply a default page.

![64_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/bde5addf-c802-445a-8f4f-0982a0f494ef)

Navigating to the manager tab, it reveals some credentials we might be able to use:

![67_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2672cf14-f9f4-457e-937d-91ece79ea06b)

## Exploitation

So we have potential default credentials, but I want to go back to that interesting port we saw earlier - port 8009. After more research I found this [exploit](https://github.com/00theway/Ghostcat-CNVD-2020-10487?source=post_page-----23a9a1ae4a23--------------------------------)

To summarize this exploit, **Ghostcat** is was big problem for Tomcat servers found by security experts. It happens because of a mistake in how Tomcat communicates. This mistake lets attackers read or use any files in the Tomcat website folders through LFI (local file inclusion). For instance, they could see secret website settings or even the code that runs the website. If you're looking for more information on this vulnerability, you can check out this [article](https://www.chaitin.cn/en/ghostcat)

![74_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/89254101-b243-45b3-baca-5326f8f01cfd)


After running the command we get read access to this page, and even better we get credentials!

The box maker must have been angry at the sky or something when making this box.

![77_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/9e41794f-2d62-47d0-8e0f-5d9748ec7de2)

## PrivEsc / Lateral Movement

Now that we have set in stone credentials, let’s ssh into the box.

We’re in!

![7_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/91f3fa79-ae6e-4629-bebc-dde914f7a92b)

We got the user flag!

![10_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/94c5d8f3-58d1-4554-8e1c-76cb8d41a150)

Our next step is to try and escalate our privileges. Unfortunately though, we don’t have permission to run `sudo -l` as the user sky****.

![13_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/366eb202-c4ad-4b0a-8c30-88aa313ff186)

In sky****'s directory, we find some binary text, and a PGP private key.

![16_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/82f73a01-e29d-4b74-86bb-a82911343ed2)

Before we jump into this though, I want to try and run linpeas on this machine.

`Victim machine:`

![22_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/52cd5b86-e3d7-457a-98ca-4fb71a5aa4c7)

`Attacking Machine:`

![25_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b2900714-5c6e-4f46-89de-06e6f2b0586c)

Now we have linpeas! Let’s give it executable permissions and run it!

![28_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/d8126e42-75af-4e9d-86d5-b47f84634067)

We couldn’t find anything with linpeas, so let’s go back to the pgp key.

I found this helpful [article](https://blog.atucom.net/2015/08/cracking-gpg-key-passwords-using-john.html) about cracking pgp keys with john. So once we get the .asc file on our local machine (using the same process as before), we can follow the blog and start cracking.

The process is rather simple. First, we want to translate the pgp key into something john can read. So we use gpg2john. After that, we can simply run john and feed it the key file with the rockyou wordlist and it almost cracks instantly.

![36_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/9c318d1a-09ff-4f18-a046-1cf2f6c1be07)

So this password that we found is the *key* we need to decrypt the pgp file we found on the box! Let’s do that now!

First we need to import the asc private key, then we try to decrypt the credential file with the password we obtained and…. there we go! We have the credentials!

![41_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/83b5f6ec-809e-4ee8-ad3b-2691859e0633)

Let’s su to merlin and poke around!

Trying a quick `sudo -l` we find something interesting!

![47_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/ad5ce9a0-9d92-4bea-bcec-dfb22db5597a)

merlin can run the zip binary as root! Here is a GTFObins [article](https://gtfobins.github.io/gtfobins/zip/#sudo) about that binary!

![50_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f64471fb-0232-44da-988d-96b95288af31)

This can be a little confusing at first, so let's break it down.

1. `TF=$(mktemp -u)`: This creates a temporary file name (TF) that's unique. Think of it like creating a placeholder for something.

2. `sudo zip $TF /etc/hosts -T -TT 'sh #'`: This command uses the zip program with elevated privileges (sudo) to create a zip archive. It puts the /etc/hosts file inside the zip archive ($TF). The -T and -TT options are used to test the integrity of the zip file, but they're abused here. The 'sh #' part is where the trick happens. It tells zip to include the string 'sh #' as a comment in the zip file. But because of a vulnerability, this string is interpreted as a shell command.

3. `sudo rm $TF`: This deletes the temporary file we created earlier, erasing our tracks.

So, in simple terms, this command tricks the zip program into running a shell command ('sh #') by including it as a comment in the zip file. Because the zip program is running with elevated privileges (sudo), the shell command also runs with those same privileges, allowing the user to escalate their privileges and gain root access.

Let’s run these commands and get root finally!

![54_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/201d91b8-032b-4cfd-bdb5-7f063ba62035)

And we obtain our root flag!

![57_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/30c1ea7a-66d1-451b-84e9-ed4277680759)












