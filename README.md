In this lab, we will showcase how to leverage the newly released Continuous Queries feature in BigQuery to perform real-time sentiment analysis on hotel customer reviews. 
This enables hotels to respond promptly to negative feedback and continuously improve based on positive reviews.

The goal is to demonstrate how to implement a complete end-to-end solution with minimal coding, using a low-code/no-code approach.
Below is the architecture we will be implementing.


<img width="938" alt="image" src="https://github.com/user-attachments/assets/8891a9fa-a46c-4146-8045-738e751a0c81">


# Project Setup

## Prerequisites
### Project Whitelisting:    
During the preview phase, you need to submit this request form to have your project whitelisted for Continuous Queries.

### Create a Reservation:
Continuous Queries are only supported in certain BigQuery editions. You'll need to create a reservation with the following configuration:

### No Auto-Scaling: 
Ensure that the baseline slot count is equal to the max slots to disable auto-scaling.
### Service Account Setup:
You'll need a service account to run the job. Follow this guide to choose the appropriate account type.

### Create a BigQuery Dataset and Table:
Set up a BigQuery dataset and table to store the data for real-time analysis.



