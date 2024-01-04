**Welcome to another easy HTB box, *`💲Two Million💲!`* Hope you learn something as I did and enjoy the box!**

# Basic Enumeration
As always, we start with a default *Nmap scan* (and an advanced one) to get the lay of the land.

![default nmap](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/4ae18464-1583-491a-9591-70f4b7cd0402)


Advanced Nmap:

![advanced nmap](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c59d5e4e-463b-4357-8e57-fa4bef2e419b)

We can see that we have an open ssh port, and an open http port, which indicates that there is a website running. The website is http://2million.htb/ (as seen in the advanced nmap scan), so lets add that to our hosts file by running:

*`sudo nano /etc/hosts`*

# Web Enumeration

  ## Web Enumeration With GoBuster
  ![gobuster1](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/2816f0d0-c903-465e-9210-0d8234dfae81)

  We have some interesting stuff, but we're mostly looking at login, register, and invite. Let's check out the webpage as a whole and see if we can come back to any of these URLs.
  
  ===============================
  
  After adding the hostname to our hostfile and navigating to the website, this is what we see:
  
 ![website](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c2cfd575-eae6-4aeb-842c-7c0f2acd090a)

It is a mock website of HTB! How cool! All of the info on this page is awesome, but we are mostly concerned about breaking in, and the best way to do that is to look for a login page. And we have one!

![login](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c6c27d78-99d7-4f90-bbed-f2c304859d2e)


The first thing I noticed was this bottom line of text, which is a common password reset link. At first I thought this could have been a potential path to exploit, but that is not the case, as it leads to nothing. -- Lets continue our journey on this box and start gathering some info!

# Info Gathering

A good start to any login page is to try default credentials

![login2](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/805fc56c-67c8-4b6a-abfc-051d180d360c)

This is our result with ‘admin:admin’  We see that it doesn’t mention anything about our “E-Mail” parameter not being in an Email format (admin@admin.com).

After trying multiple other forms of logging in, I took a little break from the login page and decided to explore the website a little more.

**Under the [ join ] section, we see that we have a redirect link that is different than all the other links on the page.**

![join](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/2a0e65f7-cde0-41c1-a4b1-c69d218cc20e)

*Note: This isn’t a good strategy at finding other directories. But this is what I found before I ran gobuster. It made more sense for the write up to be in good chronological order, so that is why the gobuster sections happens before this.*

When clicking on the join page, we are brought to a page which requires an invite code. So lets try to see where our inputted data is going with *Burp Suite*

![Invite](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c4fbcf7a-93f0-4ab1-a00b-be8412203202)

  ## Burp Suite
  To start analyzing in Burp, we need to first start our proxy and turn intercept on.
  
  ![burp intercept](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/8d238c70-3dd8-4d87-86bb-bd1814e0c4e4)

  After we turn intercept on, we enter random data into the 'invite code' section on the web page and click "Sign Up".   Burp will automatically show us our intercepted traffic. You can also find it under Proxy > HTTP history.

  ![burp response1](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/99da1dd3-993a-44c2-9597-cd46fdecf3c1)

  We know that our code is going to be invalid, which we get confirmation for, but we will use this format later to send other requests to the server. To do this we are going to user the **repeater** function in Burp. You can right click on the response at the top of the screen and click "Send to repeater", or simply press `CTRL+R`

  We can't do much with our Burp request right now, so lets head back to the invite page and see if we can uncover more secrets.

  ## Javascript Deobfuscation 
  
  A great way to find vulnerabilites in any web program is to find the source code. We can find the javascript code repsonsible for invites by going to the DevTools in our browser. In firefox you can simply right click and click "Inspect". Under the debugger tab, we find our code that we can examine.

But wait! The code is obfuscated (basically just jumbled up and super hard to read), meaning we wont be able to understand it! However, the code is obfuscated with the [packed] protocol. Luckily, we can deobfuscate (or unpack) this! A good tool we can use is https://matthewfl.com/unPacker.html  

![js packed](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/9c3d47be-e4a0-4d1c-90c9-28e177b6fced)

Here is our **Original Code**
```javascript
eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
```

Versus our **Deobfuscated Code**
```javascript
function verifyInviteCode(code)
	{
	var formData=
		{
		"code":code
	};
	$.ajax(
		{
		type:"POST",dataType:"json",data:formData,url:'/api/v1/invite/verify',success:function(response)
			{
			console.log(response)
		}
		,error:function(response)
			{
			console.log(response)
		}
	}
	)
}
function makeInviteCode()
	{
	$.ajax(
		{
		type:"POST",dataType:"json",url:'/api/v1/invite/how/to/generate',success:function(response)
			{
			console.log(response)
		}
		,error:function(response)
			{
			console.log(response)
		}
	}
	)
}
```

This function "makeInviteCode" is new to us and gives us good insight on how invite codes are made! Lets break down this code.

- The code defines a function called **`makeInviteCode`**.
- Inside the function, it makes use of jQuery's **`ajax`** function, which is used to send asynchronous HTTP requests to a server.
    - **`type: "POST"`**: Specifies that the request is a POST request, which is commonly used to send data to the server.
    - **`dataType: "json"`**: Specifies that the expected data type of the response from the server is JSON.
    - **`url: '/api/v1/invite/how/to/generate'`**: Specifies the URL to which the POST request will be sent. In this case, it's **`/api/v1/invite/how/to/generate`**.
    - **`success`**: An option that takes a function to be called if the request succeeds. In this case, it logs the response to the console using **`console.log(response)`**.
    - **`error`**: An option that takes a function to be called if the request encounters an error. In this case, it logs the response to the console using **`console.log(response)`**.
- The function is closed with a closing parenthesis and curly brace **`})`**.
- The whole script is closed with an additional closing parenthesis **`}`**.

In summary, the code defines a function that, when called, sends a POST request to the server at the specified URL (**`'/api/v1/invite/how/to/generate'`**). If the request is successful, it logs the server's response to the console. If there's an error, it also logs the response to the console.

Since we want to make an invite code, lets mock this function and send our own POST request to (**`'/api/v1/invite/how/to/generate'`**)! To do this, we can go back to the Burp repeater tab. We send the request and get our response!

![POST burp request](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/f7a5fe8d-e0d1-4d8c-aae6-6194bd4b8286)

We get some encoded ROT13 data and a hint, this is great progress!

### Decoding
Since we know what the message is encoded in, it wont be very hard to decode it. We can decode ROT13 in the terminal like so:

![Data decrpyt 1](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/387b603a-3b42-4be5-91b6-148c9ed70be8)

Or, we can use an online tool, such as [CyberChef](https://gchq.github.io/CyberChef/#recipe=ROT13(true,true,false,13)&input=VmEgYmVxcmUgZ2IgdHJhcmVuZ3IgZ3VyIHZhaXZnciBwYnFyLCB6bnhyIG4gQ0JGRyBlcmRocmZnIGdiIFwvbmN2XC9pMVwvdmFpdmdyXC90cmFyZW5ncg)

![Data decrpyt 2](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/886af09d-73b6-42b4-abed-7ca9b8eb0155)

So, now that we know how to generate an invite code, lets do it!

### Generating an invite code
We can generate a code in two ways: using Burp or using cURL.

Using Burp, its pretty simple. Just replace the path at the top with (**`'/api/v1/invite/generate'`**) and send your request!

![Burp code generation](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/296220cd-d045-44dd-aefe-beb623cb932d)

Using cURL is even easier!

![cURL code generation](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/d75cf1b9-343e-41e2-9ce8-450b3ca97c68)

This code is in base64, so lets decode it and use it!

![code decrypt](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c83eb232-406f-478a-970f-23f2be8ce2c3)

After putting our new code into the invite page, we get redirected to http://2million.htb/register

![register web page](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/a26f69c1-b1e6-44c5-9a39-558fa3f0b77e)

After putting in some arbitrary information and logging in, we get to the home page!

![home page](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/566f8b5b-c61c-42f7-8078-04edeec5d04f)

## Web enumeration (again)
Whenever you are introduced to a new web page, it's good to first explore it, and use tools like gobuster to uncover possible avenues of epxloitation. This is exactly what I did here!

![gobuster2](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/869de13a-f601-4773-8207-c2aa9c190818)

After using GoBuster to enumerate directories on the home page, we see that there are 3 places we can go (they can also all be found on the website itself):

1. access
2. changelog
3. rules

Lets explore each of these and see if we can find anything to exploit.

## Info Gathering (again)

First, lets explore the access page.

![access](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/8eae704b-0f31-4a70-86c2-0877cf9f478c)

If we look in the bottom left, notice that this VPN connection pack links us to an api endpoint at **`'/api/v1/user/vpn/generate'`**. That makes me wonder, what other API endpoints are there?

To find other endpoints, we can go back to burp and use GET requests to find a list of endpoints.

![api endpoints](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/2d260609-1592-43f3-8d24-5dbb6fcf97b7)

We see that there are 3 other endpoints for admin. The bottom **"PUT”** endpoint also leads me to believe that we can update a user account settings, maybe we can update *our* account to become an admin?

Lets navigate to these endpoints and modify them in Burp.


First, the **GET** endpoint at (**`'/api/v1/admin/auth'`**), which will check if the user is admin:
![GET ENDPOINT](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/076c4584-a340-4439-b6b7-5e6d06ebd854)


Next, our **POST** endpoint at (**`'/api/v1/admin/vpn/generate'`**), which will generate a vpn for a specific user:
![POST ENDPOINT](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/e27e80ef-d5dc-4929-9620-80efed00c4f0)


Finally, our **PUT** endpoint at (**`'/api/v1/admin/setting/update'`**), which as mentioned before, updates a user's account settings:
![PUT ENDPOINT](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/7f220242-7a72-46af-9124-657feda8f459)


# Exploitation
After gathering all of that info, the biggest thing we’re looking at is going to the PUT endpoint (**`'/api/v1/admin/settings/update'`**), so we can make our plain old user account an admin. 

Lets start exploring the different parameters we need to use this endpoint in burp. Going back to the screenshot above, we can see that our error message is “invalid content type”. So lets change our content type to what is expected, that being **`application/json`**

![burp content type](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/b8221074-1fa3-4601-925e-31a6186b1221)

After changing our content type, we see that we have a different error message. We are missing the email parameter. So lets add it and try again!

![burp email](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/3cb4f771-fcdd-4312-b071-69447fe5e970)

*Note: Make sure to add the curly braces and the correct quote notation for adding the email and other parameters in burp.*

We’re almost there! the **`is_admin`** parameter looks like a Boolean variable (true/false) so lets update our burp request again and set it to true!

![burp isadmin](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/fd9e57bc-bd36-4be1-a639-7221baae101e)

*Note: Make sure to add parameters in the same curly brace and put a comma after each parameter. Your request format should be the same as the response format parameter-wise.*

We see another error response! We know that in binary 0=false, and 1=true, so let's set is_admin to 1 and try once more!

![burp isadmin2](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/fb5150fb-dc1f-46e3-89d8-d25f14e5ef5f)

*Note: Make sure to take the “” away from the is_admin parameter. We want the request to read as a boolean, not a string.*


Now we have updated our account to become admin. We can check if this worked with the GET admin endpoint (**`/api/v1/admin/auth`**)

![admin auth](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/54711ee8-1e68-41e2-99ff-26349da7df17)

We are admin! Nice! Now lets see if we can generate an admin vpn key with the POST endpoint (**`/api/v1/admin/vpn/generate`**)

![vpngen1](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/91fb02c0-89b6-4e4e-b44b-f5bfeb0a1f66)

Looks like we’re missing the “username”  parameter, which is really strange. This is because when generating any form of info from the OS it isn’t a great idea to take user input. We might be able to manipulate the username input to give us some code execution.

![vpngen2](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/d4aff94a-15c0-450f-a9c9-c0d2f2b6dfce)

Nice! We have our certificates and other information, but what is interesting is further down the response, it takes the “username” parameter into account when generating the response. So we were right! 

Lets try inputting a command into the username field.

![vpn command inject](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c592dcf2-f654-4110-835a-fafaa222c778)

We do in fact have code execution! Lets try to get a reverse shell.

Putting the raw reverse shell into the username parameter wont work, so make sure to encode it first with base64.

![revsh base64](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/94dc6291-4c44-41ef-a8b7-3309def250f0)

Let's set up our reverse listener.

![nc listener](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/6d6c459c-6878-42fc-9061-bb0fdf6a267c)

Now its time to input your encoded reverse shell into the burp request and click send!

![burp rev shell inject](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c5e67795-5520-43fa-b136-74819ecff4a6)

*Note: Make sure to add **`| base64 -d | bash`** at the end of your reverse shell to decode the base64 and to tell the system should execute the output of the previous commands as a Bash script.*

![We're in!](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/813a2013-0076-488a-ab1e-259ff40817da)

We’re in!

*Note: I used the command **`python3 -c ‘import pty;pty.spawn (”/bin/bash”)’`** to make a more interactive and stable shell.*

# Lateral Movement & Privilege Escalation 

## Obtaining the user flag:
A good thing to always to when in a new shell is to do **`ls -a`** to show all files (even hidden ones!)

It reveals a .env file, which we can open to reveal a database login!

*Note: The ".env" file typically contains key-value pairs representing various configuration settings or environment variables used by the application.*
 
![env file](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/0d46ba91-d69f-47ec-af2f-1cf7006542f9)

With our new credentials, lets ssh into the database as admin and get our user flag!

![admin ssh](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/bd7ed356-f409-4199-9dec-9e9e6cd8992a)

After sshing in, we get our user flag, but we are also told that we have mail. Lets check that out and see if we can use that mail to escalate our privileges!

## Privilege Escalation

Let's first check out that mail:

![mail](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/2aa8a9d8-e9ad-4ac6-83f3-cbd06db0a836)

After navigating to /var/mail we see that the admin account does in fact have mail! The email talks about a CVE that the system is exposed to. The CVE is in particular is [CVE-2023-0386](https://github.com/sxlmnwb/CVE-2023-0386)

Let’s break down this CVE:
- **Problem**: In the Linux operating system, there's a mistake in how it handles certain files with special permissions.
- **Where**: Specifically, this issue is in the part of Linux that manages file systems, called OverlayFS.
- **What Happens**: When a regular user copies a certain type of file from one place to another, the system doesn't check properly and allows unauthorized access.
- **Why it's Bad**: This mistake lets a regular user gain more control over the system than they should have.
- **Impact**: It means a local user (someone like us who has a reverse shell) can increase their power on the system without permission, and hence escalate privileges.

So lets get to using this CVE. Weirdly enough, everything I needed for the CVE was already in the admin’s home directory after I checked the mail. I’m not sure if this was intended or not, but I’ll go with it.

![CVE in dir](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/cf99cd13-ddaf-498e-9db5-0379f8daad59)

*Note: If the CVE files did not appear in your home directory as they did in mine, to get the CVE files onto the admin system you can set up a netcat listener to download files from the attacker machine to the victim machine.*


*Victim Machine*
![Note victim machine](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/fcfc9dab-234e-49c7-b55c-224d91317672)

*Attacker Machine*
![Note Attacker machine](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/1a476d9b-070b-40b9-b0ad-7b43eb95f406)

*Press CTRL+C to kill the connection and if we check with ls, we have our test file!*
![Note check](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/9ae39f47-efd1-4c8e-bca0-e5772722b051)


Anyways, back to hacking! The documentation for this CVE isn’t that complicated, it simply says to run the commands in a certain order.

![CVE github](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/3beee8d1-459d-4766-87d4-20b17845ff48)

So let's get to it!

*Note: We need two terminals for this CVE. To split your terminal horizontally, you can press CTRL+SHIFT+D in Kali. Keep in mind you will have to ssh in again.*

In the first terminal, I ran **`fuse`**, **`/ovlcap/lower2`**, and **`gc`**.

![CVE first](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/a0f6ed62-8653-4f85-ac0d-64b47f072e08)


And in the second terminal, I ran **`exp`** and voila! We have root! We can now navigate to the root directory and obtain our root flag!

![CVE second](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/22d2fcd3-acb0-4c4e-85e1-ce2ce83d203e)




Hope you enjoyed the box as much as I did! I'm also trying a new formatting style to emphasize certain commands file paths, or generally important information to improve readability and comprehension. The use of *Notes* both helped me while documenting, and hopefully will help you while reading! 

Additionally, you may notice that I didn't fully explore the other two URLs GoBuster picked up while enumerating the home page. Since this is a CTF and I found the exploitable path, I didn't have the need to document the other two pathways (since they lead to nothing). Keep in mind that this is *bad Penetration Testing methodology*. Keep in mind if you're doing an actual Pentest, you should document and explore every pathway you find.

Anyways, Happy New Year and Happy Hacking!
