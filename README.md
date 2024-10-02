In this lab, we will showcase how to leverage the newly released Continuous Queries feature in BigQuery to perform real-time sentiment analysis on hotel customer reviews. 
This enables hotels to respond promptly to negative feedback and continuously improve based on positive reviews.

The goal is to demonstrate how to implement a complete end-to-end solution with minimal coding, using a low-code/no-code approach.
Below is the architecture we will be implementing.


<img width="938" alt="image" src="https://github.com/user-attachments/assets/8891a9fa-a46c-4146-8045-738e751a0c81">

In this blog, we will work with Kaggle's Reviews Dataset and carry out the following:

> **1- Stream the Dataset to Pub/Sub:** We'll send the reviews data to Pub/Sub and configure the subscription's delivery type to "write to BigQuery." This setup enables Pub/Sub to stream the data directly into our target BigQuery table, eliminating the need for a traditional ETL pipeline.

> **2- Perform Real-Time Sentiment Analysis using Continuous Queries:** We'll leverage Continuous Queries in BigQuery to invoke a remote Vertex AI model through BQML for sentiment analysis on the incoming data.

> **3- Provide Actionable Insights in Real Time:** The sentiment analysis results will be written to both a BigQuery table and a separate Pub/Sub topic, allowing real-time consumption by operational applications and empowering end-users to take immediate action.



## Project Setup

### **Permissions**
In this tutorial, we will utilize a service account to execute the continuous queries.
* The service account must have the necessary permissions to create a job. The roles that grant job creation permissions are BigQuery User, BigQuery Job User, and BigQuery Admin.
* To export data from a BigQuery table, the service account requires table export permission, which is available through the BQ Data Viewer, Data Editor, Data Owner, and BQ Admin roles.
* Lastly, to submit a job using the service account, the user must have the Service Account User role.

### **Project Whitelisting**   
During the preview phase, you need to submit this request form to have your project whitelisted for Continuous Queries.
As of this writing, Continuous Queries is in preview.  Submit a [request form](https://docs.google.com/forms/d/e/1FAIpQLSc-SL89C9K997jSm_u3oQH-UGGe3brzsybbX6mf5VFaA0a4iA/viewform).


### **Vertex AI** 
**1. Activate Vertex AI:** Make sure the Vertex AI API is enabled in your Google Cloud project.<br/>
**2. Create an external connection:** The external connection will be use to create the remote Vertex Ai model. 
<br/>To create an external connection, in BigQuery Studio, next to the Explorer panel, click on Add.

<img width="294" alt="image" src="https://github.com/user-attachments/assets/8a8090ed-5551-4a56-8f2f-59e5dc96d714">

<br/>In the new window, select Connection to external data sources as the source.

<img width="967" alt="image" src="https://github.com/user-attachments/assets/516cd433-cb72-4b61-bfd2-a3650c1411df">

<br/>In the configuration window, choose Vertex AI remote models, remote functions, and BigLake (Cloud Resource).
Enter a connection ID and select the location, ensuring it matches the location of your BigQuery dataset.
Finally, click on the Create Connection button.

<img width="309" alt="image" src="https://github.com/user-attachments/assets/fa00aca0-eef6-441d-879b-1761220d2a4c">


**3. Grant the "Vertex AI User" role to this service account:** To get the service account id, locate the external connection you set up earlier in the "External Connections" section in your BigQuery settings.

   <img width="788" alt="image" src="https://github.com/user-attachments/assets/bd8541f7-2651-4a9b-a053-49ac8d38ad85">
   
**4. Create a Vertex AI remote model with the Gemini 1.0 Pro endpoint in BigQuery ML**

```sql

CREATE OR REPLACE MODEL  `<your project id>.<your dataset>.sentimentanalysis`
REMOTE WITH CONNECTION `us-central1.gemini-1-0-pro`
OPTIONS (endpoint ='gemini-1.0-pro') 

```
### **BigQuery**
**1. Create a dataset and a table:** To create a BigQuery dataset and table for real-time analysis, begin by expanding the <img width="14" alt="image" src="https://github.com/user-attachments/assets/fac9b262-bf33-4a2c-964b-51424b07f712">.
Click on Create Dataset from the options to start configuring your new dataset.

<img width="432" alt="image" src="https://github.com/user-attachments/assets/2cc77de1-9551-4087-be3c-5d050c9b0782">


Enter the dataset id, select your region and hit the create dataset button.

<img width="310" alt="image" src="https://github.com/user-attachments/assets/7d77ec7d-90f4-477f-9a00-ef5506ec99e3">


<br/> Use the below script to create the target table<br/>

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

**3. Create a Reservation:**
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


### **Pub/Sub**
Navigate to Pub/Sub and create a new topic. During the topic creation process, you can select the option to create a default subscription.<br/>
<img width="379" alt="image" src="https://github.com/user-attachments/assets/60d992dc-bfbe-49d3-b2ca-0f98f9696a73">

With the setup complete, we will now begin streaming the data into Pub/Sub


### Streaming data to Pub/Sub

We'll be using Kaggle's reviews dataset, available at this [here](https://www.kaggle.com/datasets/ahmedabdulhamid/reviews-dataset).

There are several methods to stream data from a BigQuery table to Pub/Sub. For this demo, and since we're focusing on Continuous Queries, we'll leverage that feature to push the data into Pub/Sub from BQ.
<br/>After downloading the dataset, you can upload it to a Cloud Storage bucket. We have imported the dataset into a BigQuery table named HotelReviewsRaw.

**1. Enable the Continuous Query mode**
Before executing the query to send messages to Pub/Sub, you'll need to enable Continuous Query. To do this, go to BigQuery Studio, click on the "More" option, and select Continuous Query. <br/>
<img width="331" alt="image" src="https://github.com/user-attachments/assets/a3c4375f-9c5b-4d75-8f8c-56e4da8563fa">

<br/>Execute the below query in BigQuery studio.<br/>

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
  `<your project>.<your dataset>.HotelReviewsRaw` 

);
```

**Additional step:** Before running the query, ensure you specify the appropriate service account with the necessary permissions. You can do this in the Query Settings under the More options. Select the relevant service account to allow BigQuery to interact with Pub/Sub.

<img width="324" alt="image" src="https://github.com/user-attachments/assets/2c2b350b-786f-4d48-949b-b725c488738b">

Enter a service account that has at least BQ Editor role.
<img width="316" alt="image" src="https://github.com/user-attachments/assets/eb7546b1-59bd-48b0-82ec-100f15caa2a3">

As the query runs and sends data to Pub/Sub, you can observe that Pub/Sub is automatically streaming the data into BigQuery. It's important to highlight that no ETL job was required to load the data into BigQuery.

### Text Generation Inference on data as it streams into the BigQuery table

We can now initiate the sentiment analysis query, using the Gemini model created earlier. Execute the following query in Continuous Query mode to output the analytics into a BigQuery table. Keep in mind that the results can also be streamed to Pub/Sub or Bigtable, enabling real-time consumption by operational applications and allowing end-users to take immediate action

```sql

insert into `<your project id>.<your dataset>.HotelReviewsInsights`
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
      MODEL `<your project id>.<your dataset>.sentimentanalysis`,
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
        FROM   `<your project id>.<your dataset>.HotelReviewsCQ` 

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




