# 1. Google Cloud-DDI Automation
The purpose of this content is to give a high-level overview of configuring Google Cloud Ecosystem to automatically register and deregister VM Instance information into TCPWave IPAM and private DNS and to register the information in Google Data store.
# 2. Architecture Overview
![architecture](https://user-images.githubusercontent.com/4006576/54914264-9475c180-4f1a-11e9-8e3b-3e3d62cd9f4c.png)
# 3. Configuration Steps to send updates to TCPWave IPAM
## 3.1 Prerequisites
1.	Project should be attached to billing account. 
2.	Enable Firebase in the url: https://console.firebase.google.com/  and add the project to it.
3.	Enable Google Cloud Functions.
## 3.2 Create a Topic
Open Topics page from Big Data section in the menu and create a topic with a name.
![topic](https://user-images.githubusercontent.com/4006576/54914576-47deb600-4f1b-11e9-9090-2e54b8e91088.png)
## 3.2 Create a Function
1.	Open Cloud Functions page from Compute section in the menu and create a function.
2.	Enter Name of the function, Select Google Pub/Sub as Trigger, select the Topic that’s created in the above step.
3.	Select ZIP upload option for the Source Code and upload UpdateIPAM.zip.
4.	Note: Please unzip the file, update client.pem, client.crt and client.key with the certificates created for the IPAM, zip the file again.
5.	Select Node.js 8(Beta) as the Runtime.
6.	Select first option in the Stage bucket dropdown.
7.	Give createInstance as the function to execute.
8.	Click on Create.

