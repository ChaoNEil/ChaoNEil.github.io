
+++
title = 'SOC Active Directory Cloud Project (Splunk, Shuffle and Slack)'
+++


## Logical Diagram / Setup Structure

![](/images/soc-lab-project/0.png)

```
CompanyX-ADDC01 ip 139.x.x.x   public
CompanyX-ADDC01 ip 10.51.96.4  private

Test instance   ip 139.x.x.x   public
Test instance	ip 10.51.96.3  private
	
CompanyX-Splunk ip 139.x.x.x   public
CompanyX-Splunk ip 10.51.96.5  private

```


Using Vultr cloud resources , I set up 3 different machines:
	
    1 splunk ubuntu machine that will be the splunk server
	1 Active directory machine 
	1 test windows machine

I have set up a VPC network so that all 3 machines could interact nicely. Its basically a virtual isolated network inside the cloud. Due to VPC, i got assigned private IPs for my machines (IPs starting with 10).

The public and private VPC IPs are all mentioned above.

I have also set up firewall rules so that only I (from my local machine) with my public IP address, can access these machines on the cloud. 

![](/images/soc-lab-project/1-edited.png)

At this point in time, i can ping all of my 3 machines:

![](/images/soc-lab-project/2.png)

----

## Configuring Active Directory 

`TLDR: Configured AD server, created new user and joined a machine to newly created domain.`


Now I will configure AD to my `CompanyX-ADDC01` machine and promote it to Domain Controller. 

Root domain name : `CompanyX.local`
Password : `[Redacted]`

Added a user `Jenny Smith`
Password: `[Redacted]`

![](/images/soc-lab-project/3.png)

Now that i have created AD and DC server and a user, i will join the windows test machine to this Domain and authenticate it to the server using `jenny smith` user.

In the process of joining test-machine to the domain server, this error occurred :

![](/images/soc-lab-project/4.png)

Googling this tells me that it usually happens when there is something wrong with DNS resolution. Maybe `COMPANYX` was not able to resolve to a IP address?

![](/images/soc-lab-project/5.png)

The DNS field is empty which is causing the error. Filling the preferred DNS field with the ip address (`10.51.96.4`) of the DC server fixed this issue.

![](/images/soc-lab-project/6.png)

Domain joined !

After restarting test-machine , it will have a option to sign in as Other user. Here, sign in as `JSmith` user. 

![](/images/soc-lab-project/7.png)
`COMPANYX\` is needed because it signifies the domain of that user. 

![](/images/soc-lab-project/8.png)

After entering the creds, i encountered this error. Remote login is disabled.
To enable it, i accessed this test-machine from my vultr dashboard and allowed remote connections 
![](/images/soc-lab-project/9.png)

Added user `JSmith` to remote desktop users.

![](/images/soc-lab-project/10.png)

Now i can RDP into test-machine normally as `JSmith`.

![](/images/soc-lab-project/11.png)


---

## Installing and Configuring SPLUNK
 
*and Sending Telemetry from windows*


Installed splunk free trial:  
```
splunk username : [Redacted]
splunk password : [Redacted]
```

However to get it running on my host laptop, i had to first:

1.  add my public IP address in vultr cloud platform for splunk

![](/images/soc-lab-project/12.png)

2.  and on the `ubuntu-splunk` machine set up a firewall rule to accept incoming traffic to port 8000  which is where splunk lives .

![](/images/soc-lab-project/13.png)

--

I went to splunk indexes to set up my AD index.

`Splunk indexes are a repository of data stored in a structured and searchable format after being processed and transformed into events`
	~Google
	

To receive data from other machines, i set up a port 9997 which is the default.

`Forwarding and receiving >> Reveive data >> set port`

Also to be able to forward data , we must install splunk universal forwarder on target machines which in my case , my target machine is the `test-machine` that has the `JSmith` user.  

I RDPed into test machine and installed the splunk forwarder. Edited the `inputs.conf` file to include security logs from event viewer and Restarted the `splunk services` all as ADMIN.

`inputs.conf file` 
![](/images/soc-lab-project/14.png)

Again i had to `ufw allow 9997` to allow telemetry to flow to our host laptop. 

![](/images/soc-lab-project/15.png)

And after doing all that , i had data finally.


Now i had to do all that again for my Domain controller machine.

End result:

![](/images/soc-lab-project/16.png)

I have both windows endpoints now configured and are sending data.



---

## Creating Splunk Alerts

*Detecting unauthorized successful RDP authentications* 
(Example Scenario)

My public IP starts with `49.*` , hypothetically lets say, if an authentication comes from any other IP other than that , then it will trigger an alert .

![](/images/soc-lab-project/17.png)

This is the query that i created in splunk :
```C
index="companyx-ad" EventCode=4624 (Logon_Type=7 OR Logon_Type=10) Source_Network_Address=* Source_Network_Address!=49.*


Logon Type 7 occurs when a user unlocks their machine
Logon Type 10 indicates a user logging into a computer remotely
```

I will login from a kali machine with VPN to trigger this alert . I have also set my firewall to accept RDP connections from `anywhere`.

Here i got a result back after running above query while earlier it was null.

![](/images/soc-lab-project/18.png)
A successful login from Sweden. 

Cleaning up the query a bit more by using `stats`
```c
index="companyx-ad" EventCode=4624 (Logon_Type=7 OR Logon_Type=10) Source_Network_Address=* Source_Network_Address!=49.* Source_Network_Address!="-"
| stats count by action, user, ComputerName, _time, Source_Network_Address, Logon_Type
```

Here are the results:

![](/images/soc-lab-project/19.png)

I can now save this query as an alert to run it every hour or less depending on the situation with a severity of `Medium`.

![](/images/soc-lab-project/20.png)

That was an example alert. We can add more and save them.

----

## Integrating Slack and Shuffle (SOAR)

*Automating alerts with Shuffle SOAR*

`Shuffle allows users to create workflows to automate various security tasks and integrate with different security tools.` ~ google.

I created an account and linked the webhook URI to splunk's alert edit section.

![](/images/soc-lab-project/21.png)

Started the webhook and it generated alerts based on the alert conditions.

I got an alert from splunk:

![](/images/soc-lab-project/22.png)

The alerting works and now i want it all sent to my slack as a notification so i can get notified to my phone etc, even when i am AFK.

Linked slack and authenticated to receive alerts.

![](/images/soc-lab-project/23.png)


Following up, I clicked the slack block and filled in the values . In the `Find Actions` field , i selected `Chat Postmessage` meaning i want the slack alert bot to send me message . In the `Text` field , i filled up the important parameters that i want to see in an alert like username , time etc.

![](/images/soc-lab-project/24.png)


I was stuck for a while on linking and receiving alerts, but I eventually figured it out. It's now sending me alerts. :

![](/images/soc-lab-project/25.png)

Here the time is in epoch time, we can convert it to human readable too. 
Use this to convert `https://www.epochconverter.com/`

Converted Time :
`GMT: Saturday, July 19, 2025 6:16:19 PM`

__

### Sending alerts through Emails

Now that we sent alerts through slack notifications , its time for email notifications.

We can now for example , include user input. It can work something like `"Do you want to disable <username> user?"`

The alerts can be emailed to respective SOC analyst's shared email so that all the soc analysts on the shift can see it. There are many options in shuffle like `user-input` app . There is even a SMS option. 

Shuffle is very versatile so we can drag and drop many different apps and connect/link the nodes to it and make the workflow to our liking. 
For example , lets go beyond this and ask the analyst if `he/she wants to disable that particular account`. To achieve that , i must connect the `user-action` app to `active directory` app. There is a disable user option already in the AD app, i will select that and fill in the required values. I will also allow port 389 in the firewall for AD services. 



After running the workflow in shuffle, i received this alert email:

![](/images/soc-lab-project/26.png)

This email contains yes and no, clicking on yes disables the user and no does nothing.

![](/images/soc-lab-project/27.png)

I selected Yes here and it was working as expected.
As we can see , the user Jenny has been disabled (indicative by the downward arrow)

![](/images/soc-lab-project/28.png)

As well as a text in slack channel saying that the user has been disabled:

![](/images/soc-lab-project/29.png)


This was the final iteration of the workflow that was used:

![](/images/soc-lab-project/30.png)


##### Quick fixes:
I had to create another account with domain admin privileges in order authenticate to AD in shuffle.
Also many of things that I did here like RDP from `anywhere` , is only for testing purposes only. Never open unnecessary ports to the internet.

----

## Conclusion

This was a great SOC Active Directory lab project as I got to experience many technologies hands on and also got a glimpse of what a real SOC environment may look like. This project helped me understand how real-world SOC environments function by simulating Active Directory attacks and automating alerting using Splunk, Shuffle (SOAR), and Slack. While setting up the workflow in Shuffle, I faced issues with authentication and message formatting but resolving them taught me how important proper configuration and testing are in incident response automation . I also gained practical experience in building detection rules and sending actionable alerts to Slack channels — mimicking how modern security teams stay responsive in real time.

I have never used tools like Shuffle before so using it in this project was very insightful.  

Overall, this was a valuable hands-on exercise in designing and operating a mini cloud-based SOC.