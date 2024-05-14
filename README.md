# Azure-end-to-end-data-engineering-pipeline
Azure Data Engineering Project:
1.	Azure Data Factory
2.	Azure Data Lake Storage Gen2
3.	Azure Databricks
4.	Azure Synapse Analytics
5.	Azure Key vault
6.	Azure Active Directory (AAD) and
7.	Microsoft Power BI
The use case for this project is building an end-to-end data solution by ingesting the tables from on-premises SQL Server database using Azure Data Factory and then store the data in Azure Data Lake. Azure databricks is used to transform the RAW data to the cleanest form of data and we are using Azure Synapse Analytics to load the clean data. Finally, Microsoft Power BI is used to integrate with Azure synapse analytics to build an interactive dashboard. Also, we are using Azure Active Directory (AAD) and Azure Key Vault for monitoring and governance purposes. Please go through this document to understand the step-by-step process of how I have implemented this project. The data source for this project is GitHub – Adventureworks database provided by Microsoft.

Projects linked deployed:
GitHub Repository: All the project data files with code are attached on the GitHub repository.
https://github.com/ridhiimayenugu/Azure-end-to-end-data-engineering-pipeline

Azure Data Factory Data Ingestion Pipeline:
https://adf.azure.com/en/authoring/pipeline/DataIngestion%20Pipeline?factory=%2Fsubscriptions%2Fe5717317-972d-4a27-bb81-6fbfb0e4f3ad%2FresourceGroups%2Fridhi-data-engineering-project%2Fproviders%2FMicrosoft.DataFactory%2Ffactories%2Fpipeline-app-adf

Azure Databricks Data Transformation:
https://adb-19467951917785.5.azuredatabricks.net/browse/folders/107474624241936?o=19467951917785

Azure Synapse Analytics Data Loading:
https://web.azuresynapse.net/en/authoring/orchestrate/pipeline/Create%20View?workspace=%2Fsubscriptions%2Fe5717317-972d-4a27-bb81-6fbfb0e4f3ad%2FresourceGroups%2Fridhi-data-engineering-project%2Fproviders%2FMicrosoft.Synapse%2Fworkspaces%2Fsynapseload

Power BI report published to service:
https://app.powerbi.com/groups/me/reports/0436e390-b157-4499-b783-01505b8188bd/ReportSection?experience=power-bi

Important Note: All above links are deployed on Azure portal. 


Environment Set up:
1.)Create resource group on Azure portal:
 
 2.) Download SQL Server Instance and SQL Server Management Studio.
 3.) Create a backup of the Adventureworks database from GitHub and restore backup on SSMA.
 4.) Establish connection between SQL Server Instance and Azure.

Data Ingestion: Using Azure Data Factory
1.)Create self-hosted integration runtime to connect to on premises SQL Server.
 
 



2.)Create a pipeline with ‘copy data’ transformation and create a linked service to our on premises SQL Server
3.) Give azure data factory pipeline the access to key vault secrets
 

4.) Data ingestion Part 1: Copy of address table. 
Use copy table transformation and define “source” – establish connection to on premises sql server and connect. After the source table gets loaded define “sink” and establish connection to Azure data lake storage Gen 2 ( bronze folder). Click on debug and run the pipeline. After successfully executing the pipeline – revalidate if the file is loaded on container bronze folder in data lake in parquet format. Deploy and publish all sources.
 

 

5.) Data Ingestion Part 2: Copy all tables at once from SQL Server to ADF.
Create SQL script for schema and all tables under it
 
Create a lookup activity on ADF – connect to the ‘onpremsqlserver’ linked service we already have ( created this in above step while copying just the address table). Load all table and give the sql script in “Query” option. Uncheck the first row only checkbox. Click on debug and run it. Check output after success.
 

 

Create a foreach activity and connect with look up activity “on success”.  Go to setting in foreach activity. Click on “items”>Dynamic. Pass the lookup all tables value (@activity('Look for all tables').output.value) as an item for the foreach activity.
 


Go to the foreach activity and insert the copy table activity. Include the following script in the query option after connecting to onpremsqlservice linked service - @{concat('SELECT * FROM ', item().SchemaName, '.', item().TableName)}
The script will do the following – It will get the schema name and table name from the lookup activity and the foreach activity will get these details and it will it in the copytable activity. This process will be repeated iteratively for all tables in the schema.
 

After source – define sink ( connect to datalake gen2 > parquet > bronze folder). Now define the folder structure as follows.
 

Create parameters in the sink dataset for schemaname and tablename and give values to these parameters from the item values.
 
Now click open > connection > define the directory structure and file name as follows:
@{concat(dataset().schemaname, '/', dataset().tablename)}
@{concat(dataset().tablename,'.parquet')}

 

Click on publish all the activities we created in this pipeline
 

Will use “add trigger” option to run this pipeline now. This will ingest all data into the specified directory and file paths we have given earlier in the bronze folder.
 

 




Data Transformation:
Create azure databricks resource and add to our project. Launch the workspace and create a cluster.
In this project we are connecting Azure databricks to Azure data lake storage by enabling the credential pass through option while creating the compute cluster.
For this we need to add roles in IAM access control as “Storage blob container contributor” for our azureid.
 

Mount the storage account:
Mount azure data lake storage to DBFS using credential passthrough
configs = {
  "fs.azure.account.auth.type": "CustomAccessToken",
  "fs.azure.account.custom.token.provider.class": spark.conf.get("spark.databricks.passthrough.adls.gen2.tokenProviderClassName")
}

# Optionally, you can add <directory-name> to the source URI of your mount point.
dbutils.fs.mount(
  source = "abfss://bronze@workproject.dfs.core.windows.net/",
  mount_point = "/mnt/bronze",
  extra_configs = configs)
After this mount the silver and gold containers as well.
 
Validate the data: dbutils.fs.ls("/mnt/bronze/SalesLT")
Output will list all files present here.

Transformation 1: Bronze layer to Silver layer
-Mount to bronze layer
-PySpark DataFrame to read the input path
-Display output created from the spark job
-We must write the code to modify the data format for modified date column. Refer below for the code and result snapshots.
Note: We can use magic commands in databricks to change language. For example for sql we can use: %sql.
 

Now do transformation on all tables:
Create  array table_name and list all tables
 
 

Transform all date time columns of all the tables to date format and put them in silver layer in delta format. We can validate the data loaded by checking on data lake storage in silver folder.

 
 







Data transformed has been loaded to silver layer:
 

















Transformation 2: Silver layer to Gold layer
Correcting the column names to business standard protocols
And then loading to gold layer in delta format. Validate by navigating to gold layer folder on data lake to check for successful data load. Snapshot for code and result are as below.
 

 


Transformation Part 3: Create pipelines for bronze to silver and silver to gold notebooks on ADF.
First we need to establish a connection between ADF and Azure databricks through linked service. Set up the key and secrets and create a linked service after generating access token in the databricks notebooks user settings. Publish the saved changes afer successfully connecting.
 

Add the databricks notebook activity to our already created DataIngestion pipeline. Create one for bronze to silver and one for silver to gold and execute the pipeline.
 

Pipeline successfully executed:
 



Data Loading: Azure Synapse Analytics:
Create a azure synapse workspace in our data engineering pipeline resource group and configure our storage account and  launch the workspace. 
Create as linked service with our gold layer and connect- serverless is used here.
 

Create a pipeline for dynamically creating views for all the tables
Go to Develop and create a stored procedure with the following script and publish it.
 



Create a linked service with serverless DB that is Azure SQL database and publish the changes
 
Use this linked service connection to access the stored procedure created. Go to Integrate tab and select create pipeline > metadata activity and specify the gold container from data lake storage. After this, Go +New and add Child items to get all data from SalesLT location.

 
Create a foreeach activity and link success to getmetadata activity.Need to pass the child item array from the getmetadata activity.
Add this in the settings>items section : @activity('Get Tablenames').output.childItems
Now all chilitems will be passed as input to the foreach loop.
Now click on edit items in the foreach activity and select stored procedure activity. We are doing this to reference to the stored procedure we created in the servelessSQLdb. Create the parameter for ViewName and pass the value @item().name
 










Publish all the changes and run the pipeline. Pipeline executed successfully and gold_db has all the views added.
 
 









Power BI for Reporting:
Open Power BI desktop and connect to the data loaded Azure Synapse Analytics. Report created and published on Power BI service.
 

Security and Governance:
IAM access control – create security groups. Create the security group using Azure active directory and assign the required roles.
End to End Pipeline Testing:
Create automated triggers on ADF pipeline and schedule it accordingly.






