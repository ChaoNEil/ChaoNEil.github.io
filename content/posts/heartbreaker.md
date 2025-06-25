+++
title = 'HTB Heartbreaker'
+++

## Sherlocks - Medium

**Sherlock Scenario**

Delicate situation alert! The customer has just been alerted about concerning reports indicating a potential breach of their database, with information allegedly being circulated on the darknet market. As the Incident Responder, it's your responsibility to get to the bottom of it. Your task is to conduct an investigation into an email received by one of their employees, comprehending the implications, and uncovering any possible connections to the data breach. Focus on examining the artifacts provided by the customer to identify significant events that have occurred on the victim's workstation.

**Artifacts**:

We have been given artifacts collected using automated triage tools such as KAPE.

---

Q1
The victim received an email from an unidentified sender. What email address was used for the suspicious email?

In this directory , there is a email for the user ash
`wb-ws-01/C/Users/ash.williams/AppData/Local/Microsoft/Outlook/ash@gmail.com.ost`

This is a Microsoft Outlook OST file , and to load the file , we must setup an outlook account and its a hassle to do all that just for this challenge. Hence to solve this issue i have looked upon the internet and found a tool just for this purpose.

`https://github.com/avranju/pff-tools`

To use :
```shell
pffexport path/to/the/ost/file
```

After exporting , it will create a directory with a `.export` extension. We can browse it normally with our file explorer. 

Under `Inbox` , among all the other emails , `Message00004` seems the most suspicious
```
ashwilliams012100@gmail.com.ost.export/Root - Mailbox/IPM_SUBTREE/Inbox/Message00004
```

There is also an attachment available which is a `.tiff` file, "a Tagged Image File Format".
![](/images/heartbreaker/1.png)
We can also download the "privateMembership.tiff" file which is a `.exe` file.

All this fits the scenario that we have been told.

Coming back to the question, we can find the sender's email in the `OutlookHeaders.txt` file.

`ImSecretlyYours@proton.me` 

---
Q2
It appears there's a link within the email. Can you provide the complete URL where the malicious binary file was hosted?

Copy the URL from the `message.html` file inside the `Message00004` directory.

`hxxp[://]44[.]206[.]187[.]144:9000/Superstar_MemberCard[.]tiff[.]exe`

This is the defanged url.

---
Q3
The threat actor managed to identify the victim's AWS credentials. From which file type did the threat actor extract these credentials?

`.ost` is the file format we are analyzing and extracting information from.

---
Q4
Provide the actual IAM credentials of the victim found within the artifacts.

This can be found in the `[Gmail]` directory under `IPM_SUBTREE/`.

```
[Gmail]/Drafts/Message00001/Message.html
```

![](/images/heartbreaker/4.png)

---
Q5
When (UTC) was the malicious binary activated on the victim's workstation?

For this we will look at the various windows event logs, both default event logs and SYSMON event logs. 
Usually they are stored here `%SYSTEMROOT%\System32\Winevt\Logs%`

We can use a tool from EricZimmerman forensics to make the analysis faster.
```powershell
.\EvtxECmd.exe -d path/to/even/logs --csv /path/to/save
```

This tool will parse all the logs into a format compatible with forensic analysis tools like **Timeline Explorer**, which we will use next.

*Timeline Explorer:
Filter for `.tiff` in `Executable Info` and look for a `process creation`. 

![](/images/heartbreaker/5.png)

Going further right will show us that a process for `SuperStar_MemberCard.tiff.exe` has started :
![](/images/heartbreaker/5a.png)

![](/images/heartbreaker/5b.png)
Above, going further left will give us the time.

`2024-03-13 10:45:02`

---
Q6
Following the download and execution of the binary file, the victim attempted to search for specific keywords on the internet. What were those keywords?


Here lies the firefox profile data ---> `wb-ws-01\C\Users\ash.williams\AppData\Roaming\Mozilla\Firefox\Profiles\hy42b1gc.default-release`

Files under this directory will be in `.sqlite` which can be opened using `DB browser`.

Coming back to the question, the victim searched this after downloading the malicious binary.   
![](/images/heartbreaker/6.png)

We can also take the epoch time and convert it into UTC to further confirm this as the 
answer.

![](/images/heartbreaker/6a.png)

---
Q7
At what time (UTC) did the binary successfully send an identical malicious email from the victim's machine to all the contacts?

For this we will again go back to the `.ost` email file.
This information can be found here ---> 

`ashwilliams012100@gmail.com.ost.export/Root - Mailbox/IPM_SUBTREE/[Gmail]/Sent Mail/Message00005/OutlookHeaders.txt`

![](/images/heartbreaker/7.png)

If we look at the `Recipients.txt` file , we can see that this email has been forwarded to all `ash.williams's` contact.

---
Q8
How many recipients were targeted by the distribution of the said email excluding the victim's email account?

From the `Recipients.txt` file,  we can count the number.

![](/images/heartbreaker/8.png)

---
Q9
Which legitimate program was utilized to obtain details regarding the domain controller?

Go through `TimeLine Explorer` and clear previous filters. Now search for 
"`Superstar_MemberCard.tiff.exe`" , sort the time and look at `"payload data6`

![](/images/heartbreaker/9.png)

`nltest.exe`

---
Q10
Specify the domain (including sub-domain if applicable) that was used to download the tool for exfiltration.

Timeline Explorer--->
We can see `Winscp` which is a file transfer tool, probably used to exfiltrate data.

![](/images/heartbreaker/10.png)

`us[.]softrader[.]com`

---
Q11
The threat actor attempted to conceal the tool to elude suspicion. Can you specify the name of the folder used to store and hide the file transfer program?

From the previous image ;
Answer is `HelpDesk-Tools`

---
Q12
Under which MITRE ATT&CK technique does the action described in question #11 fall?

This attempt to conceal the tool(helpdesk-tools) to elude suspicion is called 
`Masquerading`

---
Q13
Can you determine the minimum number of files that were compressed before they were extracted?

![](/images/heartbreaker/13.png)

The attacker compressed all files above and either compressed it into `WinSCP.zip` or `WB-WS-01.zip` then exfiltrated using `winscp` tool.

26 is the ans.

---


Q14
To exfiltrate data from the victim's workstation, the binary executed a command. Can you provide the complete command used for this action?


With this search :

![](/images/heartbreaker/14.png)

and with the info from the previous question, 
we can see the binary execute a command with the script `maintenanceScript.txt ` after the compression of the data.

The Executable Info column provides the full command used by the malware.
![](/images/heartbreaker/14a.png)

```powershell
"C:\Users\Public\HelpDesk-Tools\WinSCP.com" /script="C:\Users\Public\HelpDesk-Tools\maintenanceScript.txt" 
```

---


