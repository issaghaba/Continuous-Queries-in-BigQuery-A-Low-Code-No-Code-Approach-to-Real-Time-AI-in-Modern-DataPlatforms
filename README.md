In this lab, we will showcase how to leverage the newly released Continuous Queries feature in BigQuery to perform real-time sentiment analysis on hotel customer reviews. 
This enables hotels to respond promptly to negative feedback and continuously improve based on positive reviews.

The goal is to demonstrate how to implement a complete end-to-end solution with minimal coding, using a low-code/no-code approach.
Below is the architecture we will be implementing.


<img width="938" alt="image" src="https://github.com/user-attachments/assets/8891a9fa-a46c-4146-8045-738e751a0c81">

In this blog, we will work with Kaggle's Reviews Dataset and carry out the following:

> **1- Stream the Dataset to Pub/Sub:** We'll send the reviews data to Pub/Sub and configure the subscription's delivery type to "write to BigQuery." This setup enables Pub/Sub to stream the data directly into our target BigQuery table, eliminating the need for a traditional ETL pipeline.

> **2- Perform Real-Time Sentiment Analysis Using Continuous Queries:** We'll leverage Continuous Queries in BigQuery to invoke a remote Vertex AI model through BQML for sentiment analysis on the incoming data.

> **3- Provide Actionable Insights in Real Time:** The sentiment analysis results will be written to both a BigQuery table and a separate Pub/Sub topic, allowing real-time consumption by operational applications and empowering end-users to take immediate action.



# Project Setup

## Prerequisites
### Vertex AI 
 - Enable the Vertex AI API
 - Grant the Vertex AI user role to the external connection service accountid. To find the service account Id, click on the external connection you created previously, under external connections
 <img width="810" alt="image" src="https://github.com/user-attachments/assets/62d23156-b7bb-40dd-b2b0-1ad40ffebac0">

### Project Whitelisting:    
During the preview phase, you need to submit this request form to have your project whitelisted for Continuous Queries.
As of this writing, Continuous Queries is in preview.  Submit a request form https://docs.google.com/forms/d/e/1FAIpQLSc-SL89C9K997jSm_u3oQH-UGGe3brzsybbX6mf5VFaA0a4iA/viewform to have your project whitelisted for access.

### Create a BigQuery Dataset and Table:
Set up a BigQuery dataset and table to store the data for real-time analysis.
- To create a dataset, expand the <img width="14" alt="image" src="https://github.com/user-attachments/assets/fac9b262-bf33-4a2c-964b-51424b07f712">
 and select create dataset.

<img width="432" alt="image" src="https://github.com/user-attachments/assets/66f3153e-4793-46fd-8d7f-6986d41a5929">


Enter the dataset id, select your region and hit the create dataset button.

<img width="350" alt="image" src="https://github.com/user-attachments/assets/1f701179-0663-49ca-8333-fe8ae702ee1e">

- use the below script to create the target table
```sql
CREATE TABLE `<your project id>.<your dataset>.HotelReviewsCQ`
(
  id STRING,
  dateAdded TIMESTAMP,
  dateUpdated TIMESTAMP,
  address STRING,
  categories STRING,
  primaryCategories STRING,
  city STRING,
  country STRING,
  keys STRING,
  latitude FLOAT64,
  longitude FLOAT64,
  name STRING,
  postalCode STRING,
  province STRING,
  reviews_date STRING,
  reviews_dateSeen STRING,
  reviews_rating FLOAT64,
  reviews_sourceURLs STRING,
  reviews_text STRING,
  reviews_title STRING,
  reviews_userCity STRING,
  reviews_userProvince STRING,
  reviews_username STRING,
  sourceURLs STRING,
  websites STRING
);

```

### Streaming data to Pub/Sub
#### Create a Pub/Sub topic and subscription

We'll use Kaggle's reviews dataset https://www.kaggle.com/datasets/ahmedabdulhamid/reviews-dataset. 
There are multiple ways to stream data from a bigQuery table to PubSub. For the purpose of the demo and as we are talking about Continuous Queries, we'll use that feature to push the data in Pub/Sub
After you download the dataset, you can upload it to a Cloud Storage Account.
Using the console, you can create a BigQuery table by importing the csv file.
To do so,  expand the <img width="14" alt="image" src="https://github.com/user-attachments/assets/fac9b262-bf33-4a2c-964b-51424b07f712">
 and select create table.
 <img width="300" alt="image" src="https://github.com/user-attachments/assets/6ffbb922-e2af-4d82-b5f4-14900f51a126">

File out the form and click the create table button.

<img width="244" alt="image" src="https://github.com/user-attachments/assets/1097b9ab-c278-4ef7-9611-f2cb10de1c5a">





### Create a Reservation:
Continuous Queries are only supported in certain BigQuery editions. You'll need to create a reservation with auto scaling disabled. To disable auto-scaling, ensure that the baseline slot count is equal to the max slots to disable auto-scaling.
To create a reservation, in BQ`

### Service Account Setup:
You'll need a service account to run the job. Follow this guide to choose the appropriate account type.

### Create a remote model for Gemini 1.0 Pro in BigQuery ML
```SQL
CREATE OR REPLACE MODEL  `realtimeai.MarketingDW.sentimentanalysis`
REMOTE WITH CONNECTION `us-east1.realtimeaiConn`
OPTIONS (endpoint = 'gemini-pro-vision') ```



