---
layout: post
category: azure
title: The Azure Functions Files Episode 1 - The Case of the Consumption Plan File shares 
---

For the last few months I've been building a large enterprise API composed of many (30+) Azure Functions v2 microservices, with an Azure API Management Gateway in front of them.

This week I hit one of those classic Azure pitfalls that are incredibly difficult and time-consuming to track down.
<br/>
<br/>
## Somethings not right...

There was a minor string formatting bug in one of the microservices. No problem, easy fix.  

I write a patch, run my unit and integration tests. Perfect. Everything is looking nice, green and regression free. 

I do some manual testing locally and all signals are go. So I commit and push my changes and wait for CI and CD agents in Azure Pipelines to do their thing. 

Once the service is deployed, I start my last round of manual testing in our Azure Staging environment. 

... The changes I'd just deployed were not there. The endpoint was working, but the formatting bug was still there. 
<br/>
<br/>
## Trial and Azure

So I did some detective work:

1. Check Azure Pipelines - Everything deployed OK
2. Check Kudu - Logged into Kudu and checked all the directories with the console. Everything fine there too.
3. Wrote and deployed a patch that forced all of the strings in the endpoint's response payload to upper-case. A super obvious change, no chance I'm missing that... It didn't deploy either. 

After an hour or so, I was stumped. So I started reading through the Azure docs and [I found this hidden away in there](https://docs.microsoft.com/en-us/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring)

The part that caught my attention is below:
>WEBSITE_CONTENTSHARE
For consumption plans only. The file path to the function app code and configuration. Used with WEBSITE_CONTENTAZUREFILECONNECTIONSTRING. Default is a unique string that begins with the function app name.

Consumption Plans are scalable in ways that App Service plans are not, so when we can, we deploy our Functions to Consumption Plans, and the Function in question was on a Consumption Plan.

So I checked the Functions App Settings. Everything looked good. We had both a `WEBSITE_CONTENTAZUREFILECONNECTIONSTRING` and `WEBSITE_CONTENTSHARE`. 

At this point I got curious, and lucky I did otherwise this would have taken me days to find. 
<br/>
<br/>
## Suspicious Storage

When you create a new Function App instance in the Azure Portal, you need to create a couple of support resources, namely:
- A Storage Account; and
- An optional Application Insights instance

For Function Apps that run on an App Service Plan, the Function is deployed to a dynamically provisioned VM. 

**But...** in the case of Functions that run on Consumption Plans. The Function App is deployed to a **File Share** that is automatically created under the Storage Account for that Function App Instance.

So I opened up [Azure Storage Explorer](https://docs.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer?tabs=windows) and did some snooping around in the Storage Accounts list and I found this:
![_config.yml](/images/Storage.png)

