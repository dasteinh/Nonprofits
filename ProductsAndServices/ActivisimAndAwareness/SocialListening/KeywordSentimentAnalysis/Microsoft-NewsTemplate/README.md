Keyword Sentiment Analysis using Bing News Documentation
===========================================================

# Table of Contents
1. [Introduction](#introduction)
2. [Architecture](#architecture)
3. [System Requirements](#system-requirements)
4. [How to Install](#how-to-install)
5. [Architecture Deep Dive](#architecture-deep-dive)
6. [Model Schema](#model-schema)
7. [Reports Walkthrough](#report-walkthrough)
8. [Updating your pipeline](#updating-your-search-terms)
9. [Estimated Costs](#estimated-costs)



### Introduction

The Bing News template spins up a sophisticated search solution, finding you the most relevant articles using machine learning techniques. The template stands up an end-to-end solution that finds articles via the Bing News API, enriches the data using machine learning and stores these enrichments in Azure SQL. Users can then use pre-built Power BI reports that leverage Microsoft technology to start exploring the data and finding articles most relevant to them.
The template is aimed at nonprofit oganizations who are interested in finding articles relevant to their chosen search query. It supports multiple nonprofit segments, ranging from, a conservation organziations tracking violations in regulations, to an advocacy organization monitoring hate speech being reported.
This document provides a walkthrough of the architecture, a deep dive into every component, comments on customizability as well as information on additional topics like estimated costs. For any questions not covered in this document, please contact the team at <tsiazure@microsoft.com>.

### Architecture

![Image](Resources/media/ArchitectureOverviewDiagramBingNews.png)

The flow of the Bing News solution template is as follows:

1. Logic Apps finds articles via the [Bing News Search API](https://azure.microsoft.com/en-us/services/cognitive-services/bing-news-search-api/)
2. Logic App extracts the contents of the news article
3. Azure Function calls textual analytics cognitive services to wrok out the sentiment of the article 
4. Azure Function enriches the content of the news article with machine learning
5. Power BI imports data into it from Azure SQL and renders pre-defined reports

### System Requirements

Setting up the template requires the following:

-   Access to an Azure subscription

-   Power BI Desktop (latest version)

-   Power BI Pro (to publish the template to share with others)


### How to Install

Before diving into the components of the solution, we will go through how to set things up. To get started with the solution, navigate to the [this page]( https://github.com/Microsoft/Nonprofits/tree/master/ProductsAndServices/ActivisimAndAwareness/SocialListening/KeywordSentimentAnalysis/Microsoft-NewsTemplate) and click **Deploy to Azure**.

**Getting Started:** Starting page introducing the template and explaining the architecture.

**Azure:** Use OAuth to sign into your Azure account. You will notice you have a choice between signing into an organizational account and a Microsoft (work/school account).

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https://raw.githubusercontent.com/microsoft/nonprofits/master/ProductsAndServices/ActivisimAndAwareness/SocialListening/KeywordSentimentAnalysis/Microsoft-NewsTemplate" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

Log into Azure, will gives the application access to your Azure subscription and permits spinning up Azure services on your behalf. It also lists the estimated costs of the template. If you want a more granular breakdown of the costs, please scroll down to the [Estimated Costs](#estimated-costs) section.

**Target:** Connect to an existing SQL Server or provide details which the application will use to spin up an Azure SQL on your behalf. Only Azure SQL is supported for this template. If a user chooses to spin up a new Azure SQL, this will get deployed in their Azure subscription inside the newly created resource group.
![Image](Resources/media/image8.png)


**Search Terms:** Input the search terms you are interested in tracking. Articles that match your search terms will be found using the Bing News cognitive service via the Logic App. Logic Apps supports any queries that the Bing News API supports. You can preview your search results [here]( https://www.microsoft.com/cognitive-services/en-us/bing-news-search-api).
Some recommended practices are as follows:
If you would like to look for multiple search words please use the OR operators (e.g. Microsoft OR Azure)
If you would like to look for a specific phrase please use quotation marks (e.g. “Surface Pro”)
If you would like to exclude a word from your search pleas use – (e.g. Azure -Color)
If you would like to learn how you can change your search terms once the solution is deployed, please look at the [Updating your pipeline](#updating-your-search-terms) section.

**Summary:** Summary page outlining all the choices the user made.
![Image](Resources/media/image10.png)

**Deploy:** When you navigate to the deployment page the setup process gets kicked off. The following steps take place (we will do a deep dive into all of these resources):
-	SQL scripts run to create the necessary tables, views and stored procedures
-	Azure Functions get spun up
-	Storage accounts gets spun up
-	Azure ML commitment plan get spun up and the web services get deployed
-	Cognitive services get deployed
-	2 Logic Apps get spun up

![Image](Resources/media/image11.png)

**It is important that you do not navigate away from this page while deployment takes place. This process could take 10-15 minutes** Once everything gets deployed a download link will appear for a Power BI file which consists of the pre-defined reports.

![Image](Resources/media/image12.png)

**Power BI File:** Once you download the Power BI desktop file you will need to connect it to your data. Open the pbix and follow the instructions on the front page (**it is important you do this before publishing the file to the Power BI Service. If this is not done the solution will not work inside the Service).**

![Image](Resources/media/image13.png)

Architecture Deep Dive
======================

The following section will break down how the template works by going through all the components of the solution.

![Image](Resources/media/ArchitectureOverviewDiagramBingNews.png)

Azure Resources:
----------------

You can access all of the resources that have been spun up by logging into the Azure portal. Everything should be under one resource group (unless a user was using an existing SQL server. In this case the SQL Server will appear in whatever resource group it already existed in).

![Image](Resources/media/image14.png)

Here is an example of what gets spun up for a user. We will go through each of these items one by one:

![Image](Resources/media/image15.png)

### Logic App (LogicAppMainNews):

Logic Apps are an Azure service for building integrated solutions. You can easily build business logic flows that consist of various actions and triggers. The Logic App with the name ‘LogicAppMainNews’ is the main orchestrator for this solution template. We will go through each step and explain what the Logic App does: 

![Image](Resources/media/image16.png)

The first step inside the Logic App is called the trigger. This runs every 15 minutes and finds all the articles that match the Search Query (in this case the search query is Azure). All the published articles that mentioned the word Azure in the past 15 minutes are returned (the interval and frequency variable can be customized to make data collection less frequent), see the [Updating your pipeline](#updating-your-search-terms) section.

![Image](Resources/media/image17.png)

Here is an example of the response body returned:

![Image](Resources/media/image18.png)

 You can learn more about the Bing News API [here]( https://azure.microsoft.com/en-us/services/cognitive-services/bing-news-search-api/). 
Following the trigger, there are 3 parallel steps that run. All 3 of the steps consist of Azure Functions. Functions are a way of executing some code in a serverless experience. They are very handy for event based executions just like how we use them in the Logic App. To learn more about how to edit the function code please skip over to the Functions section.

![Image](Resources/media/image19.png)

TimeUtilsWebHook and TimeUtilsWebHook 2 are used to clean up the date field returned by the news trigger. This allows for clean slicing and dicing of data inside Power BI by fields like hour, day or month. The former Function looks at the ingest date (i.e. when did we process the article) and the latter looks at the publish time of the article (when was the article published).

![Image](Resources/media/image20.png)

The UrlCleaner function is used to extract out the website url from the news response body. An example call is shown here:

![Image](Resources/media/image21.png)

Once we extract out and clean up the URL from the trigger, we can now do a GET call to return the body of the article:

![Image](Resources/media/image22.png)

This still needs cleaning before we can consume it in a meaningful way:

![Image](Resources/media/image23.png)

The ArticleCleanerWebHook and HTMLStripperWebHook clean up any unicode and html tags to leave us with text in the following form:

![Image](Resources/media/image24.png)

In parallel to the cleanup process, we also have a Function that extracts out an image URL from the article. This will be later used to load news images into the reports:

![Image](Resources/media/image25.png)

The aim of the AbstractExtractor Function is 2 fold – firstly it programmatically finds a snippet that will be displayed in the document tile inside the Power BI. The snippet it finds is contextual to the search term the user has specified (e.g. if a user is looking for Azure, the snippet will contain Azure).

![Image](Resources/media/image26.png)

The AbstractExtractor step also extracts out all the search terms it finds mentioned inside the article (e.g. if I was searching for Azure OR Google OR “Amazon Prime” and the article mentioned Azure as well as Azure Prime, the abstract extractor step would return Azure and Amazon Prime as matches). This allows us to later filter the entire report on those keywords.

Following the ‘AbstractExtractor’ Function, we now have 3 parallel steps that run. 2 of those steps come from the textual analytics cognitive services – Detect Sentiment and Key Phrases.

![Image](Resources/media/image27.png)

Detect sentiment returns a sentiment score for the article (ranging from 0 being very negative to 1 being very positive). Key phrases is an unsupervised learning method that returns back all the trending keywords found inside the document.

You can learn more about the cognitive services used in these steps over [here]( https://docs.microsoft.com/en-us/azure/cognitive-services/cognitive-services-text-analytics-quick-start).

In conjunction to the cognitive services, we also have a Function (UrlDomainWebhook) that extracts out the source of the news article found. 

![Image](Resources/media/image28.png)

We have now extracted and enriched all the data we need. The InsertDocument step runs a stored procedure (sp_write_document) that writes all of the data we need into SQL.

![Image](Resources/media/image29.png)

The final SQL step writes all of the matched search terms into SQL (the ones found via the Abstract Extractor function). There could be multiple search terms associated and matched with one document (hence the for each statement).

![Image](Resources/media/image30.png)

![Image](Resources/media/image36.png)

### Logic App (LogicAppScheduler):

Once the data is written into SQL, there are some enrichments that are done across the data (as opposed to on a document by document basis). These run on a schedule (reoccurrence every 3 hours but this can be changed inside the Logic App).

![Image](Resources/media/image31.png)

The first step following the trigger is a stored procedure called sp_clean_stage_tables. This empties out the staging tables that we are 
going to use for storing the outputs of the machine learning enrichments. If there is any data left over in them after the last time the Logic App ran, we want those truncated.

![Image](Resources/media/image32.png)

The next two parallel steps call Azure Machine Learning web services (you can read more about Azure ML in the sections below).

![Image](Resources/media/image33.png)

The ‘Entity AzureML’ webservice extracts out all the people, location and organizational entities it finds in the articles. 

The ‘Topic AzureML’ web service carries out topic clustering using a machine learning technique called Latent Dirichlet Allocation (LDA). This looks at the text of all the articles found and groups the documents together to form relevant topics. 

The final Azure ML component is the ‘Topic Images’ web service. The aim of this module is to extract out the relevant image URLs that go with the topics found. These are used to augment the topic clustering visual found in the Power BI report.

![Image](Resources/media/image34.png)

The final two components of this Logic App are stored procedures. So far, all of the Azure ML web services have been writing data into the previously cleaned out staging tables. The mergedata stored procedure replicates the data from the staging tables to their corresponding production tables.

The final stored procedure (create_topic_key_phrase) finds the keyword descriptors that correspond to the numerical values of the topic clusters (as LDA only returns the numerical equivalents). 

![Image](Resources/media/image35.png)

![Image](Resources/media/image37.png)

![Image](Resources/media/image38.png)

### Cognitive Services:

We already showed how the Azure Cognitive Service for Textual Analytics as well as Bing News get used inside ‘LogicAppMainNews’. This section will touch on the configurations that can be done to the cognitive services inside the portal.

Most importantly, a user is able to change the SKU they want to use for the Cognitive Service. As a default, we set the text analytics cognitive service to S1 (100K calls) which costs $150 per month. Depending on the anticipated traffic you can change the SKU to meet your needs (please remember the textual analytics cognitive service is used for both sentiment and key phrase extraction. This means that if e.g. you are processing 10K documents a month that translates to 20K calls to the textual analytics API).

![Image](Resources/media/image39.png)

The Bing News cognitive service is by default set to an S2 SKU (10K calls a month – one article = one call). This can be changed inside the portal just like in the textual analytics cognitive service case:

![Image](Resources/media/image40.png)

![Image](Resources/media/image41.png)

### Azure Function: 
This documentation has already covered the purpose of each Azure Function at a high level in the Logic App section. By default, you cannot edit the functions as they are deployed via continuous integration to GitHub (if you try to edit the function you will see a 
Ready only message at the top):

![Image](Resources/media/image42.png)

In order to edit the functions you will need to navigate to Function app settings and then click on ‘Configure Continuous Integration’. 

![Image](Resources/media/image43.png)
You can now disconnect the function from source control and edit the functions freely. **Please do not Sync the function from GitHub after deployment. There is a dll used by the Article Extractor function that is not available on GitHub. If you sync manually you will lose access to this dll and will have to set up the solution from scratch again**.

![Image](Resources/media/image44.png)

You can view the GitHub we sync our functions from over [here]( https://github.com/Microsoft/Nonprofits/tree/master/ProductsAndServices/ActivisimAndAwareness/SocialListening/KeywordSentimentAnalysis/Microsoft-NewsTemplate/Service/NewsFunction).

![Image](Resources/media/image45.png)
![Image](Resources/media/image46.png)
![Image](Resources/media/image47.png)

### Azure ML Web Services
The function of the Azure Machine Learning web services we spin up have been described in the ‘LogicAppScheduler’ section. 

Unfortunately, there isn’t much configuration that can be done with the Azure ML steps as it is only the web services that are deployed and not the actual experiments. 

The reason behind this is deploying the raw experiments would severely slow down the provisioning process. If you would like to edit the Azure ML experiments you can do so by:

[Approach needs to be verified and documentation updated]

![Image](Resources/media/image48.png)

### Connectors
You will notice there are a number of steps with a chain icon. These are API connections that are used inside the Logic App to authenticate and connect to various services like SQL, Cognitive Services and Azure ML. These services store your credentials securely and can be updated if your credentials/keys ever change.
For example, editing my SQL API connection looks like this:
![Image](Resources/media/image51.png)

### Model Schema

Here is an overview of the tables found in the Power BI (names correspond to the Power BI tables not the underlying SQL tables or views):

| **Table Name**       | **Description **                                                                                                                                                                                                                                              |
|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Article | Stores all the metadata about the article (snippet, source URL, news category, image URL, published time etc.)                                                                                                                                                                                  |
| Article Search Terms | Stores the search term and corresponding document ID. There can be multiple search terms found in one document                                                                                                                                                                                  |
| Article Topics | Stores the topic ID each document is associated with as well as a numerical representation of the key phrases found.                                                                                                                                                                                 |
| Compressed Entities        | Document ID and an JSON object of the entities found. The document strippet custom visual uses the JSON to visualize the entities that appear in an article hence the JSON structure. The JSON stores the entity value, entity type as well as a CSS color and class.                                                                                                                                                                                |
| Entities | Stores the same information as the Compressed Entities table but in a structured format vs. JSON. |
| Key Phrases  | Stores the key phrases found with the corresponding document ID. A key phrase can belong to many documents and a document can belong to many key phrases)                                                                                                                                                                      |
| Sentiment Scores  | Stores the sentiment of an article along with the binned sentiment score and categorical score|
| Topic Images       | Stores the topic ID and up to 4 image URLs that are found to be associated with the given topic|
| Topic Key Phrases| Stores the topic ID with the word version of the key phrases that are used in the reports instead of the numerical output of the LDA model|

Below is a breakdown of the columns found in every table:

| ** Article ** |                                          |
|--------------------------|------------------------------------------|
| **Column Name**          | **Description**                          |
| Id                 | Document Id (unique)     |
| Abstract                 | Short snippet from the news article     |
| Title                   | Title of the news article                              |
| Source URL             | News article URL|
| Source Domain                  | News publisher URL               |
| Category            | Category of news article |
| Image URL                   | URL of image from news article                              |
| Image Width| Width of image in pixels|
| Image Height                  |Height of image in pixels               |
| Sentiment score            | Sentiment of article (from 0 – 1) |
| PublishedTimestamp| Timestamp when article was published |
| Published Month Precision| Month article was published in                           |
| Published Week Precision | Week article was published in                           |
| Published Day Precision | Day article was published on                           |
| Published Hour Precision | Hour article was published in                            |
| Published Minute Precision | Minute article was published |
| Ingested Month Precision| Month article was ingested |
| Ingested Week Precision | Week article was ingested |
| Ingested Day Precision | Day article was ingested |
| Ingested Hour Precision | Hour article was ingested |
| Ingested Minute Precision | Minute article was ingested |
| Count Entities | Number of entities found in article |


| **Article Search Terms** |                                          |
|--------------------------|------------------------------------------|
| **Column Name**          | **Description**                          |
| Document Id                 | Document Id (can be repeated)    |
| Search Terms                   | The specific search term found inside a given document. Multiple search terms can be found in a single document                              |

| **Article Topics** |                                      |
|--------------------|--------------------------------------|
| **Column Name**    | **Description**                      |
| Document Id                 | Document Id (a document belongs to one topic)     |
| Topic Id                 | Id of the topic cluster     |
| Document Distance                   | A score between 0 and 1 that represents how well a document fits within a topic.  0 is perfect and 1 is very poor.  
|
| Topic Score             | The number of documents in each topic|
| Topic Key Phrase                  | Numerical representation of the keywords a topic is associated with               |
| Image URL 1            | First image associated with topic |
| Image URL  2                | Second image associated with topic                            |
| Image URL  3                | Third image associated with topic                            |
| Image URL  4                | Fourth image associated with topic                            |
| Weight | Converts the document distance to scale of 0 to 100 with 100 as the highest. Higher weighted documents are a better fit for the topic cluster.  The formula is (1 - distance) * 100.
 |

| **Compressed Entities** |                                      |
|--------------------|--------------------------------------|
| **Column Name**    | **Description**                      |
| Document Id           | Document Id (unique) |
| Compressed Entities Json              | JSON object of all the entities found inside the article. For a more detailed breakdown of the Json object please look at the entities table which has a structured view of all the fields.            |

| **Entities**    |                                                                                                                   |
|-----------------------|-------------------------------------------------------------------------------------------------------------------|
| **Column Name**       | **Description**                                                                                                   |
| Document Id           | Document Id (not unique – a document can have multiple entities associated with it) |
| Entity Type              | Categorical variable of the type of entity found. ORG refers to Organization, LOC to Location and PER to Person|
| Entity Value              |The actual name of the entity that was found |
| Offset           |Character position of the start of the entity within the document.  |
| Offset Document Percentage         |Offset divided by document length.  A value of .25 indicates that the entity is found 25% through the document |
| Length           | Entity character length |
| Entity ID           | Merge of entity type with entity value (unique) |
| Entity Class           | Font awesome icons (scalable vector icons) |

| Entity Color           |Color (specified in hex) |



| **Key Phrases**         |                                                                                                                                                                                                           |
|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Column Name**             | **Description**                                                                                                                                                                                           |
| Document ID                   | Document ID (one document can have multiple key phrases)                                                                                                                                                     |
| Phrases                    | A key phrase found inside an article |

| **Sentiment Scores**        |                                                                                                                        |
|-------------------|------------------------------------------------------------------------------------------------------------------------|
| **Column Name**   | **Descrpition**                                                                                                        |
| Id  | Document ID (unique – each document will have one sentiment score)                                                                          |
| Score              | Sentiment score ranging from 0 (very negative) to 1 (very positive)|
| Binned sentiment         | Sentiment grouped into intervals of 0.05 (e.g.  sentiment of 0.723 would fall into the 0.7 bucket)                                                |
| Sentiment         | Categorical variable for sentiment – categories include Very Negative, Slightly Negative, Neutral, Slightly Positive and Very Positive|

| **Topic Images**         |                                                                                                                                                                                                           |
|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Column Name**             | **Description**                                                                                                                                                                                           |
| Topic Id                   | Id of the topic cluster                                                                                                                                                     |
| Image URL 1            | First image associated with topic |
| Image URL  2                | Second image associated with topic                            |
| Image URL  3                | Third image associated with topic                            |
| Image URL  4                | Fourth image associated with topic                            |

| **Topic Key Phrases**         |                                                                                                                                                                                                           |
|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Column Name**             | **Description**                                                                                                                                                                                           |
| Topic Id                   | Id of the topic cluster                                                                                                                                                     |
| Key Phrase                    | The word descriptor of the topic cluster (usually 3 keywords) |

### Report Walkthrough
------------------

The following section walks through each report page outlining the intent of the page. 
In the sample report below we are tracking the Microsoft keyword and have been collecting data for the past 2 weeks. If you have just set up the template and don't see any data please be patient – the Azure ML experiments need to run prior to the topic clustering and entity extraction visuals loading. This could take a few hours (depending on whether you configure the schedule).

![Image](Resources/media/image52.png)

The template uses many custom visuals built by Microsoft Research for effective information retrieval. On the left hand side we can see the topic clustering visual (this is the output of the Azure ML Topic web service). This visualizes all the clusters generated by the machine learning model with the size of the visual determined by how many documents are present in a cluster. We also visualize the top 3 keywords that are associated with a cluster across the center.

The middle visual is the entity extraction visual that meaningfully represents the organizations, locations and people found inside the articles. This visual represents the output of the Azure ML entity web service. The colors of the entities are defined by the hex colors specified in the Entity Color field and the icons that appear next to each type of entity is specified by the Entity Class. 

The document strippet visual represents each article by an image, title and source. Clicking into a specific tile shows a short contextual snippet taken from the article. When clicking into a tile you will also notice more Entity Class icons appearing on the side. Hovering over these icons shows the specific entity they are associated with. The icons that appear represent all the entities found inside each article (and this visual is generated via the Compressed Entities JSON field).

Upon finding an article of interest you can navigate to read the full article by clicking on the article link.

![Image](Resources/media/image58.png)

Finally we also have a list of trending keyphrases (output of the key phrase cognitive service) and the average sentiment (output of the sentiment cognitive service).

As you can see, the overview page brings together the outputs of 4 different machine learning algorithms!

The following pages do a deep dive into one specific machine learning area but the intent of each page is the same - connect you with the articles most relevant and interesting to you. 

### Sentiment Analysis

The sentiment page allows you to answer questions like which entities linked to Microsoft are being perceived most negatively in the news as well as help you uncover why that is the case.

![Image](Resources/media/image53.png)

The left-hand side and middle visuals let you quickly identify and topics or entities that are consistently being talked about in a positive or negative way. The report displays sentiment on the x axis and the volume of documents on the y axis. Documents are grouped by either topics (left side) or entities (middle). Articles appearing in the top left hand corner of the graph are a cause of concern (topics/entities that are consistently negative) whereas top right hand corner articles indicate lots of positivity.

You can also drill into a specific sentiment bin to quickly isolate e.g. only the most negative articles or only the neutral ones. You can also see how sentiment changes across time.

### Topics

The topics page gives you added perspectives like which topics were trending last vs. this week. The trending key phrases complement the topic visualization very well. Upon drilling into  a topic, explore the keywords that are trending within the topic to get a very quick understanding of what the topic describes.

![Image](Resources/media/image54.png)

### Entities

We can dive deeper into entities by e.g. visualizing all of the location entities on a map. We can also take a look at which entities have been trending across time.

![Image](Resources/media/image55.png)

### Phrases

The phrases page allows us to explore which keywords the machine learning models have picked up as being trending, both on aggregate as well as across time. We can also see key phrases by sentiment – which phrases are associated with a negative vs. positive sentiment.

![Image](Resources/media/image56.png)

### Source

Finally, the sources page allows us to explore articles from a news publisher’s perspective. We can see which news publishers write about our search terms most frequently, and which are the most popular topics or entities they bring up.
If we want to find news articles to read from a certain publisher this is another nice way of filtering down the data.

![Image](Resources/media/image57.png)

### Updating your Search Terms
---------------------

Once you set up the solution template you may want to modify the search terms you are looking at.
If you want to change your search terms you will need to log into your Azure portal account and open your Logic App (LogicAppMainNews). Inside the Logic App, select Edit, and the Logic App Designer will load.  Depending on your screen resolution, you'll go to the first step, which may require scrolling.  "On news article" ha a property called "Search Query":

![Image](Resources/media/image17.png)

The Search Query can be edited and saved the Logic App to update the query. This however will not wipe out the data from the database – it will just continue accumulating new data on top of the old data.

If you would like to completely wipe out the data before updating the search term you will need to do this manually by running the SQL scripts found inside our GitHub [here](https://github.com/Microsoft/Nonprofits/tree/master/ProductsAndServices/ActivisimAndAwareness/SocialListening/KeywordSentimentAnalysis/Microsoft-NewsTemplate/Service/Database). 

Running these scripts in order will clean up and recreate the database, views and stored procedures needed for the solution.

In addition to changing the search terms, the trigger also can be configured to run at various intervals.  To save costs these interval and frequency can be modified in coordination with the other trigger for the preperation of data augmentation in the 

### Estimated Costs

Here is an estimate of the Azure costs (Logic Apps, Azure Functions, Azure SQL, Azure ML, Cognitive Services) based on the number of articles processed:

Processing 10K articles a month will cost approximately $420

Processing 50K articles a month will cost approximately $875

Processing 100K articles a month will cost approximately $1443

Please keep in mind these are estimated costs and subject to change. For a more detailed breakdown of the various components please refer to the [Azure calculator](https://azure.microsoft.com/en-us/pricing/calculator/) and select Logic App, Azure Function, Azure SQL, Cognitive Services and Azure ML. You can tweak all the options to see what the costs will look like and what modifications may suit your needs best.

The following defaults are set for you in the template (you can modify any of these after things get set up):

-   Azure SQL: Standard S1

-   App Service Plan: Dynamic

-   Logic App 1 (trigger set for every 15 minutes), 14 actions executed

-   Logic App 2 (trigger set for every 3 hours), 7 actions executed

-   Azure Functions (approximately 30 seconds run per article)

-   Azure ML (S1)

-   Cognitive Services – Text Analytics S1

-   Cognitive Services – Bing Search S2

For example, if you know you will be processing very few articles a month, you could change the SQL Server from S1 to Basic. 
Whilst the default setting should cater to most news template requirements, we encourage you to familiarize yourself with the various pricing options and tweak things to suit your needs.


### Azure ML Commitment Plan

For the Azure ML components, we spin up an S1 commitment plan. This allows us to have a total of 25 hours of compute a month, up to 10 web services and 100K transactions a month for $100. 

![Image](Resources/media/image49.png)

![Image](Resources/media/image50.png)
