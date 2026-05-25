---
title: Deploying T-POT on Azure
description: A walkthrough of deploying and securing T‑POT on Azure to capture real attack telemetry.
date: 2026-02-12
categories: [Labs & Projects, Azure]
tags: [easy, azure, honeypot, t-pot, threat-intelligence]
image: https://github.com/telekom-security/tpotce/raw/master/doc/tpotsocial.png
---

I’ve always been curious about how much malicious traffic is actually out there, so I decided to deploy a T‑Pot honeypot on Azure and see for myself. It turned out to be a simple and fun project, and in this post I’ll walk through everything I did — from creating the VM to checking the dashboards and analyzing the attacks. If you want to try it too, this guide will take you through it step by step.

> Remember that honeypots are made to intentionally attract malicious traffic, and that the created VM should never be connected to production.
{: .prompt-info }

### **Preparation**
You can do this lab completely for free with a new **Azure Free Account**, while you will need to provide a credit card for verification, Azure provides $200 of credit for the first 30 days, allowing you to complete this lab at no cost.

### **Deploying the Virtual Machine**
First we need to create the Virtual Machine that will host the **honeypot**, go to the **Virtual machines** panel and select **Create** > **Virtual machine**.
Now let's configure the virtual machine. Under the basics tab :

- **Subscription**: You have to select your subscription, if you have created a new account you probably only have one.

- **Resource group**: You will need to create a resource group that will contain all the azure components we create like the public IP or the VM. You can call the resource group `rg-honeypot`.

- **Virtual machine name**: This is VM name, you can call it however you want, like `vm-tpot` for example.

- **Region**: This is the region where the VM will be physically deployed, it has no effect on the results, you can choose the closest to you.

- **Availability options**: This are some options for physical redundancy over different regions, I recommend selecting `No infrastructure redundancy required` because this is project is not a server in production that needs high availability.

- **Security type**: Different security measures, no need as this is not a production server, select `Standard`

- **Image**: This is the OS for the VM, you can see what OS are [supported by tpot](https://github.com/telekom-security/tpotce?tab=readme-ov-file#choose-your-distro), I will use `Ubuntu Server 24.04 LTS - x64 Gen2`

- **Size**: You would need at least 16GB ram to fulfill the [system requirements](https://github.com/telekom-security/tpotce?tab=readme-ov-file#system-requirements), I have choosen `Standard_D4s_v3 - 4 vcpus, 16 GiB memory ($156.22)` 

![Azure VM Basics Configuration](assets/img/posts/az-honeypot/azure-basics1.png)

- **Administrator account**: Now you have to configure your administrator account, either you can choose to use SSH Public key or using password, I would use Password and create an username and password. Remember to allow port 22 for now, its the default option.

> User might lose their original SSH connection on Port 22 and will need to use Port 64295 for future admin access.
{: .prompt-info }

![Azure VM Basics Configuration](assets/img/posts/az-honeypot/azure-basics2.png)

- **Disks tab**: On the next tab, `disks` you would have to select a disk of at least 128GB.

![Azure VM Basics Configuration](assets/img/posts/az-honeypot/azure-disks1.png)

- **Networking tab**: Now on the networking tab check the `Delete public IP and NIC when VM is deleted` so when we are done with the project, everything gets erased correctly.

![Azure VM Basics Configuration](assets/img/posts/az-honeypot/azure-network1.png)

- **Management tab**:Leave the default configuration in the Management tab.

![Azure VM Basics Configuration](assets/img/posts/az-honeypot/azure-management1.png)

Now just click on `Review + create` tab and create the vm. After creating the VM we just need to open the [required ports](https://github.com/telekom-security/tpotce?tab=readme-ov-file#required-ports) of the honeypot services. To make that go to the **Network settings** of the VM resource and click on crete an Inbound rule with the [required ports](https://github.com/telekom-security/tpotce?tab=readme-ov-file#required-ports), We are going to be running the Standard / Hive option of T-Pot so some ports that appear on the list are not necessary.

This is Port list I allowed, it contains all except the LLM ones, check what ports you should allow in your case.
19,21,22,23,25,42,53,69,80,102,110,123,135,143,161,389,443,445,502,623,631,993,995,1025,1080,1433,1521,1723,1883,1900,2404,2575,3000,3306,3389,44818,47808,5000,50100,5060,5432,5555,5900,6379,64294,64295,64297,64305,6667,8080,8081,8090,8443,9100,9200,10001,11211,11434,25565

> Enabling all ports its a major security risk, it should never be done, It generates unnecessary risk and noise, you should only allow the strict and necessary ports. Also you should make the effort of only allowing access trough you IP to the managing ports, so do your diligence and investigate about it.
{: .prompt-danger }

![Azure VM Basics Configuration](assets/img/posts/az-honeypot/azure-ports-rule.png)

### **Installing T-Pot**

First we need to connect to the machine through ssh
```shell
ssh honeypot-admin@20.107.169.58
```

Now we need to install T-Pot, run the following command and follow the installer
```shell
env bash -c "$(curl -sL https://github.com/telekom-security/tpotce/raw/master/install.sh)"
```
When prompted to begin the installation, enter y. For the Install Type, select h for a Hive installation. Finally, set your web user credentials when prompted. After installation is completed you can reboot with `sudo reboot now`

### **Accessing to the T-Pot web Panel**

To access the dashboard go to your browser and search for `https://<Your-VM-IP>:64297` then login with the webuser credentials created during installation.

![T-Pot Panel](assets/img/posts/az-honeypot/honeypot-panel.png)

- **Attack map**: Real-time visualization of incoming attacks.
- **Cyberchef**: A "Cyber tool" for analyzing and decoding data.
- **Kibana**: The heart of the stack—used for visualizing and analyzing logs (my personal favorite).
- **SpiderFoot**: An OSINT automation tool.

### **Results (After 5 Hours)**
Even in a short window the amount of traffic captured is pretty big. Below are some snapshots of the Kibana dashboards.

![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana1.png)
_Total attacks and attacks of each honeypot_

![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana2.png)
_Statistics about the attacks_

![T-Pot Kibana Panele](assets/img/posts/az-honeypot/kibana3.png)
_Statistics about the attacks_

![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana4.png)
_Credentials used by the attackers_

![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana5.png)
_IPs, ASN of the attackers, CVEs used and Signature Alerts_

### **Results (After 4 Days)**
Final results after 4 Days, the average cost of hosting this honeypot on Azure was around 5€/day.
![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana7-1.png)
_Total attacks and attacks of each honeypot_

![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana7-2.png)
_Statistics about the attacks_

![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana7-3.png)
_Statistics about the attacks_

![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana7-4.png)
_Credentials used by the attackers_

![T-Pot Kibana Panel](assets/img/posts/az-honeypot/kibana7-5.png)
_IPs, ASN of the attackers, CVEs used and Signature Alerts_

### **Cleaning the project**
To don't generate unnecessary costs and to still be able to do more projects on Azure, we can delete the resource group and end the honeypot. First we need to go to the resource panel and select the resource group of the honeypot, then we can finally delete it.

![Azure Resource Group Management](assets/img/posts/az-honeypot/azure-delete1.png)

![Azure Resource Group Management](assets/img/posts/az-honeypot/azure-delete2.png)

### **Conclusion**
Is crazy that even that this was a simple project do develop and deploy we learned to deploy a VM on Azure, how easy is to deploy T-Pot and configure it, how many attacks are made daily around the globe what type of attacks. This by no means an exhaustive tutorial and I can not highly recommend enough to not only blindly follow the tutorial and to investigate more about all options and tools T-Pot offers.