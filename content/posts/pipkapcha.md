+++
title = 'HTB Pikaptcha'
+++

## Sherlocks - Easy

**Sherlock Scenario**

Happy Grunwald contacted the sysadmin, Alonzo, because of issues he had downloading the latest version of Microsoft Office. He had received an email saying he needed to update, and clicked the link to do it. He reported that he visited the website and solved a captcha, but no office download page came back. Alonzo, who himself was bombarded with phishing attacks last year and was now aware of attacker tactics, immediately notified the security team to isolate the machine as he suspected an attack. You are provided with network traffic and endpoint artifacts to answer questions about what happened.


---

Q1. 
It is crucial to understand any payloads executed on the system for initial access. Analyzing registry hive for user happy grunwald. What is the full command that was run to download and execute the stager?

Analyzing user "happy grunwald"'s hive with registry explorer; also follow the steps to load the transaction replay logs (.log1 and .log2) to load recent changes.

Go to the "bookmarks" tab to automatically view the important data.

Looking at RunMRU key because It's one of the best spots to find user-typed commands, we can see the full command that was run to download and execute the stager.

RunMRU (run most recently used) stores commands typed by the user into the Start run box (win+r).

---

Q2. 
At what time in UTC did the malicious payload execute?

We can see the time right there.

We now have a timeline to follow.

---

Q3. 
The payload which was executed initially downloaded a PowerShell script and executed it in memory. What is sha256 hash of the script?

To answer this question , we will head back to the pcap file and filter for the ip we found before . We can see the file `office2024install.ps1` , export the file from the pcap and hash it up.

---

Q4. 
To which port did the reverse shell connect?

Analyze the powershell script that we just exported. Spits out base64 encoded text.

```shell
strings office2024install.ps1
<base64string>
```
Decode with cyberchef and remove null bytes to make it readable. 
Here we can see the port of the reverse shell. 

---

Q5. 
For how many seconds was the reverse shell connection established between C2 and the victim's workstation?

We will go back to the pcap file again filter for that ip but now with new found port `6969` .
`ip.addr == 43.205.115.44 && tcp.port == 6969`

Take the first packet's UTC time and last packet's UTC time and put it in a time duration calculator on google, and we will find the answer (in secs).

---

Q6. 
Attacker hosted a malicious Captcha to lure in users. What is the name of the function which contains the malicious payload to be pasted in victim's clipboard?

Use this filter 
`ip.addr == 43.205.115.44 && http`
and click on the last packet and select 'line based text data' under packet data to see the source code and find the malicious function.

We can corelated this run command to the first question we answered, the `RunMRU key`.

The victim pasted this command into run and the `.ps1` file  executed a reverse shell that allowed the threat actor to further enumerate.




