# Azure SIEM

In this lab, we are going to create a VM in Azure and configure the firewall to allow all connections from the Internet.
We will log failed login attempts and pin them on a map.


# Create VM
We create a Windows 10 VM in a Resource Group.

![](https://i.imgur.com/wbV0CzZ.png)

Use the default "Size" settings:

![](https://i.imgur.com/ogceNzM.png)



Leave the Disk settings as default and continue to the Networking Tab:

![](https://i.imgur.com/oswMCMG.png)

## Create a new network security group
Delete the default rule and add the below one.
Here we allow connections to all ports of our VM. Acknowledge this settings and click "Create".

![|400](https://i.imgur.com/CyMtHIr.png)




# Create Log Analytics workspace
Here we will ingest the logs of the VM, and we will create our custom log, which also contains geographic information.

![](https://i.imgur.com/koUQ85a.png)

Click on "Review + Create" and "Create".

# Enable Azure Defender
Go to "Security Center" and select your resource group:

![|500](https://i.imgur.com/HY3fjRg.png)

Enable Azure Defender:

![|800](https://i.imgur.com/RbhzO27.png)


Save it and proceed to "Data collection". Select "All Events" and save it.

![|800](https://i.imgur.com/7DtF7Zu.png)


# Connect Log Analytics workspace to VM
Go to "Log Analytics workspaces" and select your VM:

![](https://i.imgur.com/VEMofvW.png)

Click on connect:

![|400](https://i.imgur.com/V5bu0aq.png)

# Create Sentinel SIEM
Go to "Microsoft Sentinel", select your workspace and click "add".

![|400](https://i.imgur.com/ugQMLNC.png)
# Login to VM
Proceed to your VM and copy the Public IP address:

![|800](https://i.imgur.com/uj6tXzz.png)

Login from your host machine using RDP.

![](https://i.imgur.com/2E7LlFm.png)


Login with your credentials:

![](https://i.imgur.com/hl2eBL5.png)


Try to log in from your host to the VM one more time, but now with no valid credentials, so we can see the invalid login in the "Event Viewer".
Open "Event Viewer" on the VM and open the Security logs.

![](https://i.imgur.com/ANySsbv.png)


Look at the properties of the login failure with EventID 4625.

![|400](https://i.imgur.com/jfopVpZ.png?1)

We will grab the IP address using PowerShell and determine with the "ipgeolocation.io" API from where Users are trying to login. 

Now we try to ping the VM from our host machine:
`ping <vm ip-adress>`

We get no reply, because the VM firewall is rejecting ICMP requests.

![](https://i.imgur.com/IlcmOrg.png)

We have to turn off the firewall to let users worldwide detect the VM faster.


## Disable Firewall on VM
Type `wf.msc` in the search bar of the VM and click on the "Windows Defender Firewall Properties":

![](https://i.imgur.com/zcCTKua.png)

Turn the Firewall off at the Domain, Private and Public profile:

![|400](https://i.imgur.com/rKh0iaX.png)


Now our ping to the VM is being accepted:

![|500](https://i.imgur.com/4WnyQiO.png)

## Apply API Powershell script 
[Download](https://github.com/joshmadakor1/Sentinel-Lab/blob/main/Custom_Security_Log_Exporter.ps1) this Powershell script from Github.
It retrieves the failed Logins(EventID 4625) from people and grabs the IP address, which is then passed to the [API](https://ipgeolocation.io/). The API responds with geographical data of the IP address, and the script creates a new log with the geodata in it.
The script has some sample data, with which we will train Log Analytics.

All you have to do in the code, is to replace the API key.

To get the API key, you have to sign up at [ipgeolocation](https://ipgeolocation.io/).

![](https://i.imgur.com/vq3rfkG.png)


After changing the API key run the script. 

![|800](https://i.imgur.com/EVrYmI0.png)

At the bottom, you will have the failed Logins popping up in real time.


# Create a custom log in Log analytics

In the "Log Analytics workspace", select "Custom Logs":

![](https://i.imgur.com/4xA5Me3.png)

Add a new one, and then we have to provide a sample log file. It will be used to train Log Analytics.
Our sample log file is located at the VM, so copy the log file's contents and save it as `samples.log` to your host machine.
Click next, then you have to provide the path to your log file:

![](https://i.imgur.com/s6yFly8.png)
Click next, give the log file a name and create it.
Now go the logs, here you can query for: `SecurityEvent | where EventID == 4625 `

![](https://i.imgur.com/qa0hAXB.png)

The failed Logins will show up like in the Event Viewer. 

You can save the query:
![](https://i.imgur.com/oNHCWrt.png)


Then choose your log in the query.

![](https://i.imgur.com/vcXgLDS.png)

![](https://i.imgur.com/53R8aT2.png)

## Extract fields
The next step is to extract the necessary fields from the raw data. We want to have a separate field for latitude and longitude. 

Right-click on one of the logs and select "Extract fields":

![](https://i.imgur.com/LN2yxEC.png)

Giving the sample log, we have to select the value we want to extract, give the field a name, and choose the "Numeric" field type:

![](https://i.imgur.com/cn9s13o.png)

Based on this selection, we will be given the extracting results for our provided sample data.

![](https://i.imgur.com/Ra0FF5P.png)


If the results are correct, then save the extraction.
Repeat the same process with the longitude. As we can see below, the search results don't align with our requested value, so we have to train the algorithm and select them manually for the wrong log samples:

![](https://i.imgur.com/OaaH9MQ.png)

![](https://i.imgur.com/EqZKHzv.png)

When finished, save the extraction.
Extract also the other fields from the raw logs:
- destination host
- username
- source host
- state
- country
- label (will be used to plot on the map)
- timestamp (use "Date/Time as field type")

Under "Custom Logs" and "Custom Fields", you can see all fields which should be extracted.

![](https://i.imgur.com/lOY7GEP.png)

To test this, try to login with invalid credentials, to see if the fields get extracted.

![|800](https://i.imgur.com/zeunZ84.png?1)



# Set up Sentinel Dashboard
Go to "Sentinel" and "Overview":

![](https://i.imgur.com/XriL5L1.png)


On the following day, we already got over 800 failed RDP logins!
Go to "Workbooks" add a new one, and remove the default widgets.
Then add a query:

![](https://i.imgur.com/bETIDeg.png)


Select the fields you want and click "Apply".

![|400](https://i.imgur.com/hL2UVB7.png)


Now you can see where the failed logins come from on the map.


![](https://i.imgur.com/WzmtXqd.png)


# Create Alert

Go to "Log Analytics Workspace", select "Alerts" and add a new one.

![](https://i.imgur.com/urAyvxi.png)

Run our query and click on "Continue Editing Alert":

![](https://i.imgur.com/fy5ehmf.png)


Select "Custom log search" and specify the log query.

![](https://i.imgur.com/pQA47PR.png)


![](https://i.imgur.com/jKwzxq2.png)

In the "Actions" tab, we define when the alarm should be raised. In our case, the alarm gets triggered if there are more than 15 failed logins. 

![](https://i.imgur.com/ksKswOX.png)

Give the alert a name:

![](https://i.imgur.com/FFEq6m0.png)


As we can see below, our alert is triggered every 5 minutes, as this is the frequency of evaluation I set for this rule. So more than 15 failed RDP logins happen every 5 minutes. 

![|800](https://i.imgur.com/QAaQogl.png)


## Create Action Group

Let us create an action based on this alerts. Navigate to "Action Groups":

![](https://i.imgur.com/FxssBbP.png)


Proceed to "Notifications":

![](https://i.imgur.com/ggmipgY.png)
Put an email address for the notifications:

![|500](https://i.imgur.com/zwPfpwn.png)

## Create Alert processing rule

The last thing we have to do is create an "Alert processing rule":

![|500](https://i.imgur.com/vZb2MxE.png)

![](https://i.imgur.com/I4fWueR.png)

Select "Apply action group":

![|500](https://i.imgur.com/fU999Jz.png)


Choose the Action Group we created earlier:

![|500](https://i.imgur.com/uh4Of7p.png)

Give it a name:

![](https://i.imgur.com/kC9aPeZ.png)

Now you will get an email when the alert is triggered that looks like this:


![](https://i.imgur.com/uyu7SKe.png)


