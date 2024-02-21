# THM Ignite - CTF Writeup

Another Easy box from THM! This one taught me a lot about not overcomplicating things and keeping it simple. I hope you enjoy!

## Enumeration

As always, we are going to start with a default nmap scan.

![nmap scan](https://github.com/NTHSec/CTF-Writeups/assets/150489159/96cf0856-20b1-4602-b126-a3585d20d8bb.png)

This box is pretty straightforward, as we only have one port open - port 80. We know that port 80 typically goes to HTTP, so let's check out what website is waiting for us.

Here is our website:

![Fuel CMS](https://github.com/NTHSec/CTF-Writeups/assets/150489159/00f15f2e-e849-4e97-bfab-7f4defb85e61.png)

Hmmm, Fuel CMS v1.4. I am sensing that this version might be exploitable.

Let’s explore this website some more and poke around. I am also going to run a gobuster scan in the background.

Here are our results from the gobuster:

![gobuster](https://github.com/NTHSec/CTF-Writeups/assets/150489159/29201f47-5d69-41f3-a945-aeff72a27cff.png)

Nothing too interesting, all these subdirectories simply redirect to the page we're already on. Fortunately, if we scroll down the page a little further, we find this nice note about logging in as admin!

![admin note](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e5326dc0-9d82-4717-b95b-638104ac4efc.png)

Let’s go to this page, and it is indeed an admin login page!

![admin login](https://github.com/NTHSec/CTF-Writeups/assets/150489159/242d2d49-68c8-4851-89c1-1e0b75bb42d4.png)

Let’s enter the credentials we found (admin, admin)!

We’re in! That was easy!

![admin panel](https://github.com/NTHSec/CTF-Writeups/assets/150489159/b96444c5-e924-41e0-bd32-fe40a0834873.png)

Let's explore the admin page some more.

## Info Gathering

On the admin page, it looks like there is file upload functionality. Maybe we can upload a reverse shell onto the website?

![file upload](https://github.com/NTHSec/CTF-Writeups/assets/150489159/f6eb26f4-24ce-432d-a853-55f1c00dc37a.png)

For my reverse shell of choice, I am using the PentestMonkey PHP reverse shell. Let's try and upload this file!

![upload error](https://github.com/NTHSec/CTF-Writeups/assets/150489159/49da5407-315e-4a62-bf82-65f484a635b0.png)

Hmm, we get an error.

![upload error](https://github.com/NTHSec/CTF-Writeups/assets/150489159/e2b3d9d9-bc2d-4dc7-a88a-3f382c98bfb0.png)

Looks like this box may not be as straightforward as we thought. Maybe there is another path we haven't considered yet?

## Exploitation

After going back to the original page, I remembered the version being 1.4. I looked online for a [CVE](https://github.com/ice-wzl/Fuel-1.4.1-RCE-Updated) and found one!

This PoC is super easy to use, so after following the documentation on the GitHub post, I was able to get my reverse shell in no time!

![reverse shell](https://github.com/NTHSec/CTF-Writeups/assets/150489159/28078cf1-a0aa-404d-9e0a-7f23c6c58524.png)

The first step once you get a shell is to upgrade it. Right now, our shell is unstable, so upgrading and stabilizing it will give us more functionality. We can stabilize our shell by running this command: `python3 -c 'import pty;pty.spawn("/bin/bash")'`

After navigating to the home directory, we find our user flag!

![user flag](https://github.com/NTHSec/CTF-Writeups/assets/150489159/9698c1bf-d8ec-48fb-af46-29e560a1841c.png)

## Privilege Escalation

The first thing I like to do when looking for an avenue of PrivEsc is to try and get linpeas on the system. The most reliable way to do this is to navigate to the /tmp directory on the victim machine and use netcat to transfer the file.

On your victim machine, run `nc -lvnp {PORT_NUMBER} > {FILE_TO_RECEIVE}`:

![linpeas transfer](https://github.com/NTHSec/CTF-Writeups/assets/150489159/5861d6cd-9ecb-4d39-938c-9a746e145b67.png)

On the attacking machine, run `nc {VICTIM_IP} {PORT_NUMBER} < {FILE_TO_TRANSFER}`

![linpeas transfer](https://github.com/NTHSec/CTF-Writeups/assets/150489159/05421bf7-7ba3-4805-997e-66698f96bfa1.png)

Now, since we have Linpeas on the victim machine, let's give it executable permission with `chmod +x` and run it!

![linpeas output](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7e6ec831-f57d-4711-be9b-25b5004fe78e.png)

Here we go!

![linpeas output](https://github.com/NTHSec/CTF-Writeups/assets/150489159/7994745b-6707-40c6-84b6-b6a8a1409e36.png)

After a while, we finally got something from our linpeas output!

![linpeas result](https://github.com/NTHSec/CTF-Writeups/assets/150489159/2f26afef-681f-451c-9fcc-63e7dca93596.png)

Looks like we have login credentials for the SQL database!

I also navigated to that directory and found our username to be root!

![SQL credentials](https://github.com/NTHSec/CTF-Writeups/assets/150489159/0df4620d-a57b-4e17-8318-5971a52b5927.png)

*Note: At this stage of the process, it's important to note that if you encounter a username containing the word 'root,' it's worth checking if it matches any credentials. I made things more complicated by attempting to log in to the SQL database to retrieve hashes. I'll still demonstrate that approach, as it's valuable to explore all possibilities, but keep in mind that you can simply switch to the root user at this point.*

## Logging into the SQL Database

Now that we have the credentials to the Database, let's check it out! We can log in with this command `mysql -u root -p fuel_schema`. Enter our password and we're in!

![SQL login](https://github.com/NTHSec/CTF-Writeups/assets/150489159/5eca555a-8f37-49f8-b3df-03ab14714f27.png)

The first thing I like to do when checking out a database is to look for the users table. This is the table that usually carries sensitive password hashes for users.

First, let's list all the tables using `show tables;`

![show tables](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c56c072c-319e-44ed-a33a-2ed8b7ef7df4.png)

Look at that! At the bottom, we see fuel_users! Looks promising! Let's display all of the data from that table with `select * from fuel_users;`.

![fuel_users data](https://github.com/NTHSec/CTF-Writeups/assets/150489159/82267194-f614-47c7-9327-6b561bb47fa6.png)

Hmm, the only hash we have is the admin hash. We know that the password is admin though, so I think that this is useless information to us.

After cracking the hash with hashcat, we get confirmation that the hash is indeed admin.

![admin hash](https://github.com/NTHSec/CTF-Writeups/assets/150489159/08dbd199-58fc-4ee4-9e56-bfa5294c6b32.png)

What if this was all an extra step? I just realized that I might have overthought all of this. In this database, we are logged in as root. By that logic, what if the password for the database is the same password for the root user?

This is indeed the case! That is quite the oversight by me!

![root password](https://github.com/NTHSec/CTF-Writeups/assets/150489159/c71d067e-b69a-4897-a1ad-4f47adc66b3f.png)

This box has taught me not to overlook the simplest solutions! Sometimes it is easier than you think!
