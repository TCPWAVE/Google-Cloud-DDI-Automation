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

![function](https://user-images.githubusercontent.com/4006576/54915789-fb48aa00-4f1d-11e9-9516-24e8b0c9d5b1.PNG)
## 3.3 Create Instance using Startup/Shutdown scripts
1.  Go to Compute Section of the menu and open VM Instances page from Compute Engine.
2.  Click on Create Instance.
3.  After entering the Machine type, region etc, select Set Access for each API as option for Access Scopes and select Cloud DataStore and Cloud Pub/Sub and enable them.

![instance](https://user-images.githubusercontent.com/4006576/54916880-8460e080-4f20-11e9-8c4c-69475a230efb.png)

4.  Expand Management, security, disks, networking, sole tenancy section and enter the below script in the Startup script text box in Automation section. Give the name of the topic that is created in the gcloud publish command.

        #! /bin/bash

        orgName='Internal'
        domainName='tcpwave.com'
        ttl='1200'
        action='add'

        instanceid=$(curl http://metadata/computeMetadata/v1/instance/id -H "Metadata-Flavor: Google")
        hname=$(curl http://metadata/computeMetadata/v1/instance/name -H "Metadata-Flavor: Google")
        iip=$(curl http://metadata/computeMetadata/v1/instance/network-interfaces/0/ip -H "Metadata-Flavor: Google")
        projId=$(curl "http://metadata.google.internal/computeMetadata/v1/project/project-id" -H "Metadata-Flavor: Google")

        gcloud pubsub topics publish DDIAutomation  --attribute                   
        id=$instanceid,name=$hname,ip=$iip,orgName=$orgName,domainName=$domainName,ttl=$ttl,action=$action,projectId=$projId

5.  In the Metadata section, add key value pair, shutdown-script as the key and below script as the value. Give the name of the topic that is created in the gcloud publish command.

        #! /bin/bash

        orgName='Internal'
        domainName='tcpwave.com'
        ttl='1200'
        action=’delete’

        instanceid=$(curl http://metadata/computeMetadata/v1/instance/id -H "Metadata-Flavor: Google")
        hname=$(curl http://metadata/computeMetadata/v1/instance/name -H "Metadata-Flavor: Google")
        iip=$(curl http://metadata/computeMetadata/v1/instance/network-interfaces/0/ip -H "Metadata-Flavor: Google")
        projId=$(curl "http://metadata.google.internal/computeMetadata/v1/project/project-id" -H "Metadata-Flavor: Google")

        gcloud pubsub topics publish DDIAutomamtion  --attribute  
        id=$instanceid,name=$hname,ip=$iip,orgName=$orgName,domainName=$domainName,ttl=$ttl,action=$action,projectId=$projId
        
Create instance with the above settings.
With the above successful settings, an object in IPAM will be created when instance is started and object will be deleted when the instance is stopped.
# 4. Configuration Steps to send updates to Google Datastore
## 4.1 Prerequisites
Project should be attached to billing account. 
Enable Google Cloud Functions.
## 4.2 Create Entity
1.	Open Entities page from Datastore menu of Storage section from main menu.
2.	Click on Create Entity. 
3.	Enter cloud_automation in the Kind input box.
4.	Select Numeric ID as the Key identifier.
5.	Click on Create.

![entity](https://user-images.githubusercontent.com/4006576/54973953-2e3f7c00-4fb8-11e9-9e24-bce622dacbdd.png)

Repeat the steps that are present in the section: Configuration Steps to send updates to TCPWave IPAM with UpdateDatastore.zip as the function to be uploaded. Function to execute is updateDataStore.
As the zip file doesn’t contain certificate files, there is no need to update them.
After the completion of above steps, if the instance is started/stopped, the row with instance details will be added in the cloud_automation table.

![entity1](https://user-images.githubusercontent.com/4006576/54973984-54fdb280-4fb8-11e9-9381-3ab3a48f3c12.png)




