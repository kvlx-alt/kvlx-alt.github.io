# VulnHub | Momentum: 1

VIDEO: [Momentum 1 VulnHub: Penetration Testing on VulnHub (No Commentary) with Lofi Music](https://youtu.be/bw6yuV2m6CE)

Skills:
- Web Enumeration
- Abusing CryptoJS - Decryption Process
- SSH Credentials Guessing
- Abusing Internal Service (Redis) + Information Leakage [Privilege Escalation]

## The Walkthrough/Writeup

Let's continue with `nmap`. We'll use the `-p-` parameter to control the range of ports we want to scan, and since we want to scan all 65535 ports, we'll use a hyphen `-` to indicate that we want to scan the entire range. We want it to report only open ports, so we'll use `--open`. With the `-sS` parameter, we'll launch a SYN scan, which allows us to speed up the scan and is quite stealthy and precise. We'll combine this with `--min-rate 5000` to indicate that we don't want to process packets less than 5000 packets per second. We'll use the `-n` parameter to indicate that we don't want DNS resolution, and finally, we'll use `-Pn` to indicate that we won't perform host detection via ping, and that we'll attempt to scan the ports of all specified hosts regardless of whether they respond to pings or not. We're configuring it this way because we're in a controlled environment and want to quickly exploit it.

```
nmap -p- --open -sS --min-rate 5000 -n -Pn victimipaddress
```

Now we'll launch a much more exhaustive scan. We'll use the `-sC` parameter to indicate that we want to launch a set of basic reconnaissance scripts, and finally, with the `-sV` parameter, we want to show the version and service running on the selected ports.

```
nmap -sCV -22,80 victimipaddress
```

Let's see what we're up against. We have found that port 22 is open and running OpenSSH 7.9, which is superior to version 7.7, so we won't be able to enumerate valid users through the user enumeration vulnerability. The only port left is port 80, so let's launch the `whatweb` tool to identify the technologies being used by the web server.

```
whatweb http://victimipaddress
```

We don't see anything important, so let's use an `nmap` script that acts as a fuzzer. This script has a small dictionary of routes that will allow us to quickly find potential directories.

```
nmap --script http-enum -p80 victimipaddress
```

We see `/css/`, `/img/`, `/js/`, and `/manual/` directories, but there's nothing we can use at the moment, so let's check the website. While exploring the web, we see that it has an XSS vulnerability, but it doesn't seem to be of much use to us.

So let's enumerate a bit more with the help of `gobuster` to perform a more powerful reconnaissance. We'll use `seclists`, which is a Github repository, but can be easily installed with `pacman -S seclists` on Arch Linux.

```
gobuster dir -u http://victimipaddress -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20
```

We still haven't found anything that could be useful, so let's check out `/js/` directory.

 Maybe we'll get lucky.

```
http://ipvictim/js
```

When we open `main.js`, we can see something related to CryptoJS. Let's ask ChatGPT what it's about.

Let's go to the victim's homepage and check our session cookies to see where CryptoJS might be used.

We have a cookie value (`U2FsdGVkX193yTOKOucUbHeDp1Wxd5r7YkoM8daRtj0rjABqGuQ6Mx28N1VbBSZt`), a password (`SecretPassphraseMomentum`), and the encryption method used by CryptoJS, so let's see the `codepen.io` example to do that.

```javascript
// INIT
var encrypted = "U2FsdGVkX193yTOKOucUbHeDp1Wxd5r7YkoM8daRtj0rjABqGuQ6Mx28N1VbBSZt";
var myPassword = "SecretPassphraseMomentum";

// PROCESS
var decrypted = CryptoJS.AES.decrypt(encrypted, myPassword);
document.getElementById("demo1").innerHTML = encrypted;
document.getElementById("demo2").innerHTML = decrypted;
document.getElementById("demo3").innerHTML = decrypted.toString(CryptoJS.enc.Utf8);
```

Now that we have something: String after Decryption: `auxerre-alienum##`, how do we use it? We've already tried brute-forcing to find something and haven't had any success. We still have port 22 with SSH, and we have something that could be a username or password or both, so let's try SSH and use "auxerre" as the username and "auxerre-alienum##" as the password. If that doesn't work, we'll try other combinations.

```
ssh auxerre@ipvictim
```

We've established an SSH connection, and we have the flag, which seems to be the first part of this machine.

Let's go to the root directory with `cd /` and look for SUID privileges to see if we can find any interesting binaries that we can use to elevate our user privileges.

```bash
find -perm -4000 2>/dev/null
```

None of the ones we found are interesting, so let's try recursively searching for capabilities from the root directory using `getcap`.

```bash
getcap -r / 2>/dev/null
```

What we found isn't interesting either, so let's keep looking, this time for active processes on the system.

```bash
ps -faux
```

We see that `redis-server` is running, and this might be useful. 

Let's check if the port that we see in the open process, 6379, is open.

```bash
ss -nltp
```

With this, we can see that the port is open and only visible internally. 

Let's use `hacktricks` to find out how to audit `redis-server`.

We see several options, but we're interested in "Dumping Database," which explains how we can do it. So let's get to work.

We connect to the local machine via port 6379.

```
> nc localhost 6379
> INFO keyspace
> SELECT 0
> KEYS *
```

We have something stored, a key that we're going to try to get with "get". "GET rootpass"

That must be the root password, so let's try logging in as root and see what happens. 

We log in and can now see the flag! We've successfully completed this vulnhub machine!
