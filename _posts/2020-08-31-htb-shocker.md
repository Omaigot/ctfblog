---
layout: post
title: HackTheBox - Shocker
date: 2021-08-31 13:32:00 +0100
description: Shocker” is a surprisingly simple Linux box that requires proper enumeration to discover its vulnerability. Further privilege escalation is necessary to achieve root-level access
tags: [HackTheBox]
image: /assets/img/htb/machines/linux/easy/shocker/shocker.png
---

## Foothold

Before we start I always reset the box, it is often that services have crashed or behaves in unintended ways after others have exploited them. And I do not want any spoilers that may have been left by others on the box

### `nmap` scan

First, as usual. i did the initial enumeration of my box using Nmap.

{% highlight bash %}
$  nmap -min-rate 5000 --max-retries 1 -sV -sC -p- -oN Shocker-full-port-scan.txt 10.10.10.56
Nmap scan report for 10.10.10.56
Host is up (0.099s latency).
Not shown: 65532 closed ports
PORT      STATE    SERVICE VERSION
80/tcp    open     http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
2222/tcp  open     ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
53649/tcp filtered unknown
{% endhighlight %}

### Apache/2.4.18 (port 80) 

![dont-bug-me](/assets/img/htb/machines/linux/easy/shocker/dont-bug-me.png)

![dirb](/assets/img/htb/machines/linux/easy/shocker/dirb.png)

Seeing that there is a `/cgi-bin directory`, the webserver is probably vulnerable to Shellstock bash RCE.

> **Shellshock bash remote code execution vulnerability**: affects web servers utilizing **CGI** (**C**ommon **G**ateway **I**nterface) &rarr;  a system for generating dynamic web content. Directories such as `/cgi-sys`, `/cgi-mod`, `/cgi-bin` can be found.

Adding `-X .sh` to `dirb`, we found a [user.sh](http://10.10.10.56/cgi-bin/user.sh) file:

![user-sh](/assets/img/htb/machines/linux/easy/shocker/user-sh.png)

![user-sh-content](/assets/img/htb/machines/linux/easy/shocker/user-sh-content.png)


We can have a shell via `metasploit`:

![shellshock](/assets/img/htb/machines/linux/easy/shocker/shellshock.png)

![meterpreter](/assets/img/htb/machines/linux/easy/shocker/meterpreter.png)

However, since I do this box to get ready for **OSCP**, I want to exploit this vuln manually. Via `curl` or from `burp`, I replace `User-agent`'s content by a **reverse shell** payload:

![burp-1](/assets/img/htb/machines/linux/easy/shocker/burp-1.png)

First we've got the following error: `/bin/bash: bash: Nu such file or directory`.

So I replace `bash` by `/bin/bash` and it worked:

![burp-2](/assets/img/htb/machines/linux/easy/shocker/burp-2.png)

## User (shelly)

Flag:

![user-flag](/assets/img/htb/machines/linux/easy/shocker/user-flag.png)

We can run `perl` with sudo:

![sudo -l](/assets/img/htb/machines/linux/easy/shocker/sudo-l.png)

Let's check [GTFObins](https://gtfobins.github.io/gtfobins/perl/) privesc:

![gtfoperl](/assets/img/htb/machines/linux/easy/shocker/gtfoperl.png)

## Root

{% highlight bash %}
shelly@Shocker:/home/shelly$ sudo /usr/bin/perl -e 'exec "/bin/sh";'
sudo /usr/bin/perl -e 'exec "/bin/sh";'
id
uid=0(root) gid=0(root) groups=0(root)
cd /root
cat root.txt 
9be1ed1fbbe0c3319f9cc05dbcdb7941
{% endhighlight %}

___

## Useful links

- [CVE-2014-6271](https://nvd.nist.gov/vuln/detail/CVE-2014-6271#vulnCurrentDescriptionTitle)
- [Exploit Shellshock on a Web Server Using Metasploit](https://null-byte.wonderhowto.com/how-to/exploit-shellshock-web-server-using-metasploit-0186084/)
- [GTFObins](https://gtfobins.github.io/gtfobins/perl/)
