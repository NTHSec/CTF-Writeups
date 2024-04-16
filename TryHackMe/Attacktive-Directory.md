# Attacktive Directory

This is my first ever Active Directory box! It was super fun and relatively straightforward. I used netexec for the majority of this box, which isn't the tools that the THM room originally suggests, but I found it easier this way. Enjoy!

## Enumeration

### Nmap scan:

![11_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/1b80526a-fa43-4ccf-bfa4-3c318a41e12f)

Since this is my first AD machine, all of these ports are pretty foreign to me, so I don't really know what to look for. With that being said, I started with some basic AD enumeration with netexec.

### Active Directory Enumeration

With this command: `nxc smb [IP] -u '' -p ''`, we can check for something called "Null Authentication", which is basically authenticating without a username or password. Looks like we succeed with this command!

![17_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/fe36a819-bfb6-48a6-86ff-ad846bff8a36)

The next step I wanted to take was trying to obtain the shares on the machine with the Null session we authenticated with. Unfortunately, this and some other methods I tried resulted with no success.

![20_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/92966fd8-274b-43c5-a0b4-de0ca8d442b6)

However, we can still move onto enumerating users. Once we have a list of users, it makes enumeration a lot easier because we can be a little more specific and targeted.

We can obtain this list of users by brute forcing the RID's of users due to our null authentication. We can do this by running this command: `nxc smb [IP] -u '' -p '' --rid-brute`

![23_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/0c3b67e6-d398-4714-968e-640391d433aa)

From here, we can make a `users.txt` to make enumeration a little easier on ourselves.

![25_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/67ef7273-7c8e-44d8-bf2c-c2412e8de605)

We saw earlier in the nmap scan that kerberos was enabled on this machine. With this in mind, let's try to using kerberos to authenticate our users list, still with that blank, or null password.

We can do this by running this command: `nxc smb [IP] -u users.txt -p '' -k`

![28_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/9f43b60c-a4ff-46c2-a649-37dc4a4dedbc)

From our output we see that we have one hit with svc-admin! Looks like svc-admin is vulnerable to an asreproast attack. Let’s try it out!

# Exploitation

To learn more about ASREPRoast attacks and why they work, check out this [HackTricks article!](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/asreproast)

To run an asreproast attack, we can run this command: `nxc ldap [IP] -u 'svc-admin' -p '' --asreproast asreproast.txt`

*Note: Make sure to change the mode to ldap!*

After looking at our output, we see that we have obtained the hash for svc-admin!

![34_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/83ecfa8e-651e-4c05-b2e1-4bdc6943522b)

We can simply throw this hash right into hashcat and let it work!

![37_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/406f7e57-f495-4ebe-b9be-ad6109abd603)

Within a few moments, we get our credentials!

![40_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/09a6659a-95fd-4ed1-a906-c2e6b611fd91)

Let's double check that these credentials are right by testing authentication, and it works!

![43_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b99ec7e8-779d-475e-aa2a-8f8a4dce1c4e)

Now that we have authentication, let's check the shares on this machine again.

![47_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/5be1a574-57b0-46f7-ac5e-fb7cd9b4edd8)

The only thing that stands out to me is being able to read the share called backup. Let's use smbclient to login and see if we can exfiltrate any data from this share.

After logging in, we see a text file called backup_credentials!

![52_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/5e583c38-f15d-43a7-8563-34af48a0fe9d)

By using the `get` command, we can download this file to our local machine.

![55_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/4a3d6e25-3ae3-4450-b2fb-138a1b2a7d3a)

Back on our local machine, we find that we have base64 encoded credentials, that we can easily decode. As the file name suggests, we have credentials for the backup user!

![58_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/cf239a76-9185-48fc-9a14-b2abb1f8e60a)

Doubling checking that these credentials work, and yes! We authenticate!

![61_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/39e8ef84-dfe0-4037-8822-0fd8298c5b68)

Using these credentials, we can run a tool from Impacket called secretsdump, which is essentially a python script that tries to dump hashes and other information from a Windows machine. It's similar to getting the information from the /etc/shadow on a Linux machine. To learn more about this script check out this [article](https://medium.com/@benichmt1/secretsdump-demystified-bfd0f933dd9b)!

After running secretsdump here is the output:

![64_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e2d345e8-1255-42ee-84b9-9c112d706f45)

Nice! We have the administrators hash!

## Privilege Escalation

This step is quite easy now that we have the admin's hash. We can simply use Evil-WinRM to login as the administrator!

![67_image](https://github.com/NTHSec/CTF-Writeups/assets/150489159/1f5e9067-7196-43de-9353-bc5efd60b8f8)

And just like that, we’re in, and we have rooted the box!

Overall, this box was a joy to do because all of the new things I learned regarding AD. It got me super excited, and I will be looking to do more AD in the future. Hope you all learned something too, happy hacking!

