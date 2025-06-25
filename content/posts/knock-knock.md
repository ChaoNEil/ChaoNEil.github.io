+++
title = 'HTB Knock-Knock'
+++

## Sherlocks - Medium

**Sherlock Scenario**

A critical Forela Dev server was targeted by a threat group. The Dev server was accidentally left open to the internet which it was not supposed to be. The senior dev Abdullah told the IT team that the server was fully hardened and it's still difficult to comprehend how the attack took place and how the attacker got access in the first place. Forela recently started its business expansion in Pakistan and Abdullah was the one IN charge of all infrastructure deployment and management. The Security Team need to contain and remediate the threat as soon as possible as any more damage can be devastating for the company, especially at the crucial stage of expanding in other region. Thankfully a packet capture tool was running in the subnet which was set up a few months ago. A packet capture is provided to you around the time of the incident (1-2) days margin because we don't know exactly when the attacker gained access. As our forensics analyst, you have been provided the packet capture to assess how the attacker gained access. Warning : This Sherlock will require an element of OSINT to complete fully.

**Artefacts**:

A wireshark capture file (.pcap) .

---
Q1.
Which ports did the attacker find open during their enumeration phase?

I know for a fact that nmap sends `TCP SYN` packets to target server to scope out possible open ports. 
I went into wireshark's `statistics ---> Endpoints` and sorted by most packets. Found a weird looking (3.109.209.43) recurring IP address sending tons of syn packets to a static destination address (172.31.39.46)
Possible sus ip : 3.109.209.43
![](/static/images/knockknock/1.png)

To further clear out the noise i tweaked the filter a bit:
`ip.addr==3.109.209.43`
This confirmed even further that this IP is the attacker-owned IP. 
The `172.31.39.46` might be the dev server.

Now to know how many open ports the attacker found , we must look for `SYN/ACK` packets . `Syn/ack` packets indicate that the server responded and there is a service on the target server.

This is the final filter I used  :
`ip.addr==3.109.209.43 && tcp.flags.syn == 1 && tcp.flags.ack == 1`
(here 1 indicates a Yes , 0 is null/no).

Apply this filter, the first 5 packets will include the answer ports.

Answer: `21,22,3306,6379,8086`

I advice you all to watch this video,
How NMAP Works and How to Detect Port Scans in Wireshark (A good refresher):
https://www.youtube.com/watch?v=Maec4f2gt2c 

---
  
Q2.
Whats the UTC time when attacker started their attack against the server?

Remove the previous final filter and apply initial filter 
`ip.addr==3.109.209.43`

This will show us the nmap scan.

![](/static/images/knockknock/2.png)

---
Q3.
What's the MITRE Technique ID of the technique attacker used to get initial access?

Using the filter above , scroll down near the end. You will see some TCP packets sent to port 21 and eventually you will see FTP packets . Going down further , you will see user `alonzo.spire` trying to gain access to his account followed by a brute force password spraying attack by the attacker.

![](/static/images/knockknock/3.png)

After port scanning, the attacker found several open ports and has decided to attack FTP port 21 instead. He opted for password spraying (a form of brute force attack) in hopes of getting a hit for the user `alzono.spire` or `tony.shephard` or `lin.bayley`.  

The MITRE technique ID of this attack is `T1110.003`.

---
Q4.
What are valid set of credentials used to get initial foothold?

Keep scrolling down looking through the requests , you will finally see a `230 login successful` ftp packet. Then look for the last username and password that led to this login successful event, you will get your cred.

This is the cred that worked  `tony.shephard:Summer2023!` 

There might be a better and more efficient way of doing this but i did it this way.

---
Q5.
What is the Malicious IP address utilized by the attacker for initial access?

From the investigation that i did up until this point, the ip `3.109.209.43` is definitely malicious.

Answer : `3.109.209.43`

---
Q6.
What is name of the file which contained some config data and credentials?

Using this filter `ip.addr==3.109.209.43 && ftp`, we can see the attacker access `.backup` (a hidden file), `fetch.sh` and enumerating the file server. 

To see what is inside those two files , we can either :
`export objects ---> ftp-data` then save those files to our VM or 
use the `ftp-data` filter and follow tcp stream on those 2 files to read without exporting. 

`ip.addr==3.109.209.43 && ftp || ftp-data`

To answer this question , i found the config data and creds from the `.backup` file .

At first glance , i thought i knew what is going on, but after carefully analyzing this, honestly have no idea. I only recognize the command `iptables` so this might be doing something related to getting access to a particular port ? idk .

Further analysis of this will be done in question 8.
![](/static/images/knockknock/6.png)

The attacker also got creds to a mysql database server from the `.fetch` file, see below ss:
![](/static/images/knockknock/6a.png)

---
Q7.
Which port was the critical service running?

See the port above, `22456`. However that critical service is hidden until we do the pre-requisites. Analysis on the next question.

---
Q8.
Whats the name of technique used to get to that critical service?

I had no clue what this technique was before i googled it and learnt about it, pretty interesting concept too. 
Basically `port knocking` is like a sequence of requests to open a door.

By looking at the .backup file , the sequence of requests someone has to make is to ports is `29999 → 50234 → 45087` in the right order and timing (withing 5 secs). The service then recognizes this as a "knock" from a trusted client. It then runs an iptables command to allow that IP access to port `24456`. 

Pretty cool huh! It was cool but it seems this was very easily bypassed. Now this technique is seen less solely but can be seen along with other layers of security.

Answer : `port knocking`

---
Q9.
Which ports were required to interact with to reach the critical service?

`29999 → 50234 → 45087`

---
Q10.
Whats the UTC time when interaction with previous question ports ended?

The time when the port knocking sequence has ended. I simply added the port numbers as filters.

`ip.addr==3.109.209.43 && tcp.port == 29999 || tcp.port == 50234 || tcp.port == 45087`

Grab the last packet's timestamp from the filter output.

---
Q11.
What are set of valid credentials for the critical service?

We can see the creds in the `.backup` file . See the screenshot from question 6.

---
Q12.
At what UTC Time attacker got access to the critical server?

I used this filter `ip.addr==3.109.209.43 && tcp.port==24456`

I was glancing through each packets looking at the hex dump pane and found out that the packet number `210797` includes the password that i saw in the `.backup` file. 

![](/static/images/knockknock/12.png)

Also after following `TCP stream` on this packet, we can see the complete conversation gaining further insights. The hex dump pane also showed that this is an internal FTP server.
![](/static/images/knockknock/12a.png)


Answer `21/03/2023 11:00:01`

---
Q13.
Whats the AWS AccountID and Password for the developer "Abdullah"?

To get this answer , i had to look at the packets again. Why did wireshark , if this was confirmed ftp packets , not showing me in ftp-packets-format like before ?  That format was easier to read and analyze, instead the ftp data is thrown at me like normal TCP packets. 
I had to consult my old friend Google and it turns out Wireshark is doing this because the port is a non-standard FTP port. Wireshark can only analyze ftp-data nicely when it is coming from a standard ftp port. 

To fix this , i decoded the packets to ftp . Right click on a non-clear ftp packet and `decode as ftp` . 

![](/static/images/knockknock/13.png)

The display is now clear, and the FTP data appears properly formatted, just as it should be.

I `followed TCP stream` and there are some files the attacker downloaded as well as the entire conversation.

![](/static/images/knockknock/13a.png)

Now to view them for myself, I `exported objects` from now reprocessed ftp data.

![](/static/images/knockknock/13b.png)

I found the AWS AccountID and Password for the developer "Abdullah" inside `.archived.sql` mysql dump.

Answer : `391629733297','yiobkod0986Y[adij@IKBDS`

---
Q14.
Whats the deadline for hiring developers for forela?

I found this inside `Tasks_to_get_Done.docx`
Answer: `30/08/2023`

---
Q15.
When did CEO of forela was scheduled to arrive in pakistan?

Open `reminder.txt` 
Answer: `08/03/2023`

---
Q16.
The attacker was able to perform directory traversel and escape the chroot jail.This caused attacker to roam around the filesystem just like a normal user would. Whats the username of an account other than root having /bin/bash set as default shell?

Followed the TCP stream and read the conversation to find the answer. The attacker traversed the directory , accessed the `/etc/passwd` file and did basic host enumeration commands like whoami etc. 

Answer: `cyberjunkie`

---
Q17.
Whats the full path of the file which lead to ssh access of the server by attacker?

The attacker also downloaded a `.reminder` file , I opened that file and saw a note telling me to clean a github repo to avoid leaking sensitive stuff. 
I think this is how the attacker got access to ssh which is evident from the future questions.

The path where the attacker found this .reminder file is 
`/opt/reminders/.reminder`. 

I found this while i was reading through the TCP stream conversation.

Answer: `/opt/reminders/.reminder`. `

---
Q18.
Whats the SSH password which attacker used to access the server and get full access?

Was very lost on this one, wasn't finding anything on wireshark either apart from that one note from `reminder.txt` telling us that there is github repository with sensitive information. I remembered that in the investigation's scenario , it mentioned an OSINT element . So i googled this exact term `github forela` and found a public repo:

![](/static/images/knockknock/18.png)

I explored the `internal-dev.yaml` file then went into `commit history` and found a removed commit. There i found the SSH user `cyberjunkie` and his pass `YHUIhnollouhdnoamjndlyvbl398782bapd`. 

Never commit sensitive information directly to the repository especially if it's public as it can be accessed by anyone and poses a serious security risk.

Answer: `YHUIhnollouhdnoamjndlyvbl398782bapd`

---
Q19.
Whats the full url from where attacker downloaded ransomware?

Using this filter , i was able to get the ssh server's IP (`172.31.39.46`)

`ip.addr==3.109.209.43 && ssh`

![](/static/images/knockknock/19.png)

Since the attacker now have gained access to the ssh server with previously found creds, the download for the ransomware will happen from that ssh server , so i used it in my filters:

`ip.addr==172.31.39.46 && http`

and found a GET request for a `Ransomware2_server.zip` file after scrolling through the packets.
![](/static/images/knockknock/19a.png)

Answer:`http[:]//13.233.179.35/PKCampaign/Targets/Forela/Ransomware2_server.zip`

---
Q20.
Whats the tool/util name and version which attacker used to download ransomware?

Can be found in the packet's HTTP layer information.
![](/static/images/knockknock/20.png)

Answer: `Wget/1.21.2`

Q21.
Whats the ransomware name?

Did this in a VM. 

I exported the ransomware file from wireshark and unzipped it. There i found the source code for the ransomware. Its written in python and the name is `GonnaCry`.
There is also a presentation slide explaining the malware's feature but its in Portuguese and i dont understand Portuguese. 

`GonnaCry`

---

##### Analysis Summary:

An attacker from IP **3.109.209.43** exploited an exposed Dev server (**172.31.39.46**) by scanning for open ports and using leaked FTP credentials to gain initial access. The TA (threat Actor) bypassed further controls using PORT KNOCKING to unlock a hidden service and extracted internal credentials from a config file.

The TA then used an exposed SSH password found in a public GitHub repo related to that Dev server to gain full access as user `cyberjunkie` which the attacker then further used to download a ransomware named GonnaCry via `wget` tool. Also multiple internal files including AWS creds , MySQL database creds and company secrets were compromised.

---

All documenting and solving took me almost 2 days . Worth it though.

---
