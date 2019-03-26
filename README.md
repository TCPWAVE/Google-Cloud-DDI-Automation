# 1. Google Cloud-DDI Automation
The purpose of this content is to give a high-level overview of configuring Google Cloud Ecosystem to automatically register and deregister VM Instance information into TCPWave IPAM and private DNS and to register the information in Google Data store.
# 2. Architecture Overview
![architecture](https://user-images.githubusercontent.com/4006576/54914264-9475c180-4f1a-11e9-8e3b-3e3d62cd9f4c.png)
# 3. Configuration Steps to send updates to TCPWave IPAM
## 3.1 Prerequisites
1.	Project should be attached to the billing account. 
2.	Enable Firebase in the url: https://console.firebase.google.com/  and add the project to it.
3.	Enable Google Cloud Functions.
## 3.2 Create a Topic
Open Topics page from Big Data section in the menu and create a topic with a name.

![topic](https://user-images.githubusercontent.com/4006576/54914576-47deb600-4f1b-11e9-9090-2e54b8e91088.png)
## 3.3 Create a Function
1.	Open Cloud Functions page from Compute section in the menu and create a function.
2.	Enter Name of the function, Select Google Pub/Sub as Trigger, select the Topic that’s created in the above step.
3.	Select ZIP upload option for the Source Code and upload UpdateIPAM.zip.
4.	Note: Please unzip the file, update client.pem, client.crt and client.key with the certificates created for the IPAM, zip the file again.
5.	Select Node.js 8(Beta) as the Runtime.
6.	Select first option in the Stage bucket dropdown.
7.	Give createInstance as the function to execute.
8.	Click on Create.

![function](https://user-images.githubusercontent.com/4006576/54915789-fb48aa00-4f1d-11e9-9516-24e8b0c9d5b1.PNG)
## 3.4 Create Instance using Startup/Shutdown scripts
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
Project should be attached to the billing account. 
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

# 5. Configuration Steps to update QIP
## 5.1 Prerequisites
1.	updateDataStore function must be created and added as a trigger to the Cloud Pub/Sub topic. This topic must be mentioned in the start-up and shutdown scripts while creating instances.
2.	Node js must be installed.
## 5.2 Steps
### 5.2.1 Install google-cloud/datastore packages using below command
        npm install --save @google-cloud/datastore
### 5.2.2 Create Google Cloud Service Account
Follow the steps provided in the below screenshot and download the JSON to the current folder and give this path as keyFileName in the node js function created in the next step.
https://cloud.google.com/docs/authentication/production#obtaining_and_providing_service_account_credentials_manually

![service-account](https://user-images.githubusercontent.com/4006576/54974230-57144100-4fb9-11e9-867c-f703296facc4.png)

### 5.2.3 Create a node.js function
Create a function with below code and give a name to it.
This function gets data from Google Datastore table ‘cloud_automation’ and processes the data and deletes the processed rows from cloud_automation table.
In a row, if the value of operation is add, the function adds objects in QIP. If the operation is delete, it deleted object from QIP.
A CRON job can be created to execute this function at a regular interval of time.
Note: In the function, modify the HTTPS_PROXY, domain, projectId and keyFileName before executing it.

Command to execute the function is below
        
	node test.js(File name)

File content is below
        
	const Datastore = require('@google-cloud/datastore');
        const exec = require('child_process').execSync;
        const moment = require('moment');
        exec('. /opt/qip/etc/shrc');
        exec('export HTTPS_PROXY=https://cspapiproxy-lab.nam.nsroot.net:80');
        exec('touch /tmp/process-add');
        exec('/bin/chmod +x /tmp/process-add');
        exec('touch /tmp/process-delete');
        exec('/bin/chmod +x /tmp/process-delete');
        var domain= 'aws.namdev.nsrootdev.net';
	exec('echo "ObjectAddress,ObjectName,DomainName,ObjectClass,ObjectDescription,Aliases,NameService,DynamicDNSUpdate" > /var/tmp/format');
        const datastore = new Datastore({
          projectId: 'jyothi-218412',
          keyFilename: 'Jyothi-630530137299.json'
        });

        const query = datastore.createQuery('cloud_automation');

        datastore.runQuery(query, function(err, entities) {
          // entities = An array of records.

          // Access the Key object for an entity.
          const firstEntityKey = entities[0][datastore.KEY];
        console.log("Number of rows fetched from Google datastore: "+entities.length);
          for(var i in entities)
          {
                        var e = entities[i];
                        var ip = e.ip;
                        var ts = e.timestamp;
                        var op = e.operation;
                        var n = e.name;
                        var key = e[datastore.KEY]
                        var id = key.id;
                                        var d = new Date(Number(ts));

                if(op == 'add')
				{
					var v =  ip+','+n+','+domain+',server,,,\'A,PTR\',\'A,PTR,CNAME,MX\'';
					var cmd = 'echo \"'+v + '\" > /tmp/process-add';
					exec(cmd);
					console.log('Adding object: '+ip+' '+n+' '+d);
					const addObj = exec('qip-setobject -f /var/tmp/format -d /tmp/process-add');

					addObj.stdout.on('data', function(data){
						console.log('Added object: '+ip+' '+n+' '+d);
						if(data == '0')
						{
							datastore.delete(key, function(err) {
							  if (!err) {
									console.log('Record deleted successfully. Deleted record id: '+key.id);
							  }
							});
						}	
					});

					addObj.stderr.on('data', function(data){
						console.log('Failed to add object: '+ip+' '+n+' '+d+" "+data);
					});
				}
				else if(op == 'delete')
				{
					var v =  'qip-del -t object -a '+ip;
					var cmd = 'echo \"'+v + '\" > /tmp/process-delete';
					exec(cmd);
					console.log('Deleting object: '+ip+' '+n+' '+d);
					
					const deleteObj = exec('/tmp/process-delete');

					deleteObj.stdout.on('data', function(data){
						console.log('Deleted object: '+ip+' '+n+' '+d);
						if(data == '0')
						{
							datastore.delete(key, function(err) {
							  if (!err) {
									console.log('Record deleted successfully. Deleted record id: '+key.id);
							  }
							});
						}	
					});

					deleteObj.stderr.on('data', function(data){
						console.log('Failed to delete object: '+ip+' '+n+' '+d+" "+data);
					});
				}
                
  }

});



