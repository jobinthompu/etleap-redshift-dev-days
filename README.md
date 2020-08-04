# Build an Amazon Redshift data warehouse with Etleap

[Introduction](https://github.com/jobinthompu/etleap-redshift-dev-days#introduction)\
[Prerequisites](https://github.com/jobinthompu/etleap-redshift-dev-days#prerequisites)
1. [Set up an AWS VPC with Redshift, Glue, and S3](https://github.com/jobinthompu/etleap-redshift-dev-days#1-set-up-an-aws-vpc-with-redshift-glue-and-s3)
2. [Connect Etleap to Redshift and the data sources](https://github.com/jobinthompu/etleap-redshift-dev-days#2-connect-etleap-to-redshift-and-the-data-sources)\
	2.1 [Set up the S3 Input connection](https://github.com/jobinthompu/etleap-redshift-dev-days#21-set-up-the-s3-input-connection)\
	2.2	[Set up the MySQL connection](https://github.com/jobinthompu/etleap-redshift-dev-days#22-set-up-the-mysql-connection)\
	2.3 [Set up the Redshift connection](https://github.com/jobinthompu/etleap-redshift-dev-days#23-set-up-the-redshift-connection)		
3. [Create Etleap ETL pipelines from the sources to Redshift](https://github.com/jobinthompu/etleap-redshift-dev-days#3-create-etleap-etl-pipelines-from-the-sources-to-redshift)\
	3.1	[Set up the S3-to-Redshift pipeline](https://github.com/jobinthompu/etleap-redshift-dev-days#31-set-up-the-s3-to-redshift-pipeline)\
	3.2 [Set up the MySQL-to-Redshift pipeline](https://github.com/jobinthompu/etleap-redshift-dev-days#32-set-up-the-mysql-to-redshift-pipeline)
4. [Track ETL progress](https://github.com/jobinthompu/etleap-redshift-dev-days#4-track-etl-progress)
5. [Run queries on Redshift](https://github.com/jobinthompu/etleap-redshift-dev-days#5-run-queries-on-redshift)
6. [Create an Etleap Model](https://github.com/jobinthompu/etleap-redshift-dev-days#6-create-an-etleap-model)
7. [Run queries against this model](https://github.com/jobinthompu/etleap-redshift-dev-days#7-run-queries-against-this-model)
8. [Create Etleap ETL pipelines from the sources to S3 and Glue Catalog](https://github.com/jobinthompu/etleap-redshift-dev-days#8-create-etleap-etl-pipelines-from-the-sources-to-s3-and-glue-catalog)\
	8.1 [Set up the S3 Data Lake connection](https://github.com/jobinthompu/etleap-redshift-dev-days#81-set-up-the-s3-data-lake-connection)\
	8.2 [Set up the S3-to-S3/Glue pipeline](https://github.com/jobinthompu/etleap-redshift-dev-days#82-set-up-the-s3-to-s3glue-pipeline)\
    8.3 [Connect Redshift to Glue Catalog](https://github.com/jobinthompu/etleap-redshift-dev-days#83-connect-redshift-to-glue-catalog)
9. [Delete the AWS resources](https://github.com/jobinthompu/etleap-redshift-dev-days#9-delete-the-aws-resources)\
[Conclusion](https://github.com/jobinthompu/etleap-redshift-dev-days#10-conclusion)

## Introduction

Here is the scenario we'll be working through in this workshop. You are a data engineer at DoggySwag, an online retailer of dog products. Your analytics team wants to understand how your customers navigate your website in order to find and buy products, in order to optimize the user experience and increase spending.

User behavior data is available through events that have been captured with a web analytics tool, whereas the customer spend data is stored in your own SQL database. The web analytics tool has a reporting layer that provides user behavior insight, but this doesn't include data about customer spending. Your analysts can run SQL queries against the customer spend data, but can't correlate this with their online behavior data.

In this workshop you'll learn how to create a Redshift data warehouse that centralizes data from both these data sources. This way your analysts can run queries across the data sets.


## Prerequisites

- You must have an AWS account and an [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) user with sufficient permissions to interact with the AWS Management Console and creating various resources. 
- Your IAM permissions must also include access to create IAM roles and policies created by the AWS CloudFormation template.


## 1. Set up an AWS VPC with Redshift, Glue, and S3

In this section we'll set up a new VPC with the following resources:

- A Redshift cluster along with the security group rules necessary for accessing it from Etleap. 
- An S3 bucket to host our data lake.
- A Glue Catalog database to store data lake metadata.
- An IAM role that will be assigned to the Redshift cluster and used to access the data lake.
- An IAM user that will be used by Etleap to access the data lake.

Log into your AWS account and Deploy the CloudFormation Stack by Clicking the button Below:
[![Foo](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/LaunchStack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateUrl=https://cloudformation-template-repo.s3.amazonaws.com/ETLeapWorkshop.json&stackName=EtleapRedshiftDevDayStack). 

- Make sure the AWS region selected is N. Virginia (us-east-1).
- Specify a root password of your choice for your Redshift cluster. This must consist of at least 8 alphanumeric characters only, and must contain a lower-case letter, an upper-case letter, and a number.
- The other fields have sensible defaults, but feel free to modify as you see fit.
- You may update your email address if you wish to receive SNS notifications when stack creation is completed.
- Review the details on the screen, Check the box for 'I acknowledge that AWS CloudFormation might create IAM resources' and then choose 'Create' to start building the resources.
- Click 'Create stack' and wait for the creation to complete.

   ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/CreateCFStack.png)


## 2. Connect Etleap to Redshift and the data sources

We'll ETL data from two different data sources into your Redshift data warehouse. One source, an S3 bucket, contains JSON-formatted click event data. The other, a MySQL database, contains information about users. 

The first thing you'll need to do is log into Etleap using the credentials that were provided to you via email. The email has the subject line 'Your Etleap Account'.

In the rest of this section we'll connect Etleap to the data sources and Redshift destination, so that we can create pipelines in the next section.


### 2.1. Set up the S3 Input connection

Set up the S3 Input connection [here](https://app.etleap.com/#/connections/new/S3_INPUT). Use the following values:

| Property	        	| Value         								   | 
| ----------------------|:------------------------------------------------:| 
| Name      			|`Website Events`								   | 
| IAM Role:     		|`arn:aws:iam::525618399791:role/amazonjobin`	   |  
| Data Bucket 			|`etleap-redshift-devdays` 					   	   | 
| Base Directory		|`events`	      								   | 
| Additional properties |Leave as defaults     							   | 

Click 'Create Connection'.

   ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/S3Connections.png)


### 2.2. Set up the MySQL connection

Set up the MySQL connection [here](https://app.etleap.com/#/connections/new/MYSQL). Use the following values:

| Property	        	| Value         									| 
| ----------------------|:-------------------------------------------------:| 
| Name      			|`Webstore`											| 
| Connection Method     |`Direct`     										|  
| Address 				|`MySQLInstanceId` from your CloudFormation output	| 
| Port			 		|`3306`	      										| 
| Username			 	|`etleapuser`      									| 
| Password			 	|`Password123`										| 
| Database 				|`mv_webstore`      								| 
| Additional properties |Leave as defaults     								| 

Click 'Create Connection'

   ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/MySQLConnections.png)

### 2.3. Set up the Redshift connection

For this setup you'll need the values from your CloudFormation stack. These are available on the **Outputs** tab in the [Stack Info page](https://console.aws.amazon.com/cloudformation/home?region=us-east-1). 

   ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/CloudFormationOutput.png)

Set up the Redshift connection [here](https://app.etleap.com/#/connections/new/REDSHIFT). 

| Property	        	| Value         													| 
| ----------------------|:-----------------------------------------------------------------:| 
| Name      			|`Amazon Redshift`													| 
| Connection Method     |`Direct`     														|  
| Address 				|`RedshiftClusterHostnameOutput` from your CloudFormation output	| 
| Port			 		|`5439`	      														| 
| Username			 	|`root`      														| 
| Password			 	|`RedshiftClusterPasswordOutput` from your CloudFormation output	| 
| Database 				|`warehouse`      													| 
| Schema 				|`public`      														| 
| Additional properties |Leave as defaults     												| 

Click 'Create Connection'.

   ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/RedshiftConnection.png)

## 3. Create Etleap ETL pipelines from the sources to Redshift

In this section we'll configure pipelines that will ETL data from the sources into the Redshift database.

### 3.1. Set up the S3-to-Redshift pipeline

- Click the 'Create' button in the top nav-bar in Etleap.
- Pick 'Website Events' as the source.
- This page lists the files and folders available in S3. Click the radio button in the top-left to select the top-level directory.
- Click 'Wrangle Data'.
	![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/s3Pipeline.png)

- Wrangle the data. 
  - The JSON object should already be parsed by the wrangler, and each key is a column.
  - The `timestamp` column needs to be loaded as a `datetime` object in the warehouse. Click on the column header. On the right hand side, the Wrangler will suggest "Interpret `timestamp` as date and time...". Select the option, then click "Add".
  	![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/editS3Pipeline01.png)
  - Select the `ip` column header, and the Wrangler will suggest the "Look up geolocation..." transform. Select it, and click "Add".
  	![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/editS3Pipeline02.png)

- Click 'Next'.
- Pick 'Amazon Redshift' as the destination.
- Specify the following destination values:
  - Table name: `Website_Events`
  - Pipeline name: `Website Events`
- Click 'Next'.
- Click 'Start ETLing'.
  	![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/startS3Pipeline.png)


### 3.2. Set up the MySQL-to-Redshift pipeline

- Click the 'Create' button in the top nav-bar in Etleap.
- Pick 'Webstore' as the source.
- Select all the tables.
- Click 'Next'.
  	![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/mysqlPipeline.png)
- Leave the settings as they are
- Pick 'Amazon Redshift' as the destination.
- Leave all the options as their defaults in this step and click 'Next'.
- Click 'Start ETLing'.
  	![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/StartMysqlPipeline.png)

## 4. Track ETL progress

Etleap is now ETL'ing the data from the sources to Redshift. This will take 5-10 minutes. You can monitor progress [here](https://app.etleap.com/#/activities). Once you see events saying 'Website Events loaded successfully' and 'purchases loaded successfully' you can proceed to the next section.
  ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/ETLprogress.png)


## 5. Run queries on Redshift

Now that we have our data in Redshift we'll run a query that uses both datasets: we'll figure out how many clicks users have on average on our site segmented by whether or not they have made at least one purchase.	

For this setup you'll need the values from your CloudFormation stack. These are available on the **Outputs** tab in the [Stack Info page](https://console.aws.amazon.com/cloudformation/home?region=us-east-1).

- Go to the [Redshift query editor](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#query-editor:).
- Connect to your Redshift cluster in the 'Credentials' input:
  - Cluster: Pick the cluster that begins with 'etleapredshiftdevdaystack'.
  - Database: `warehouse`
  - Database user: `root`
  - Database password: Use the 'Value' of 'RedshiftClusterPasswordOutput' from your CloudFormation stack.
- Enter the following query:

```
WITH spend_per_user AS (
  SELECT u.external_id, SUM(i.price) AS spend
  FROM public.purchase p
    INNER JOIN public.line_item li ON li.purchase_id = p.id
    INNER JOIN public.item i ON i.item_id = li.item_id
    INNER JOIN public.user u ON p.user_id = u.id
  GROUP BY u.external_id
)
SELECT s.external_id, SUM(s.spend)/COUNT(e.external_id) AS spend_by_login
  FROM spend_per_user s
  INNER JOIN public.user u ON u.external_id = s.external_id
  INNER JOIN public.website_events e ON e.external_id = u.external_id
GROUP BY s.external_id
ORDER BY spend_by_login desc
LIMIT 10;
```
- Click 'Run'.

  ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/TableQuery.png)

The above query takes a while to return the expected results.
Let's see if we can improve on this with a Materialized View.

## 6. Create an Etleap Model

Now that all the data is in Redshift, let's create a model to speed up the runtime of the previous query.

 - Click the 'Create' button in the top nav-bar in Etleap.
 - Pick 'Amazon Redshift' as the source.
 - Select 'Model'
  ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/CreateModel01.png)
 - In the query editor enter the following query:

```
SELECT u.id AS user_id, COUNT(e.external_id) AS logins
FROM public.user u, public.Website_Events e
WHERE u.external_id = e.external_id
GROUP BY u.id;
```

 - Click `Validate & Next`
 - Select 'Materialized View' and click 'Next'
 - Name the table as `logins_by_user`, leave everything else as default and click 'Next'.
 - Finally, click 'Create Model' and you created a new materialized view in Etleap!

  ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/CreateModel02.png)

 The model model will take a few minutes to create.
 Once it is created, you can go on to next step.

## 7. Run queries against this model

Similar to section 5, once the model update is complete, run the following query:

```
WITH spend_per_user AS (
  SELECT p.user_id, SUM(i.price) AS spend
  FROM public.purchase p 
    INNER JOIN public.line_item li ON li.purchase_id = p.id
    INNER JOIN public.item i ON i.item_id = li.item_id
  GROUP BY p.user_id
)
SELECT s.user_id, SUM(s.spend)/SUM(l.logins) AS spend_by_login 
FROM spend_per_user s
  INNER JOIN public.logins_by_user l ON l.user_id = s.user_id
GROUP BY s.user_id
ORDER BY spend_by_login desc
LIMIT 10;
```
  ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/MVQuery.png)

As you can see, this time the query ran considerably faster the before.
This is the power of Materialized Views.


## 8. Create Etleap ETL pipelines from the sources to S3 and Glue Catalog

In this section we'll configure pipelines that will ETL data from the sources into your S3/Glue Catalog data lake.

### 8.1. Set up the S3 Data Lake connection 

Set up the S3 Data Lake connection [here](https://app.etleap.com/#/connections/new/S3_DATA_LAKE). Use the following values:

- Leave the name as `Amazon S3 Data Lake`
- For the Data Lake IAM Role, use the `DataLakeRoleARN` output from your CloudFormation stack. Make sure you remove any whitespace at the end of the input.
- For the bucket, use the `S3DataLakeBucket` output from your CloudFormation stack. Make sure you remove any whitespace at the end of the input.
- Leave the base directory as '/'.
- For the Glue database, use the `GlueCatalogDBName` output from your CloudFormation stack. Make sure you remove any whitespace at the end of the input.
- For the Glue catalog region, specify 'us-east-1'.
- Click 'Create Connection'. Click 'Ignore and Continue' for the warning about not having data in the input path.

 ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/CreateDataLake.png)

### 8.2. Set up the S3-to-S3/Glue pipeline

This is similar to the S3-to-Redshift pipeline, except this time the destination is S3/Glue.

- Click the 'Create' button in the top nav-bar in Etleap.
- Pick 'Website Events' as the source.
- This page lists the files and folders available in S3. Click the radio button in the top-left to select the top-level directory.
- Click 'Skip Wrangling'.
 ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/GluePipeline1.png)
- Select the script from the 'Website Events' pipeline and click 'Next'.
 ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/GluePipeline2.png)
- Select `Amazon S3 Data Lake` as the destination.
- Specify the following destination values:
  - Table name: `Website_Events`
  - Pipeline name: `Website Events - Lake`
- Click 'More destination options' and select 'Parquet' as the output format.
- Click 'Start ETLing'.
 ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/GluePipeline3.png)

### 8.3. Connect Redshift to Glue Catalog

Now for the fun part - we're going to use Redshift Spectrum to query data stored both in Redshift and S3 in the same query. First we need to hook up Redshift to Glue Catalog:

- Go to the [Redshift query editor](https://console.aws.amazon.com/redshiftv2/home?region=us-east-1#query-editor:).
- Replace the `GlueCatalogDBName` and `RedshiftSpectrumIAMRole` output from your CloudFormation stack in the query below and execute it.

```
CREATE external SCHEMA spectrumdb
FROM data catalog DATABASE '<GlueCatalogDBName>'
IAM_ROLE '<RedshiftSpectrumIAMRole>'
CREATE external DATABASE if not exists;
```
 ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/CreateSpectrum.png)

- Now we're ready to execute the query. Let's use the same query as before, but now using the data in S3 instead. The only difference in this query is the 'spectrumdb.' prefix in the from-clause in the clicks_per_user common table expression.

```
WITH spend_per_user AS (
  SELECT u.external_id, SUM(i.price) AS spend
  FROM public.purchase p 
    INNER JOIN public.line_item li ON li.purchase_id = p.id
    INNER JOIN public.item i ON i.item_id = li.item_id
    INNER JOIN public.user u ON p.User_id = u.id
  GROUP BY u.external_id
)
SELECT s.external_id, SUM(s.spend)/COUNT(e.external_id) AS spend_by_login
  FROM spend_per_user s
  INNER JOIN public.user u ON u.external_id = s.external_id
  INNER JOIN spectrumdb.Website_Events e ON e.external_id = u.external_id
GROUP BY s.external_id
ORDER BY spend_by_login desc
LIMIT 10;
```
 ![alt tag](https://github.com/jobinthompu/etleap-redshift-dev-days/blob/master/Resources/Images/SpectrumQueryResult.png)

## 9. Delete the AWS resources

- Go to [AWS CloudFormation console](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/).
- Make sure the AWS region selected is N. Virginia (us-east-1).
- Select the CloudFormation Stack you created in this workshop.
- Click Delete.
- Click “Delete Stack”.

This will take few minutes to delete all the resources. 

## Conclusion

As part of this workshop, you deployed an Amazon Redshift cluster and other resources using cloudformation, created S3, Redshift and Mysql connections in ETLeap, created multiple pipelines in ETLeap, created and ETLeap Materialized view model and ran queries against that model. You also did created a pipeline to setup s3 datalake connections to create Glue catalog tables and use them to run Redshift federated queries to join data stored in s3 as well as in Redshift.