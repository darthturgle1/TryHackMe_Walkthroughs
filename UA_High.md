# TryHackMe Walkthrough: U.A. High School
### By Parker Lovin

## Intro
Welcome to my walkthrough! This room is one of the most enjoyable CTFs I've completed so far. Let's get started!

## Setup
For convenience, I am adding this room to /etc/hosts so I don't have to remember the IP address.

```sudo nano /etc/hosts```

Once Nano is open, go to the end of the file and type the target IP address and the hostname (ua.thm). Your IP address will be different from mine, but it should follow this format:
`10.10.159.59  ua.thm`. Once this has been added, type CTRL X and follow the directions to save the file.

In case anyone needs context for this, /etc/hosts is a file that correlates IP addresses with hostnames. The operation we just performed means we can type "ua.thm" instead of the IP address when running future commands.


## Port scan
In nearly every CTF, I begin by performing a port scan with Nmap. Open your terminal in Kali Linux, then type `sudo nmap -sS ua.thm`
Your output should look like this:
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

## Port scan explained
If you are fairly familiar with CTFs, feel free to skip this section of the writeup. For anyone who is new to hacking, let me explain the output of our scan.

**Port 22 -- SSH:** This is a port used to gain remote access to someone else's machine. SSH tends to be highly secure unless the password is leaked, so we will not attempt to attack it yet.

**Port 80 -- HTTP:** This means a website is running on port 80. In our next step, we will examine this website for vulnerabilities.

## Directory search
I initially wasted a few minutes manually enumerating the web page through Firefox, and I even attempted XSS (if you don't know what that is, don't worry -- you won't need it for this room).
Since this didn't work, I decided to make things easier for myself by letting Gobuster do some work for me:
`gobuster dir -u http://ua.thm -w /usr/share/wordlists/dirb/common.txt`

Results:
```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://ua.thm
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 271]
/.hta                 (Status: 403) [Size: 271]
/.htpasswd            (Status: 403) [Size: 271]
/assets               (Status: 301) [Size: 301] [--> http://ua.thm/assets/]
/index.html           (Status: 200) [Size: 1988]
/server-status        (Status: 403) [Size: 271]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```

The directory **/assets** is the most intriguing part of these results.

## Directory search explained
If you are familiar with Gobuster, feel free to skip this part of the walkthrough. For those who are new to this, here are the concepts:

A website (such as ua.thm) can have many components, and directories help organize these components. Consider a hypothetical website called http://example.com which serves as a shop. It might have a subdirectory called http://example.com/login, and this would serve as a login page. Similarly, http://example.com/search might allow a user to search for specific items.

Gobuster takes a wordlist (in this case, a list of common directory names) and searches a website for them. Here is the breakdown of the command we used:

**gobuster dir**: This tells us that we are using the gobuster command to find directories. Gobuster has other functions which are beyond the scope of this room.

**-u http://ua.thm**: This is the website Gobuster is scanning.

**-w /usr/share/wordlists/dirb/common.txt**: This is a path to a wordlist of common directory names. This is essentially telling Gobuster what directories to look for.


Now to explain the results:
A status code of 400 or greater means that we cannot access the directory. As a result, we only need to look at /index.html and /assets.

Going to Firefox and searching http://ua.thm/index.html gave me the normal web page which seems not to contain anything of value. As a result, we will scan /assets for directories.

## Directory search (/assets)
