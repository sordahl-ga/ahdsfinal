# Challenge-06  - FHIR Export for Data Analytics and Machine Learning in Azure

## Introduction
Welcome to Challenge-06!

In this challenge, you will be using [FHIR Analytics Pipelines](https://github.com/microsoft/FHIR-Analytics-Pipelines) (OSS) to export data from the FHIR service for analytics and ML model development using Azure Synapse and Azure ML.

## Background
Health data aggregated in FHIR offers rich analytics potential for medical research of all sorts. With the FHIR service in Azure Health Data Services, FHIR data can be exported for use in Azure Synapse and Azure ML - giving researchers an end-to-end pipeline for health data ingestion, exploration, and model training. Practitioners can then use the models to run inference on patients' data in clinical workflows - leading to improved health outcomes for patients.

### Learning Objectives for Challenge-06
By the end of this challenge you will be able to 

* implement AHDS Synapse Sync Agent (Parquet Export with Near Real Time Sync on FHIR Events)
* create external SQL Table representations of data in Synapse Serverless Pools
* create a SQL View for exploration and analysis of length of stay (LOS) and cost variables
* create a Synapse Notebook for Exploration and Analysis
* create and connect to an Azure ML Workspace
* export Features for Auto ML Training
* register Trained Model
* deploy model to an Azure ML Endpoint
* call the Azure ML model
* create an Azure logic app decision workflow
* have the decision workflow call Azure ML scoring model
* decision workflow triggered by AHDS events (extra)
* explore data with PowerBI (extra)


### Prerequisites 
* FHIR service deployed (completed in Challenge-01)
* FHIR data imported into FHIR service (completed in Challenge-03 and Challenge-04)
* A copy (clone) of this repo on your local PC

## Getting Started

In this challenge, you will be exporting data from your FHIR service to Azure Synapse for data exploration and analytics. In Synapse studio, you will use Azure AutoML to train a predictive scoring model. You will then publish and consume the model in realtime. 

## Part 1. FHIR data export to ADLS and Azure Synapse

### Step 1 - Create a Synapse Workspace
1. Open the Azure portal, and at the top search for Synapse.
2. In the search results, under Services, select Azure Synapse Analytics.
3. Select Add to create a workspace.
4. In the Basics tab, give the workspace a unique name. We'll use xxxahdschallengews in this document
5. Under Select Data Lake Storage Gen 2, click Create New and name it xxxahdshallengelake.
6. Under Select Data Lake Storage Gen 2, click File System and name it users.
7. Select Review + create > Create.

Your workspace should be ready in a few minutes.

### Step 2 - Deploy the FHIR to Synapse Sync Agent
1. Deploy FHIR Analytics Pipelines and export data in Parquet format to ADLS v2. This deployment will also create a SQL Database representation of the data that you can access from Azure Synapse or any client able to connect to SQL Server.
You should follow all of the deployment instructions detailed in the [FHIR to Synapse Sync Agent](https://github.com/microsoft/FHIR-Analytics-Pipelines/blob/main/FhirToDataLake/docs/Deployment.md) site before proceeding

### Step 3 - Reset Your SQL Admin Password
1. From the Azure Portal Navigate to your Synapse Workspace
2. Select Reset SQL Admin Password
3. Change to a compliant password and make a note of it.

Access Synapse Studio from the Portal and check out how your data is organizaed by FHIR Resource type in your serverless SQL Database.

## Part 2. Data Analysis and Model Training
How well do certain factors in patients' health data predict patients' healthcare costs and/or length of stay (LOS) in treatment facilities? Can we use the insights we gained to predict multi-day length of stays to help initiate cost and quality control measures.

### Step 1 - Configure your Synapse Environment for Analytics 
#### Create an Apache Spark Pool
1. From Synapse Studio Select the Manage Icon from the Left Navigation Bar
2. Select Apache Spark Pools
3. Name Your new Spark Pool AHDS24MLPOOL
4. Leave All Other Basics Settings as Defaulted
5. Select Additional Settings Tab
6. On the Apache Spark DropDown select version 2.4
7. Select Review & Create
8. Select Create
#### Upload Needed Resources into Blob Storage
1. From Azure Portal Access your Synapse Workspace Storage Account
2. Select Containers
3. Click the ```+ Container``` button
4. Name the container `resources`
5. Click the Create Button
6. Select the resources container
7. Click the ```Upload``` button
8. Click the select a file icon
9. Browse to the resources directory of challenge-6
10. Select all files in directory
11. Click Open
12. Click Upload button

### Step 2- Create a SQL View of Data
We will combine some FHIR resource extract data into a SQL View to help us better identify features that might impact multi-day LOS and encounter costs.  The main driver will be condition data from Encounters with co-related Claim, Patient, Procedure data joined in.
1. In Azure Synapse Studio Select the Develop Icon from the the Left Navigation Bar
2. Select the `+` sign to add a new resource
3. Select SQL Script
4. Change the Use Database dropdown to 'fhirdb'
5. Paste the following SQL Code block into the SQL Script Tab
```
 CREATE VIEW fhir.ConditionView as
 SELECT JSON_VALUE([Condition].[code.coding],'$[0].code') as "ConditionCode",
 [Condition].[code.text] as "ConditionDescription",
  DATEDIFF(year,[Patient].[Birthdate],GETDATE()) as "Age",
 [Patient].[Gender],
 [Patient].[PostalCode],
 [Patient].[Country],
 [Encounter].[LOSDays],
 [Encounter].[Encounter-Type] as EncounterType,
 [Encounter].[Encounter-Description] as EncounterDescription,
(SELECT COUNT(*) as "ProcedureCount" FROM [fhir].[Procedure] WHERE [Procedure].[encounter.reference]=[Condition].[encounter.reference]) as "ProcedureCount",
(SELECT SUM([total.value]) as "EncounterCost" FROM [fhir].[Claim]
 WHERE [fhir].[Claim].[patient.reference]=[Condition].[subject.reference] 
 AND CONVERT(datetime,SUBSTRING([billablePeriod.start],1,19))>=[Encounter].[Encounter-StartDate] 
 AND CONVERT(datetime,SUBSTRING([billablePeriod.end],1,19))<=[Encounter].[Encounter-EndDate]
 ) as "EncounterCost"
 FROM [fhir].[Condition] as [Condition]
 INNER JOIN (
   SELECT
     [id],JSON_VALUE([name],'$[0].family') as "name",CONVERT(datetime,[birthDate]) as "Birthdate",[gender] as "Gender",
     JSON_VALUE([address],'$[0].postalCode') as "PostalCode",
     JSON_VALUE([address],'$[0].country') as "Country"
   FROM [fhir].[Patient] 
    AS [Patient]
 ) [Patient] on [Patient].[id]=SUBSTRING([Condition].[subject.reference],9,100)
 INNER JOIN (
   SELECT
     [id],
     JSON_VALUE([type],'$[0].coding[0].code') as "Encounter-Type",
     JSON_VALUE([type],'$[0].coding[0].display') as "Encounter-Description",
     CONVERT(datetime,SUBSTRING([period.start],1,19)) as "Encounter-StartDate",
     CONVERT(datetime,SUBSTRING([period.end],1,19)) as "Encounter-EndDate",
     DATEDIFF(day,CONVERT(datetime,SUBSTRING([period.start],1,19)),CONVERT(datetime,SUBSTRING([period.end],1,19))) as "LOSDays"
     FROM [fhir].[Encounter] 
    AS [Encounter]
 ) [Encounter] on [Encounter].[id]=SUBSTRING([Condition].[encounter.reference],11,100)
 
```
6. Press the Run Button
7. The new ConditionView should now be created in your fhirdb database.
### Step 3 - Explore Data with SQL Server
1. Replace the SQL Code block in SQL Script 1 window with the following:
``` select * from fhir.ConditionView ```
2. Press the Run Button
3. The query results may take several minutes
The results contains a tabular view of the data we joined as described above. You can also use the chart component to visual the query results.

### Step 4 - Import the ConditionAnalysis Notebook into Synapse
1. In Azure Synapse Studio Select the Develop Icon from the the Left Navigation Bar
2. Select the `+` sign to add a new resource
3. Select Import
4. Browse to the notebook directory of challenge-6 and select the ```ConditionAnalysisNotebook.json``` file.
5. Click Open
6. On the Attach To dropdown select the AHDS24MLPOOL
7. On the Language dropdown select PySpark(Python)

<I>In the following steps we will be traversing and executing cells of functionality in the Synapse Notebook we just imported. Please see this [link](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-development-using-notebooks) for information on Synapse Notebooks.</I>

### Step 5 - Configure the ConditionAnalysis Notebook
1. In the Cell ```Import Libraries and Set Environment Variables``` you will need to replace variables containe in ```<>``` with the values for your synapse envrironment 
2. Execute the ```Import Libraries and Set Environment Variables``` cell in the notebook after defining variables

### Step 6 - Loading, Exploring and Augmenting Data
1. Read and Execute the following cells in order. Theses cells are related to exploring data in PySpark environment: </br>
```Load the ConditionView into a DataFrame```</br>
```Calculate the MultiDayLOS flag```</br>
```Scatter Plot to see effect of Age on EncounterCost```</br>

### Step 7 - Training a model using Azure Auto ML
1. Read and Execute the following cells in order. Theses cells are related to configuring and running training experiments in Azure Machine Learning using AutoML: </br>
```Select Features for Auto ML Training```</br>
```Create Azure ML Workspace```</br>
```Connect to Azure ML Workspace```</br>
```Convert Training Data to Azure ML TabularData Set```</br>
```Define Training settings```</br>
```Configure Auto ML```</br>
```Train the automatic regression Model```</br>
```Retrieve the best model```</br>
```Test Model Accuracy```</br>
```Root Mean Square```</br>
```MAPE and Accuracy```</br>
```Model fitting to Data Test```</br>
### Step 8 - Register and Publish Predictive Model
1. Read and Execute the following cells in order. Theses cells are related to publishing a scoring model to Azure Machine Learning: </br>
```Register Model to Azure ML for Publication```</br>
```Initialize Scoring Environment```</br>
```Copy Score Script from Blob Storage```</br>
```Deploy Model Azure ML Endpoint```</br>
```Test Deployed Service```</br>

## Part 3. Eventing workflow based on Azure ML scoring (Bonus) 
>Using Azure Health Data Service Eventing and the Predictive Model we published above, can you invoke the model as part of an event driven decision support process?
Please refer to this [tutorial](https://learn.microsoft.com/en-us/azure/healthcare-apis/events/events-consume-logic-apps) as a guiding example.

## Part 4. Data visualization with Power BI (Bonus)
> If you are in possession of a [Power BI license](https://docs.microsoft.com/power-bi/fundamentals/service-features-license-type), please refer to this [tutorial](https://learn.microsoft.com/en-us/azure/synapse-analytics/get-started-visualize-power-bi) on exploring data with Power BI in Azure Synapse


### What does success look like for Challenge-06?
+ Successfully export data from FHIR service using Synapse Sync Agent (OSS)
+ Successfully explore data in synapse using SQL Views
+ Successfully use the data for statistical analysis and ML modeling
+ Successfully publish a scoring model for realtime consumption

## Next Steps

Click [here]() to proceed to the next challenge.
