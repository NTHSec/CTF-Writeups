**Welcome to another easy HTB box, *💲Two Million💲!* Hope you learn something as I did and enjoy the box!**

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
  
  After adding the hostname to our hostfile and navigating to the website, this is what we see.
  
 ![website](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/c2cfd575-eae6-4aeb-842c-7c0f2acd090a)

It is a mock website of HTB! How cool! All of the info on this page is awesome, but we’re mostly concerned about breaking in, and the best way to do that is to look for a login page. And we have one!

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

  After we turn intercept on, we enter random data into the 'invite code' section on the web page and click "Sign Up".   Burp will automatically show us our intercepted traffic. You can also find it under the Proxy > HTTP history.

  ![burp response1](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/99da1dd3-993a-44c2-9597-cd46fdecf3c1)

  We know that our code is going to be invalid, which we get confirmation for, but we will use this format later to send other requests to the server. To do this we are going to user the **repeater** function in Burp. You can right click on the response at the top of the screen and click "Sent to repeater", or simply press `CTRL+R`

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

We get some encrypted ROT13 data and a hint, this is great progress!

### Decryption
Since we know what the message is encrypted in, it wont be very hard to decrypt it. We can decrypt ROT13 in the terminal like so:

![Data decrpyt 1](https://github.com/NoahHeroldt/CTF-Writeups/assets/150489159/387b603a-3b42-4be5-91b6-148c9ed70be8)

Or, we can use an online tool, such as [CyberChef](https://gchq.github.io/CyberChef/#recipe=ROT13(true,true,false,13)&input=VmEgYmVxcmUgZ2IgdHJhcmVuZ3IgZ3VyIHZhaXZnciBwYnFyLCB6bnhyIG4gQ0JGRyBlcmRocmZnIGdiIFwvbmN2XC9pMVwvdmFpdmdyXC90cmFyZW5ncg)


# Exploitation


# Privilege Escelation 

