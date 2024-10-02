In this lab, we will showcase how to leverage the newly released Continuous Queries feature in BigQuery to perform real-time sentiment analysis on hotel customer reviews. 
This enables hotels to respond promptly to negative feedback and continuously improve based on positive reviews.

The goal is to demonstrate how to implement a complete end-to-end solution with minimal coding, using a low-code/no-code approach.
Below is the architecture we will be implementing.


<img width="938" alt="image" src="https://github.com/user-attachments/assets/8891a9fa-a46c-4146-8045-738e751a0c81">

In this blog, we will work with Kaggle's Reviews Dataset and carry out the following:

> **1- Stream the Dataset to Pub/Sub:** We'll send the reviews data to Pub/Sub and configure the subscription's delivery type to "write to BigQuery." This setup enables Pub/Sub to stream the data directly into our target BigQuery table, eliminating the need for a traditional ETL pipeline.

> **2- Perform Real-Time Sentiment Analysis using Continuous Queries:** We'll leverage Continuous Queries in BigQuery to invoke a remote Vertex AI model through BQML for sentiment analysis on the incoming data.

> **3- Provide Actionable Insights in Real Time:** The sentiment analysis results will be written to both a BigQuery table and a separate Pub/Sub topic, allowing real-time consumption by operational applications and empowering end-users to take immediate action.



# Project Setup

## Prerequisites
#### Permissions
In this tutorial, we will utilize a service account to execute the continuous queries.
* The service account must have the necessary permissions to create a job. The roles that grant job creation permissions are BigQuery User, BigQuery Job User, and BigQuery Admin.
* To export data from a BigQuery table, the service account requires table export permission, which is available through the BQ Data Viewer, Data Editor, Data Owner, and BQ Admin roles.
* Lastly, to submit a job using the service account, the user must have the Service Account User role.

#### Project Whitelisting   
During the preview phase, you need to submit this request form to have your project whitelisted for Continuous Queries.
As of this writing, Continuous Queries is in preview.  Submit a [request form](https://docs.google.com/forms/d/e/1FAIpQLSc-SL89C9K997jSm_u3oQH-UGGe3brzsybbX6mf5VFaA0a4iA/viewform).


##### Vertex AI 
1. Activate Vertex AI : Make sure the Vertex AI API is enabled in your Google Cloud project.
2. Grant the "Vertex AI User" role to this service account. To get the service account id, locate the external connection you set up earlier in the "External Connections" section in your BigQuery settings.

<img width="810" alt="image" src="https://github.com/user-attachments/assets/db29271a-1775-41f3-9af1-4272729fdce4">


##### BigQuery
> **1. Create a dataset and a table** 
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

> **2. Create a remote model for Gemini 1.0 Pro in BigQuery ML**
```sql
CREATE OR REPLACE MODEL  `realtimeai.MarketingDW.sentimentanalysis`
REMOTE WITH CONNECTION `us-east1.realtimeaiConn`
OPTIONS (endpoint = 'gemini-pro-vision')
```


> **3. Create a Reservation:**
Continuous Queries are only supported in certain BigQuery editions. Make sure you create the reservation in the same region as your BQ dataset.


To create a reservation in BQ, select Capacity Management under Administration

<img width="207" alt="image" src="https://github.com/user-attachments/assets/450aef10-4f12-4e77-be43-48f4d410a8f2">


Click create a reservation.

<img width="457" alt="image" src="https://github.com/user-attachments/assets/04de0088-9f4f-40da-bfd6-2c4dc132cf84">

Select the size that matches your workload. Ensure that the baseline slot count is equal to the max slots to disable auto-scaling as it is not supported.

<img width="253" alt="image" src="https://github.com/user-attachments/assets/97428ae1-2b8e-45d5-8e9f-3a7b5a62232b">


Click on the reservation you just created and select assignments. Create an assignment, specify your project and select  "Continuous" as Job type.

<img width="543" alt="image" src="https://github.com/user-attachments/assets/f6175ee7-6210-4f0e-8324-5d35672ac08a">


<img width="646" alt="image" src="https://github.com/user-attachments/assets/2c19d4af-ccf7-4028-acc5-1a63e6c0f922">


### Pub/Sub
Go to Pub/Sub and create a topic. While creating a topic, you can check the create a default subscription box.
<img width="379" alt="image" src="https://github.com/user-attachments/assets/60d992dc-bfbe-49d3-b2ca-0f98f9696a73">


Now that the setup is complete, we are going to start the data into pubsub.

### Streaming data to Pub/Sub

We'll use Kaggle's reviews dataset https://www.kaggle.com/datasets/ahmedabdulhamid/reviews-dataset. 
There are multiple ways to stream data from a bigQuery table to PubSub. For the purpose of the demo and as we are talking about Continuous Queries, we'll use that feature to push the data in Pub/Sub.
After you download the dataset, you can upload it to a Cloud Storage Account.
We imported the data into a BigQuery table called HotelReviewsRaw.


> **1. Enable the Continuous Query mode**
Before executing the below query and send the messages to Pub/Sub, you will need to enable COntinuos Query. To do so, in BigQuery Studio, under the "More" option, select Continuous Query.

<img width="331" alt="image" src="https://github.com/user-attachments/assets/a3c4375f-9c5b-4d75-8f8c-56e4da8563fa">

run the below query 
```sql
EXPORT DATA
  OPTIONS (
    format = 'CLOUD_PUBSUB',
    uri = 'https://pubsub.googleapis.com/projects/iba-demos-prj/topics/ibarealtimedataingestion')
AS (
  SELECT
 TO_JSON_STRING(
      STRUCT(
        address ,
        categories ,
        city ,
        country ,
        latitude ,
        longitude ,
        name ,
        postalCode ,
        province ,
        reviews_date ,
        reviews_dateAdded ,
        reviews_doRecommend ,
        reviews_id ,
        reviews_rating ,
        reviews_text ,
        reviews_title ,
        reviews_userCity ,
        reviews_username ,
        reviews_userProvince 
    
  )) AS message
FROM
  `iba-demos-prj.ContinuousQueries.HotelReviewsRaw` 

);
```


To run this query, you will need to specify a service account. To do, go in the query settings under more

<img width="324" alt="image" src="https://github.com/user-attachments/assets/2c2b350b-786f-4d48-949b-b725c488738b">

Enter a service account that has at least BQ Editor role.
<img width="316" alt="image" src="https://github.com/user-attachments/assets/eb7546b1-59bd-48b0-82ec-100f15caa2a3">

As the query is running and sending the data in Pub/Sub, you can check that Pub/Sub is automatically streming the data in BigQuery. Note that no ETL job was implemented to load the data in BQ.

### Applying Gen AI on data as it arrives in the BQ table

Now, we can start the query for for the sentiment analysis and leverage Gemini model and our Gen Ai capability.
Execute the below query in Continuous Query mode to output the analytics in a BQ Query table. Not that you could send this result to a Pub/Sub, Bigtable or AlloyDb for operational systems to take actions based on the insights inreal time.

```sql

insert into `iba-demos-prj.ContinuousQueries.HotelReviewsInsights`
SELECT
    address ,
    categories ,
    city ,
    country ,
    latitude ,
    longitude ,
    name ,
    postalCode ,
    province ,
    reviews_date ,
    reviews_dateAdded ,
    reviews_doRecommend ,
    reviews_id ,
    reviews_rating ,
    reviews_text ,
    reviews_title ,
    reviews_userCity ,
    reviews_username ,
    reviews_userProvince ,
    prompt,
    ml_generate_text_llm_result
    
FROM
    ML.GENERATE_TEXT(
      MODEL `iba-demos-prj.ContinuousQueries.sentimentanalysis`,
      (
        SELECT
             address ,
            categories ,
            city ,
            country ,
            latitude ,
            longitude ,
            name ,
            postalCode ,
            province ,
            reviews_date ,
            reviews_dateAdded ,
            reviews_doRecommend ,
            reviews_id ,
            reviews_rating ,
            reviews_text ,
            reviews_title ,
            reviews_userCity ,
            reviews_username ,
            reviews_userProvince  ,
              CONCAT(
                'for each of the following reviews, say if it a positive, negative or neutral ', reviews_text

              ) AS prompt
        FROM   `iba-demos-prj.ContinuousQueries.HotelReviewsCQ` 
          where city ='Boston'

      ),
      STRUCT(
        50 AS max_output_tokens,
        1.0 AS temperature,
        40 AS top_k,
        1.0 AS top_p,
        TRUE AS flatten_json_output))
      AS ml_output
```



### Service Account Setup:
You'll need a service account to run the job. Follow this guide to choose the appropriate account type.




