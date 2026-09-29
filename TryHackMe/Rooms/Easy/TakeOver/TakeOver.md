
### 1. Overview

- **Room and platform:** TryHackMe, Takeover (difficulty: Easy)
- **Objective:** find the hidden hostname and retrieve the flag
- **Target:** `futurevera.thm` (`10.48.160.9`)
- **Tools:** nmap, gobuster, browser
- **Outcome:** Found a hidden subdomain leaked through a TLS certificate's SAN field.

![[Pasted image 20260929171908.png]]

![[Pasted image 20260929171929.png]]

### 2. Enumeration (nmap)

Keep your command and output, then **interpret** it:

- `22 (SSH, OpenSSH 8.2p1 on Ubuntu)`, `80 (Apache, redirects to HTTPS)`, `443 (Apache with a self-signed cert for `futurevera.thm`)`.
- The HTTP-to-HTTPS redirect gave you the domain name, and the certificate expired in 2023.
- Only web is worth attacking, since SSH has no credentials.

```
root@tryhackme:~# nmap -sC -sV -T4 10.48.160.9
```

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-09-29 11:54 UTC
Nmap scan report for ip-10-48-160-9.ap-south-1.compute.internal (10.48.160.9)
Host is up (0.00018s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ad:38:2b:d8:e0:48:8d:94:09:99:3e:30:db:bd:0f:f6 (RSA)
|   256 cd:37:e2:3b:6d:ba:d4:4a:fb:6c:bb:cf:43:c6:04:44 (ECDSA)
|_  256 8e:b1:1f:98:c5:8a:72:49:9e:15:5d:57:bd:00:1b:a2 (ED25519)
80/tcp  open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Did not follow redirect to https://futurevera.thm/
|_http-server-header: Apache/2.4.41 (Ubuntu)
443/tcp open  ssl/http Apache httpd 2.4.41 ((Ubuntu))
|_ssl-date: TLS randomness does not represent time
|_http-title: FutureVera
| ssl-cert: Subject: commonName=futurevera.thm/organizationName=Futurevera/stateOrProvinceName=Oregon/countryName=US
| Not valid before: 2022-03-13T10:05:19
|_Not valid after:  2023-03-13T10:05:19
|_http-server-header: Apache/2.4.41 (Ubuntu)
| tls-alpn: 
|_  http/1.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.68 seconds

```

## 3. Web and Virtual Host Discovery 
### Initial look at the site 
The nmap scan showed that port 80 redirected to `https://futurevera.thm/`, so I added the domain to `/etc/hosts` so my machine could resolve it: 
``` 10.48.160.9 futurevera.thm ``` 

![[Pasted image 20260929172845.png]]

Port 443 served a self-signed certificate, so the browser showed a warning. I accepted it to continue. 

![[Pasted image 20260929173141.png]]

The homepage was a generic company page with no visible links, login forms or useful content.

![[Pasted image 20260929173447.png]]

### Directory enumeration 
Since the homepage gave nothing, I brute-forced directories with gobuster. `-k` skips TLS verification because of the self-signed certificate: 
```bash gobuster dir -u https://futurevera.thm -w /usr/share/wordlists/dirb/common.txt -k https://futurevera.thm ``` This returned nothing useful, so the main site had no interesting hidden paths.
I done the directory enumeration using `gobuster` but didn't find anything. So, I pivot to subdomains enumeration using same tool.


### Virtual host enumeration 
Next I looked for other sites hosted on the same server. One web server can serve many domains and choose the site based on the `Host` header, so a subdomain may exist even if it has no public DNS record. 
I used gobuster in `vhost` mode instead of `dns` mode. The target only resolves through my `/etc/hosts` file, so a DNS brute force would have found nothing. In `vhost` mode, gobuster sends requests to the server's IP with a different `Host` header for each wordlist entry:

```
root@tryhackme:~# gobuster vhost -u https://futurevera.thm -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 20 -k
```

```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             https://futurevera.thm
[+] Method:          GET
[+] Threads:         20
[+] Wordlist:        /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: blog.futurevera.thm Status: 421 [Size: 408]
Found: support.futurevera.thm Status: 421 [Size: 411]
Progress: 4997 / 4998 (99.98%)
===============================================================
Finished
===============================================================
```

### Understanding the 421 status 
Both hits returned HTTP `421 Misdirected Request`. Gobuster connected over TLS using `futurevera.thm` as the server name but sent a different name in the `Host` header, and Apache rejected that mismatch. Every other wordlist entry got a different response. The 421 responses therefore showed that Apache treated these two names differently from the rest, meaning they were configured virtual hosts. I counted them as valid hits and confirmed by visiting them in the browser. 
### Adding the subdomains to /etc/hosts ``` 10.48.160.9 futurevera.thm blog.futurevera.thm support.futurevera.thm ``` 
### Results 
**`blog.futurevera.thm`:** a simple static blog with no forms, logins or interesting links, so it was a dead end. 
 **`support.futurevera.thm`:** a support page with nothing exploitable on the surface. Since it was served over HTTPS, I inspected its TLS certificate, which led to the key finding in the next section.

![[Pasted image 20260929174850.png]]

## 4. Certificate Analysis (Key Finding)
### Why I looked at the certificate
Directory brute-forcing on `futurevera.thm` returned nothing, and vhost enumeration only found `blog` and `support`. The blog was a plain page with nothing useful. TLS certificates often list every hostname they cover, so I inspected the certificate for `support.futurevera.thm` to see whether it exposed any other names. 
### How I inspected it
In the browser, I opened the certificate details (Advanced → View Certificate). 

![[Pasted image 20260929175222.png]]

### What I found The 
**Subject Alternative Name (SAN)** field lists all the hostnames a certificate is valid for. Besides the expected `support.futurevera.thm`, it contained an unlisted hostname: `secrethelpdesk934752.support.futurevera.thm` That name didn't show up in the wordlist-based vhost scan, because it's too random to be in a top-5000 subdomain list. Certificates are the one place where it's disclosed. 
### Exploitation 
I added the new hostname to `/etc/hosts`: ``` 10.48.160.9 futurevera.thm blog.futurevera.thm support.futurevera.thm secrethelpdesk934752.support.futurevera.thm ``` Browsing to `https://secrethelpdesk934752.support.futurevera.thm` returned the flag 

### 5. Result

![[Pasted image 20260929175646.png]]

### 6. Remediation

- Don't publish internal hostnames in public certificates.
- Use a private CA or wildcard cert instead.
- Put authentication or IP restrictions on hidden hosts.
- Monitor Certificate Transparency logs.

### 7. Takeaways

- Check certificates when wordlists fail.
- Obscure names are not security.
- Alternative tools: `ffuf` for vhosts, `crt.sh` for public certificate records.