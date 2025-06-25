+++
title = 'HTB Kuber'
+++

## Sherlocks - Easy

**Sherlock Scenario**

As a digital forensics investigator, you received an urgent request from a client managing multiple proxy Kubernetes clusters. The client reports unusual behavior in one of their development environments, where they were testing a proxy via SSH. This environment was exposed to the internet, raising concerns about a potential security breach. You have been provided with a dump of the `kube-system` namespace, as most of the testing activity occurred there. Your task is to thoroughly analyze the data and determine if the system has been compromised.

---
We are given kube-system dump files which are basically kubernetes resource data files.

I am using VS code for this one.

---

Q1.
At which NodePort is the `ssh-deployment` Kubernetes service exposed for external access?

We look into `service.yaml`. This file has the information relating to exposed network applications that is running.
Filter for `NodePort` and we can find the app `ssh-deployment` that has the nodeport of 31337.

![](/images/kuber/1.png)

---

Q2. 
What is the ClusterIP of the kubernetes cluster?

We can see the cluster ip above.

---

Q3.
What is the flag value inside ssh-config configmap?

Filter for `ssh-config` in `configmaps.yaml` file and we can see the flag.

`ConfigMaps.yaml` file are used to store non-sensitive configuration data like environment variables or configuration files.

---

Q4.
What is the value of password (in plaintext) which is found inside ssh-deployment via secret?

For this one, we will head over to the `secrets.yaml` file, obviously!

Scroll down for the password, and decode the base64 string with cyberchef.
![](/images/kuber/4.png)

---

Q5.
What is the name of the malicious pod?

This one was a bit tricky , so I had to filter out by the IP address we found earlier.
Inside the `pods.yaml` file , we will find the name of pod  where we find the IP.

![](/images/kuber/5.png)
If we scroll to the right we can find the ip.

---

Q6.
What is the image attacker is using to create malicious pod?

In the same place as the previous question, we will find the name of the malicious image .
![](/images/kuber/6.png)

---

Q7.
Whats the attacker IP?

Again , same place , we find the attacker IP with a suspicious port number 4444, classic.
![](/images/kuber/7.png)

---

