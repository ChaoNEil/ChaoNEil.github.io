+++
title = 'HTB NeuroSync-D'
+++

## Sherlocks - Easy

**Sherlock Scenario**

NeuroSync™ is a leading suite of products focusing on developing cutting edge medical BCI devices, designed by the Korosaki Coorporaton. Recently, an APT group targeted them and was able to infiltrate their infrastructure and is now moving laterally to compromise more systems. It appears that they have even managed to hijack a large number of online devices by exploiting an N-day vulnerability. Your task is to find out how they were able to compromise the infrastructure and understand how to secure it.



We are given these as _artefacts_:

access.log: Logs incoming HTTP requests to a web server, including details like IP addresses, request paths, timestamps, and response codes. 

bci-device.log: Likely logs related to a Brain-Computer Interface (BCI) device, including device status, errors, or data transfers. 

data-api.log: Likely logs related to interactions with a data API, including request/response details, errors, and performance metrics. 

interface.log: Likely logs interactions with user interfaces, possibly capturing user input, events, or errors within a GUI or web interface. 

redis.log: Logs from a Redis database, capturing system events, commands, errors, and other activities within the Redis instance.

---

Q1. 
What version of Next.js is the application using?

We can find this in the `interface.log`

---

Q2.
What local port is the Next.js-based application running on?

We can find this in the `interface.log`

---

Q3. 
A critical Next.js vulnerability was released in March 2025, and this version appears to be affected. What is the CVE identifier for this vulnerability?

Google is your friend
CVE-2025-29927

---

Q4. 
The attacker tried to enumerate some static files that are typically available in the Next.js framework, most likely to retrieve its version. What is the first file he could get?

Enumerating some static files likely means he accessed the site , so we look at `access.log` to look for web requests and look for the first  `HTTP GET 200`.

---

Q5. 
Then the attacker appears to have found an endpoint that is potentially affected by the previously identified vulnerability. What is that endpoint?

Analyzing the access.log , we can see a whole lot of requests to the `/api/bci/analytics` endpoint and again in the `interface.log` we can see the same endpoint being accessed.

---
Q6. 
How many requests to this endpoint have resulted in an "Unauthorized" response?

In the access.log , we can only see 5 `401` unauthorized response code.

---
Q7. 
When is a successful response received from the vulnerable endpoint, meaning that the middleware has been bypassed?

After 401s  , we can see a 200 response . 2025-04-01 11:38:05

---
Q8. 
Given the previous failed requests, what will most likely be the final value for the vulnerable header used to exploit the vulnerability and bypass the middleware?

So in simple terms , in this exploit , it lets an attacker **bypass authentication** by sending a special header in their request in a Next.js app.

If a Next.js app protects a page using **middleware** (like checking if a user is logged in), an attacker can skip that middleware by adding this HTTP header: 
```header
x-middleware-subrequest: middleware
```
The attacker can increase the header using this pattern until it bypasses the auth check.
```
"x-middleware-subrequest":"middleware"
"x-middleware-subrequest":"middleware:middleware"
"x-middleware-subrequest":"middleware:middleware:middleware"
```
There are 4 failed (401 code) `middleware` strings present in the `interface.log` and after that we can see a 200 response code indicating a success. 
So the repetition of the middleware string should go up-to 5 .

`middleware-subrequest: middleware:middleware:middleware:middleware:middleware`
This is the answer.

---
Q9. 
The attacker chained the vulnerability with an SSRF attack, which allowed them to perform an internal port scan and discover an internal API. On which port is the API accessible?

Look at `data-api.log` , match the timestamp of the attack, and we can see the port number 4000.

---

Q10. 
After the port scan, the attacker starts a brute-force attack to find some vulnerable endpoints in the previously identified API. Which vulnerable endpoint was found?

In the same log file as above , we can see the endpoint `/logs` is vulnerable followed by a `/etc/passwd` command on that endpoint.

---

Q11. 
When the vulnerable endpoint found was used maliciously for the first time?

On 2025-04-01 11:39:01 , the attacker accessed the `/etc/passwd` file from that endpoint for the first time.

---

Q12. 
What is the attack name the endpoint is vulnerable to?

We can see a bypass technique with unusual path traversal to read the `passwd` file.
This is a classic example of `Local file inclusion (LFI)`.

---

Q13. 
What is the name of the file that was targeted the last time the vulnerable endpoint was exploited?

Start from the bottom of data.api.log file , we can see `secret.key` file being accessed.

---

Q14. 
Finally, the attacker uses the sensitive information obtained earlier to create a special command that allows them to perform Redis injection and gain RCE on the system. What is the command string?

This question suggests we look in the `redis.log` file and we sure do see a base64 OS command there.

---

Q15. 
Once decoded, what is the command?

Decode the command on cyberchef.


