# Welcome to the Dataflow Streaming Tutorial

## Intro

In this tutorial we'll create the following scenario:

<!-- ERRORS
https://stackoverflow.com/questions/64577178/how-to-specify-image-path-in-cloud-shell-when-writing-gcp-walkthrough-tutorial
 -->
<!--![image info](/home/vinicius_higa/cloudshell_open/runbook-gcp/images/streaming-pubsub-bq.png) -->
1. Generate JSON Files through a schema with fake info
2. Populate a PubSub Topic
3. Stream Data from PubSub
4. Transform the JSON file into rows
5. Populate a Big Query Table

## Set the GCP Project

Pick up your project

<walkthrough-project-setup></walkthrough-project-setup>

## Pre-reqs

To be able to run the next steps, you're gonna need a valid GCP project with the following configs:

 - PubSub API Enabled
 - BigQuery API Enabled
 - Dataflow API  Enabled
 - At least permission to create/write:
    - Topic
    - Subscription
    - BQ table/dataset
    - Dataflow jobs 
    - GCS Bucket


### Turn on Google Cloud APIs (beta)

<walkthrough-enable-apis apis=
  "compute.googleapis.com,dataflow,cloudresourcemanager.googleapis.com,logging,storage_component,storage_api,bigquery,pubsub">
</walkthrough-enable-apis>

For further details, please check the GCP documentation according to the component.

## Visiting google Console

[ ! ] *This tutorial does not work well if you're using Cloud Shell in a separate window.*

Please click [here](https://console.cloud.google.com/home/dashboard?project={{project-id}}&cloudshell=true) for a better experience

After opening the GCP console page, copy/past the following command in the cloudshell:
```bash
cloudshell launch-tutorial -d dataflow/streaming-pubsub-bq.md
```

## PubSub Topic/Subscription

### Open Google Cloud Shell

Open Cloud Shell by clicking
<walkthrough-cloud-shell-icon></walkthrough-cloud-shell-icon>
[icon][spotlight-open-devshell] in the navigation bar at the top of the console.


### Create PubSub Topic
```bash
gcloud pubsub topics create streaming.${USER/_/.}.pubsub.bq 
```


### Create PubSub Subscription
```bash
gcloud pubsub subscriptions \
    create streaming.${USER/_/.}.pubsub.bq-sub --topic streaming.${USER/_/.}.pubsub.bq \
    --ack-deadline=60 \
    --retain-acked-messages
```


### Navigate to the Pub/Sub section

Open the [menu][spotlight-console-menu] on the left side of the console.

Then, select the **Pub/Sub** section. Now check if the previous commands worked.

<walkthrough-menu-navigation sectionId="CLOUDPUBSUB_SECTION"></walkthrough-menu-navigation>

[ ! ] **Quick Tip**: On left side, there is a panel with a few options. Click on "Topics" afterwards "Subscriptions".


## Big Query Dataset/Table


### Open Google Cloud Shell

Open Cloud Shell by clicking
<walkthrough-cloud-shell-icon></walkthrough-cloud-shell-icon>
[icon][spotlight-open-devshell] in the navigation bar at the top of the console.


### Create Dataset
```bash
bq --location=southamerica-east1 mk \
--dataset \
--description "Tutorial Dataset" \
{{project-id}}:$USER
```

### Share Dataset with Dataflow Account

```bash
PROJECT_NUMBER=$(gcloud projects describe pix-analytics-bv-des | grep -i projectNumber | awk -F' ' {'print $2'} | tr -d "'")
DATAFLOW_ACCOUNT=$PROJECT_NUMBER'-compute@developer.gserviceaccount.com'

bq show \
--format=prettyjson \
 {{project-id}}:$USER > policy.json

jq '.access += [ {"role": "WRITER", "userByEmail": "'${DATAFLOW_ACCOUNT}'" } ]' policy.json > policy_updated.json

bq update --source=policy_updated.json \
{{project-id}}:$USER
```


### Create Table
```bash
bq mk \
--table \
--description "Streaming table" \
{{project-id}}:$USER.streaming_pubsub_bq \
schema/bq-table-ddl.json
```

### Navigate to the BQ section

Open the [menu][spotlight-console-menu] on the left side of the console.

Then, select the **BigQuery** section. Now check if the previous commands worked.

<walkthrough-menu-navigation sectionId="BIGQUERY_SECTION"></walkthrough-menu-navigation>

[ ! ] **Quick Tip**: Find out the "Explorer" in your left, expand the nodes until you find the dataset/table. Click on the table to get more details.

## Dataflow Scenario


### Introduction

In the next steps, we'll reproduce 2 scenarios where in the first one, PubSub receives a bunch of JSON files through a Dataflow Job (template: Streaming Data Generator). 
For the last one, after publishing these mocked data on Pubsub, we'll gather the data and transform into a Big Query table, by also calling a Dataflow Job (template: Pub/Sub Subscription to BigQuery).

[ ! ] **Disclaimer**: This tutorial is not intended to show how-to deduplicate streaming incoming data.


### Open Google Cloud Shell

Open Cloud Shell by clicking
<walkthrough-cloud-shell-icon></walkthrough-cloud-shell-icon>
[icon][spotlight-open-devshell] in the navigation bar at the top of the console.


### Create a GCS Bucket

```bash
gsutil mb gs://tutorials_$USER
```


### JSON Schema

Since we need to populate PubSub, we will use the following schema to 'fake' our data:

```json
{
	"id" : "83dasdsa",
	"first_name" : "Vinicius",
	"last_name" : "Murakami",
	"family" : [ "Mulan" , "Mortadela"],	
	"contact" : {
				"email" : "vinicius.murakami@gmail.com",
				"ssn" : "noneofyourbusiness",
				"phone" : "+551100000000",
				"ipv4" : "0.0.0.0"
	},
	"birth_date" : "2021-05-25",
	"country" : "Brazil",
	"company" : "N/A"
}
```


### Uploading schema to GCS

To upload the provided schema (already filled with the necessary parameters) to your GCS bucket, execute the command below:

```bash
gsutil cp schema/runbook-streaming.json gs://tutorials_$USER/schema/
```

[ ! ] This JSON schema will be used in the next steps, it's extremely important to execute this step.


### Go to the Cloud Storage page

Open the [menu][spotlight-console-menu] on the left side of the console.

Then, select the **Storage** section.

<walkthrough-menu-navigation sectionId=STORAGE_SECTION></walkthrough-menu-navigation>

Now check if your file were uploaded. 

## 1st Scenario : Generate JSON Files to PubSub


### Sampling Data with a Dataflow *trick*

Now, finally we will call our first Dataflow Template: "Streaming Data Generator".  
This template is almost a swiss-knife where gives you a few options to play around with BQ, PubSub and etc.
The main objective as the name says, it generates data for streaming purpose, in our scenario we'll use a simple JSON  (already in your GCS bucket, if you followed the previous step) as an example to mock the data and send to PubSub.

Run the following command to bootstrap the template:

<!-- TODO MAKE IT DYNAMIC PROPERTIES -->

```bash
NETWORK_PROJECT=default #FILL_WITH_YOUR_NETWORK
SUBNET_PROJECT=default #FILL_WITH_YOUR_SUBNETWORK
PUBLIC_IP=false #FILL_WITH_YOUR_PUBLIC_IP_BOOL

gcloud beta dataflow flex-template run streaming-${USER/_/-}-data-gen \
--template-file-gcs-location gs://dataflow-templates-southamerica-east1/latest/flex/Streaming_Data_Generator \
--region southamerica-east1 \
--parameters \
schemaLocation=gs://tutorials_$USER/schema/runbook-streaming.json,topic=projects/{{project-id}}/topics/streaming.${USER/_/.}.pubsub.bq,qps=10000,messagesLimit=1000000,network=$NETWORK_PROJECT,subnetwork=$SUBNET_PROJECT,usePublicIps=$PUBLIC_IP
```

The command above consumes an already provided template from GCP [Github](https://github.com/GoogleCloudPlatform/DataflowTemplates).
If everything went well, you'll see a "Job summary" message.
Now the job it's gonna "mock" 1M of json files to your PubSub topic.

[ ! ] Feel free to change your REGION, since i'm running from Brazil, most likely we use southamerica-east1 as the preferred REGION.


### Check if your job is running

Open the [menu][spotlight-console-menu] on the left side of the console.

Then, select the **Dataflow** section.

<walkthrough-menu-navigation sectionId="DATAFLOW_SECTION"></walkthrough-menu-navigation>

Navigate to your job and find out if your pipeline is up and running.

## 2nd Scenario : Streaming PubSub into BigQuery

### Connecting PubSub to BQ

On this second part, we'll use a template called "Pub/Sub Subscription to BigQuery". 
This is a quite simple way to stream data from a message stream to BigQuery.

All we have to do, is to fill out the following parameters:

 - BQ table to stream the data (must exists)
 - A PubSub Subscription (hence a topic must exists)
 - Share BQ Dataset with the Dataflow Account

Run the following command to bootstrap the template:

<!-- TODO MAKE IT DYNAMIC PROPERTIES -->

```bash
gcloud dataflow jobs run streaming-${USER/_/-}-pubsub-bq \ 
--gcs-location gs://dataflow-templates-southamerica-east1/latest/PubSub_Subscription_to_BigQuery \
--region southamerica-east1 \
--staging-location gs://tutorials_$USER/temp \
--subnetwork $SUBNET_PROJECT \
--network $NETWORK_PROJECT \
--disable-public-ips \
--enable-streaming-engine \
--parameters inputSubscription=projects/{{project-id}}/subscriptions/streaming.${USER/_/.}.pubsub.bq-sub,outputTableSpec={{project-id}}:$USER.streaming_pubsub_bq,outputDeadletterTable={{project-id}}:$USER.streaming_pubsub_bq_dlt
```

The command above consumes an already provided template from GCP [Github](https://github.com/GoogleCloudPlatform/DataflowTemplates).
If everything went well, you'll see a "Job summary" message.
Now the job it's gonna pull out the subscription data to BQ table.

[ ! ] Feel free to change your REGION, since i'm running from Brazil, most likely we use southamerica-east1 as the preferred REGION.

[ ! ] If you face any errors on Dataflow, check if your dataflow account has permission to write to the big query table

### Check if your job is running

Open the [menu][spotlight-console-menu] on the left side of the console.

Then, select the **Dataflow** section.

<walkthrough-menu-navigation sectionId="DATAFLOW_SECTION"></walkthrough-menu-navigation>

Navigate to your job and find out if your pipeline is up and running.

## Check the Big Query Table


### Counting Streaming Data

We're almost finishing, get back to Cloud Shell and hit the following query. 

```bash
bq query --nouse_legacy_sql \
'SELECT COUNT(*) FROM {{project-id}}.'${USER}'.streaming_pubsub_bq'
```

See if the output makes sense for you.

[Note] - If your job didn't finish, replay this step until you see the total amount expected (1M of rows)

## はじめに

このチュートリアルでは、Java を使用して簡単なサンプル パイプラインを実行し、Cloud Dataflow サービスの基本を学習します。

Nah, just kidding. We'll keep it up in english ;P 

## THE END

Congrats! You made it! 
Finished the tutorial "Streaming PubSub to BQ"

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

[spotlight-console-menu]: walkthrough://spotlight-pointer?spotlightId=console-nav-menu
[spotlight-open-devshell]: walkthrough://spotlight-pointer?spotlightId=devshell-activate-button
[spotlight-buckets-link]: walkthrough://spotlight-pointer?cssSelector=.p6n-cloudstorage-path-link
[spotlight-job-logs]: walkthrough://spotlight-pointer?cssSelector=#p6n-dax-job-logs-toggle
