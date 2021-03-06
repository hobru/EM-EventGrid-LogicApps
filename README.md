﻿# Connecting SAP Cloud Platform Enterprise Messaging with Microsoft Azure Event Grid

This project outlines a POC on how you can leverage [SAP Cloud Platform Enterprise Messaging, EM](https://help.sap.com/viewer/bf82e6b26456494cbdd197057c09979f/Cloud/en-US/df532e8735eb4322b00bfc7e42f84e8d.html) to send Events to Microsoft [Azure Event Grid](https://azure.microsoft.com/services/event-grid/) and from there via [Logic Apps](https://azure.microsoft.com/services/logic-apps/) to Outlook. The initial event that is sent to EM will be created via a simple REST call to the EM Queue. 

> Keep in mind this is not (yet) a scenario that can be used in production (e.g. the authentication from EM to Event Grid is done via a SAS Token in a Query parameter which you do not want to do in production!)


The first step is to enable EM on the SAP Cloud Platform. Following the steps to [Create Instance of SAP Cloud Platform Enterprise Messaging Service](https://developers.sap.com/tutorials/cp-enterprisemessaging-instance-create.html) you need to navigate to your Space and from there to Services -> Service Marketplace -> Enterprise Messaging Service. 
![Service Marketplace](Images/MarketplaceEM.jpg)

> If you do not see Enterprise Messaging, you need to configure entitlements on the Subaccount level
![Entitlements](Images/Entitlements.jpg)

Select Instances -> New Instance
![NewInstance](Images/NewInstance.jpg)

In the new *Create Instance* dialog click on Next in the step *Choose Service Plan*. In the next step *Specify Parameters (Optional)* you need to add some parameters. 

>Unlike in the Tutorial mentioned above you need to provide the following information in the **SCP Trial** environment as mentioned [here](https://help.sap.com/viewer/bf82e6b26456494cbdd197057c09979f/Cloud/en-US/6f694dcc608c41d390218848873dc6f3.html)

```json
{
	"emname": "HbrEMInstance",
	"options": {
		"management": true,
		"messagingrest": true
	}
}
```
![SpecifyParameters](Images/SpecifyParameters.jpg)

Now you can just continue to click on Next. In the last step *Confirm* you can enter a name for your new instance, e.g. EMInstanceHbr
![Confirm](Images/Confirm.jpg)

With this the new EM Instance is created 
![Instance](Images/Instance.jpg)
and you can check out the Dashboard by clicking on the Instance and then on Open Dashboard 
![OpenDashboard](Images/OpenDashboard.jpg)


As a next step we need to create a Queue which we will use to publish events to. 

In the Dashboard previously opened select  *Queues* and click on *Create*
![CreateQueue](Images/CreateQueue.jpg)

Now enter a name for the queue which we will publish events to, e.g. emQueue
![CreateNewQueue](Images/CreateNewQueue.jpg)

The queue is now created and in the next steps we can publish a first event to Enterprise Messaging. 
![QueueCreated](Images/QueueCreated.jpg)


---

In order to publish an event we need to make a POST request to our queue. Since we are already running on the SAP Cloud Platform we can use the [SAP Business Application Studio, BAS]() which has a built in http client that allows us to easily posts requests. If you have not done so, please follow the steps outlined in this tutorial to active BAS  [here](https://developers.sap.com/tutorials/appstudio-onboarding.html) and create your own.

Once you are in your Dev Space, click on *Clone from Git* 
![NewFile](Images/NewFile.jpg)

and enter the URL to this GitRepostiory https://github.com/hobru/EM-EventGrid-LogicApps
![CLoneGit](Images/CloneGit.jpg)

Now click on Open Workspace and select the *EM-EventGrid-LogicApps* folder. 
![OpenWorkspace](Images/openWorkspace.jpg)

One of the files of the cloned repo is file called request.http. This file allows you to issue HTTP Calls directly from BAS. Open this file and click on the first *Send Request* above the http://www.microsoft.com URL. On the right hand side you will see the a response from the URL http://microsoft.com, which just proves that the HTTP client from BAS is working.  
```
GET http://microsoft.com
```
![MicrosoftCom](Images/MicrosoftCom.jpg)

As next step we need to authenticate to our Queue which means we need to request a JWT token. In order to do that we need to create a Service key. So back in the SAP Cloud Platform Enterprise Messaging -> Service Instance -> click on Service Keys
![CreateServiceKey](Images/CreateServiceKey.jpg)


Click on Create and enter a name for the new key, e.g. ServiceKeyForRestAccess. Then click on Save
![CreateServiceKey](Images/CreateServiceKey2.jpg)

We need to issue a POST call to the URL mentioned in the Service Key Configure file at the very bottom in the *protocol* -> *httprest* section
* *tokenendpoint*, e.g. https://1c5ba676trial.authentication.us10.hana.ondemand.com/oauth/token 
* and use the authentication username from *clientid*, e.g. sb-clone-xbem-service-broker-e2c2f27ddc514381997e167a191ebaf1-clone!b46544|xbem-service-broker-!b2436 
* and password *clientsecret* e.g. 359f1170-cf25-4f68-b53c-48ca3e2fc290$iQn2WIW91tddV9EJ4kOccTs9ldAUUXz2Nt1hGdheFyw=

![ServiceKeyValues](Images/ServiceKeyValues.jpg)

Just copy and replace your values from the Service Key with the variables in the request.http file. 
* Finally update the *queuname* with your name of the queue, e.g. emQueue. 

Then click on *Send Request*
```
POST {{tokenendpoint}}?grant_type=client_credentials&response_type=token
Authorization: Basic {{clientid}} {{clientsecret}}
```

![OAuthToken](Images/RequestToken.jpg)


As a result you will get a long *access_token* which you can use for the actual message POST. This is automatically done using the login.response.body.access_token parameter. So the only thing left is to publish an event to our queue is to click on *Send Reqeust*
```
POST {{uri}}/messagingrest/v1/queues/{{queuname}}/messages
Authorization: Bearer {{login.response.body.access_token}}
Content-Type: application/cloudevents+json; charset=utf-8
x-qos: 0

{
    "data": {
      "lastname": "Bruchelt",
      "firstname": "Holger"
    },
  "specversion": "1.0",
  "type": "information",
  "source": "/hbr/poc/enterprisemessaging/eventgrid",
  "id": "123"
}
```

![SendMessage](Images/SendMessage.jpg)

The response from the server should be a *204 No Content* which means that the message was received. 

>Note : Azure Event Grid does support several messages formats. One of these formats is the [CloudEvents Schema](https://github.com/cloudevents/spec/blob/v1.0/spec.md). This schema requires certain attributes and headers that need to be set: the required CloudEvents attributes specversion, type, source and id. 
![CloudAttributes](Images/CloudAttributes.jpg)
And also the Content-Header variable application/cloudevents+json
![Content-Type](Images/ContentType.jpg)



---

Now we are ready to switch the sides and head over to Azure. 
> If you do not have an Azure account, you can create a [free Trial](http://azure.com/free). 

Open the [Azure Portal](https://portal.azure.com/) and click on *+ Create a resource*
![CreateResource](Images/CreateResource.jpg)

Search for Event Grid and click on *Event Grid Topic*
![SearchEventGrid](Images/SearchEventGrid.jpg)

Now click on *Create* to create a new Event Grid
![CreateEvntGrid](Images/CreateEvntGrid.jpg)

Create a new Resource Group called *EventGridRG*
![EventGridRG](Images/EventGridRG.jpg)

and enter a Topic name, e.g. EventGridSAPTopic 
![TopicName](Images/EventGridTopic.jpg)

Now make sure to click on *Next: Advanced" to configure the Event Schema. Make sure to select *Cloud Event Schema v1.0* from the Event Schame drop-down. 
![EventSchema](Images/CloudEventSchema.jpg)

Finally click on *Review + Create* and once the validation is successful on Create. After a few minutes the Event Grid should be deployed and by clicking on *Go to resouce* you can open the Event Grid in the Azure Portal.
![Event Grid](Images/EventGrid.jpg)

Under Topic Endpoint you can see the URL that must be called from SCP Enterprise Messaging if a new event arrives, e.g. https://eventgridsaptopic.northeurope-1.eventgrid.azure.net/api/events

Before we configure this on the SAP side, lets configure a Logic App which can pick up the incoming event and processes the result. In our case the incoming data will just be send via Email. Obviously this could be much more. Logic App comes with a huge list of action you can choose from. 

so in the top Search bar look for *Logic App* and select Logic Apps under *Services*
![SearchLogicApps](Images/SearchLogicApp.jpg)

Now click on *+ Add* to create a new Logic App
![CreateLogicApp](Images/AddLogicApps.jpg)

First create a new Resource Group, e.g. LogicAppRG
![LogicAppRG](Images/LogicAppRG.jpg)

Then enter the name of the Logic App, e.g. EventGridLogicApp and click on *Review + create* and after successful Review on *Create* again. 
![EventGridLogicApp](Images/LogicAppName.jpg)

Once the Logic App is created, it will immediately open the *Logic app designer*. Scroll down and select the *+ Blank Logic App*
![BlankLogicApp](Images/BlankLogicApp.jpg)

Now in the Designer search for the *Event Grid* Trigger and select *Azure Event Grid*
![DesignerEventGrid](Images/DesignerEventGrid.jpg)

Select the *When a resource event occurs" trigger
![EventGridEvent](Images/EventGridEvent.jpg)

When you select your Subscription and the Resource Type *Microsoft.EventGrid.Topics* you should see your Event Grid under *Resource Name* that you had previously created. 
![LAFirstStep](Images/LAFirstStep.jpg)

With this you can click on *+ Next step* to select a next action. As mentioned this could be any service running on Azure or also services from 3rd party like SAP. We will send the properties via Email using Office 36z5 Outlook. Search for *Office 365* and select *Office 365 Outlook*  
![Office365](Images/Office365.jpg)

> You can obviously also select any other SMTP server to send email an email (e.g. using Outlook.com or Gmail)

Search for Send and select *Send an Email (V2)* 
![SendEmail](Images/SendEmail.jpg)

Enter the required properties. For the Body field use the *Dynamic Content* on the right side. You might need to click on *See more* next to *When a resource event occurs* to see all available properties from the first step in this Logic App flow. 
![CreateEmailBody](Images/CreateEmailBody.jpg)

Make sure to click on Save!

> There is one manual fix that we need to do since we are using the *Cloud Events* schema and not the *Event Grid* schema. Whereas Event Grid expects and array structure, the Cloud Events schema starts with a single structure. 

Please open the *Code View* and remove the splitOn line.    
![SplitOn](Images/splitOn.jpg)

If you would head back to the Event Grid Topic previously created, you can see that new Subscription pointing to the Logic App is now shown. 
![EventGridSubscription](Images/EventGridSubscription.jpg)

---

Now we are ready to go back to the SAP Cloud Platform Enterprise Messaging Dashboard. 

Click on *Webhook Subscriptions* and click on *Create*
![WebhookSubscription](Images/WebhookSubscription.jpg)

Here comes the POC part. Currently Event Grid does only support authentication via Shared Access Keys, while SAP SCP Enterprise Messaging does support *No Authentication*, *Basic Authentication* and *OAuth2ClientCredentials*. Since headers cannot be modified in Enterprise Messaging, we need to send the SAS Key for the Event Grid as a Query parameter visible in the URL -- which is obviously not good in a production environment. Still for this POC we will take the *Access Key 1* from Azure, 
![AccessKey](Images/AccessKeys.jpg)

Since we send them via an HTML URL we need to URL encode them. This basically means that we change the ending *=* into a *%3D*
![SCPWebhookURL](Images/SCPWebhookURL.jpg)
>Note: If your SAS Token has other special characters you need to URL encode your token first. For this POC you use an online URL encoder like [https://www.urlencoder.org/](https://www.urlencoder.org/)

The status of this new Webhook Subscription might be *Paused*. So click on the *Resume* button to active this service. 
![WebhookStatus](Images/WebhookStatus.jpg)

---

Now all is done. Head back over to the Business Application Studio and trigger another POST call to publish an event to Enterprise Messaging. EM will then use the configured Webhook to send the event to Azure Event Grid. From there Logic App will pick it up and send an email to your account. 
![EmailReceived](Images/EmailReceived.jpg)










