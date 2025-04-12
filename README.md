# Amazon Data Engineering Project
## 🛑 Problem Statement

Imagine an e-commerce business experiencing rapid growth. While exciting, this growth presents significant data challenges:

* **Manual Data Handling:** Analysts rely on manually exporting transaction data (like orders, customer info, product details) from the production MySQL database into CSV files. These files are then processed using tools like Excel.
* **Scalability Issues:** As order volumes and customer interactions increase, the manual export/import process becomes incredibly time-consuming and prone to human error. It simply doesn't scale.
* **Delayed Insights:** Reporting is often delayed (e.g., weekly or monthly) because of the inefficient process. This means the business can't react quickly to trends or issues.
* **Performance Impact:** Running complex analytical queries directly on the production database slows it down, potentially affecting the customer experience on the website or app.
* **Data Silos:** Data might be scattered across different spreadsheets or local machines, leading to inconsistent views of the business.
* **Limited Analytics:** The tools and processes limit the complexity of analysis that can be performed, hindering deeper insights into customer behavior, sales patterns, or inventory management.

Essentially, the business struggles to get timely, reliable, and comprehensive insights from its growing data assets, impacting decision-making and operational efficiency.
## 💡 Proposed Solution

To overcome the limitations of the manual process, this project implements a modern, automated data pipeline using a suite of AWS services. The core idea is to create a robust workflow that handles data from ingestion to visualization with minimal human intervention.

The key aspects of the solution are:

* **Automated Data Replication:** Implement AWS Database Migration Service (DMS) to continuously capture changes from the source MySQL database and load the raw data into an Amazon S3 data lake (Bronze layer).
* **Structured Data Lake:** Organize the data in S3 using the Medallion architecture (Bronze, Silver, Gold layers - though Gold is primarily in Redshift here) to ensure data quality and traceability.
* **Scalable Transformation:** Utilize AWS Glue ETL jobs to automatically clean, transform, and enrich the raw data (Bronze -> Silver) and prepare it for loading into the data warehouse (Silver -> Redshift/Gold).
* **Centralized Analytics Hub:** Employ Amazon Redshift as a cloud data warehouse to store the final, aggregated, analysis-ready data (Gold layer), optimized for complex querying.
* **End-to-End Orchestration:** Use AWS Step Functions to manage the dependencies and sequence of the pipeline tasks (e.g., run Glue jobs after DMS sync, load Redshift after Glue transformations).
* **Scheduled Execution:** Leverage Amazon EventBridge to automatically trigger the entire pipeline on a regular schedule (e.g., daily), ensuring fresh data for analysis.
* **Insightful Visualization:** Connect Amazon QuickSight to Redshift to build interactive dashboards, providing stakeholders with easy access to key business metrics and insights.

This automated, cloud-based approach replaces the slow, manual workflow with a scalable, reliable, and efficient system capable of providing timely data insights.

## Prerequisite Knowledge

### 🔧 AWS Components Used

This pipeline leverages several core AWS services, each playing a specific role:

| Service              | Role in Pipeline                                       |
| :------------------- | :----------------------------------------------------- |
| **AWS Lambda** | Simulates transactional inserts (synthetic data)* |
| **Secrets Manager** | Secures MySQL database credentials                     |
| **AWS DMS** | Migrates data continuously from MySQL to S3 (Bronze)   |
| **Amazon S3** | Provides layered storage: Bronze (raw), Silver (cleaned) |
| **AWS Glue Crawler** | Automatically discovers schema & catalogs data in S3 |
| **AWS Glue Jobs** | Cleans, transforms data (Bronze→Silver, Silver→Gold) |
| **Step Functions** | Orchestrates the sequence of tasks in the pipeline     |
| **Amazon EventBridge**| Schedules the pipeline execution (e.g., daily runs)    |
| **Amazon Redshift** | Stores the final, cleaned, aggregated Gold data       |
| **Amazon QuickSight**| Visualizes data from Redshift via interactive dashboards|
| **Amazon VPC** | Provides network isolation for resources like RDS & Redshift |


This breakdown shows the main tools used to build the automated workflow.

---
## ⚙️ Architecture Overview

This section details the structure and operational flow of the automated AWS data pipeline. The design uses the Medallion Architecture (Bronze, Silver, Gold stages) to progressively refine data for analysis.

![AWS Data Pipeline Architecture](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/e9e131fc21e07ed51f4f48bd008d100590e3031e/Architecture%20of%20the%20work.png)  
*End-to-end data flow showing how AWS services collaborate.*

**How the Pipeline Works Step-by-Step:**

This pipeline operates as an orchestrated workflow, transforming raw database transactions into actionable insights. Here’s the sequence:

1.  **Scheduling the Start:** The entire process is kicked off automatically by **Amazon EventBridge**, which triggers the pipeline based on a schedule (e.g., every night). EventBridge's job is simply to start the main workflow defined in AWS Step Functions.

2.  **Orchestrating the Flow:** **AWS Step Functions** acts as the "brain" or conductor of the pipeline. It receives the trigger from EventBridge and manages the execution sequence, dependencies, and error handling for all the subsequent steps described below.

3.  **Getting the Raw Data (Ingestion):**
    * The first major task managed by Step Functions is data ingestion using **AWS DMS (Database Migration Service)**.
    * DMS connects securely to the source **Amazon RDS MySQL** database (using credentials safely retrieved from **AWS Secrets Manager**).
    * DMS captures changes (new orders, updates, etc.) from the database transaction logs (or does a full load) and replicates this raw data *as-is* into the **Bronze layer** within an **Amazon S3** bucket.
    * *How this helps:* This reliably copies data without impacting the performance of the source database significantly and creates a raw backup in cheap S3 storage.

4.  **Understanding the Raw Data (Cataloging):**
    * Once new data lands in the Bronze S3 bucket, Step Functions triggers an **AWS Glue Crawler**.
    * The Crawler examines the files in the Bronze layer, automatically figures out their structure (schema), and updates the **AWS Glue Data Catalog**.
    * *How this helps:* This eliminates the need to manually define the structure of raw data, allowing downstream jobs to easily understand and query it.

5.  **Cleaning the Data (Bronze → Silver):**
    * With the catalog updated, Step Functions starts the first **AWS Glue ETL Job**.
    * This job reads the raw data from the Bronze S3 bucket (using the schema from the Glue Data Catalog). It then applies transformations like:
        * Handling missing or null values.
        * Standardizing date/time formats.
        * Correcting data types.
        * Perhaps masking sensitive information.
    * The cleaned and standardized data is written back to a different location in **Amazon S3** – the **Silver layer**.
    * *How this helps:* This ensures data quality and consistency, creating a reliable dataset suitable for further processing or direct querying if needed.

6.  **Preparing for Analysis (Silver → Gold):**
    * After the Silver layer is updated, Step Functions initiates the next stage, which involves loading data into **Amazon Redshift** (the Gold layer). This might be done by:
        * A second **AWS Glue ETL job** that reads from the Silver S3 layer, performs business-specific aggregations (e.g., calculating total daily sales per product), joins data (e.g., joining order data with customer data), and then writes the final, aggregated results to tables in Redshift.
        * *(Alternatively, as the diagram hints)* A Glue job might load Silver data into *staging* tables in Redshift, and then Step Functions could trigger a **Stored Procedure** within Redshift itself to perform the final transformations and load the Gold tables.
    * *How this helps:* This step creates the final, business-ready datasets optimized for analytics queries in Redshift. Aggregating data here makes reporting much faster.

7.  **Visualizing the Results:**
    * Finally, **Amazon QuickSight** connects directly to the Gold layer tables in **Amazon Redshift**.
    * Business users can access pre-built dashboards or explore the data interactively through QuickSight's interface.
    * *How this helps:* This provides easy, on-demand access to insights derived from the processed data, replacing manual report creation.

8.  **Security Foundation:** All network traffic between sensitive resources like RDS, DMS, and Redshift is secured within an **Amazon VPC (Virtual Private Cloud)**, using **VPC Endpoints** for private communication with services like S3 and Glue where needed.

This automated, step-by-step process ensures data flows reliably and efficiently from its raw state to valuable business insights.

### Automated Pipeline Trigger: Amazon EventBridge

The entire data pipeline process is initiated automatically using Amazon EventBridge (formerly CloudWatch Events). A scheduled rule is configured to trigger the pipeline workflow at regular intervals (e.g., daily, hourly). This eliminates the need for manual starts and ensures that the data is consistently refreshed for analysis.

EventBridge acts as the serverless scheduler, invoking the AWS Step Functions state machine which then orchestrates the downstream tasks like DMS replication and Glue transformations.

![EventBridge](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/4a1e83a81711357331833f70f7a5cbd305fecae3/eventbridge.png)

### Data Ingestion: AWS Database Migration Service (DMS)

The initial step of getting data from the source transactional database (Amazon RDS MySQL) into our data lake is handled by AWS Database Migration Service (DMS). We configure a DMS replication task to connect to the source database (using credentials securely stored in AWS Secrets Manager) and replicate the data to our designated Amazon S3 bucket (the Bronze layer).

DMS can be configured for continuous replication using Change Data Capture (CDC) or for one-time full loads. This service is crucial as it automates the complex process of data extraction and reliably transfers raw data to S3 with minimal impact on the performance of the source database.

![AWS DMS Replication Task Status](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/42ea2bf9b09e76a396380d8423a2aace2e6ad5e9/dms.png)


### Data Lake Storage: Amazon S3

Amazon Simple Storage Service (S3) serves as the central data lake for this pipeline. It provides a highly durable, scalable, and cost-effective platform to store data during various stages of processing, following the Medallion architecture principles:

* **Bronze Layer:** Raw data lands here directly from the DMS replication task. It's an immutable copy of the source data, providing a historical archive.
* **Silver Layer:** Data that has been cleaned, validated, and transformed by AWS Glue jobs is stored here. This layer typically serves as the source for further analytics or loading into the data warehouse.

Using S3 decouples storage from the compute services (like AWS Glue and Redshift), allowing each to scale independently. The structured layers within S3 help maintain data lineage and quality throughout the pipeline.

![S3 Bucket Structure Showing Bronze and Silver Layers](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/ee1b4c1e9b763e60a5388f55f6ab1079cf2ca520/s3.png)
### Data Transformation: AWS Glue Jobs

AWS Glue is the serverless ETL (Extract, Transform, Load) service used to process and transform the data within our pipeline. We utilize several components of Glue:

* **Glue Crawlers:** These are configured to scan the data arriving in the S3 Bronze layer. They automatically infer the schema and update the AWS Glue Data Catalog, making the data readily queryable by ETL jobs.
* **Glue Data Catalog:** Acts as a central metadata repository for all datasets within the pipeline.
* **Glue ETL Jobs:** These are serverless Apache Spark jobs (written typically in Python or Scala) that perform the heavy lifting of data transformation. In this project:
    * One job reads data from the S3 Bronze layer, applies cleaning and structuring rules, and writes the results to the S3 Silver layer.
    * Another job reads data from the S3 Silver layer, performs aggregations and business-logic transformations, and loads the final data into the Amazon Redshift Gold layer tables.

Using AWS Glue allows us to perform complex data transformations at scale without managing any underlying server infrastructure. The jobs are triggered as part of the Step Functions workflow.

![AWS Glue Job Run History](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/d0f25dbb2aab999cd5cd5c93b2b03d266f512701/gluejobs.png)
### Workflow Orchestration Logic: AWS Step Functions Design

AWS Step Functions is used to orchestrate the entire data pipeline, coordinating the various AWS services involved and managing the dependencies between them. The logic of this workflow is defined visually using the Step Functions Workflow Studio (or by writing Amazon States Language JSON).

This visual designer allows us to easily define the sequence of operations (e.g., run the Glue Crawler, then run the first Glue job, then the second), implement parallel processing where applicable (like processing different data dimensions simultaneously), add error handling, and include conditional logic. It provides a clear graphical representation of the pipeline's control flow.

![Step Functions Workflow Design View](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/336c2beb43a95f95471639441d264220503d064f/StepFunctions.png)
### Pipeline Monitoring: Step Functions Execution Graph

While the Step Functions designer view shows *how* the pipeline is structured, the execution graph provides a visual representation of a *specific run* of the pipeline. This is invaluable for monitoring the progress and troubleshooting any issues.

The graph highlights the path taken by the execution, showing which steps were successful (usually in green), which failed (red), and which might be currently in progress. It allows us to quickly identify bottlenecks or points of failure within our orchestrated workflow.

![Successful Step Functions Pipeline Execution](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/9b397151fc6bf42c4ea78462b75d82093466cb04/SuccessfulStepFunctions.png)

### Data Warehouse: Amazon Redshift Serverless

The final destination for our processed, analysis-ready data (the Gold layer) is Amazon Redshift Serverless. Redshift is a fast, fully managed, petabyte-scale data warehouse service optimized for analytical queries. The serverless option automatically provisions and scales the underlying compute capacity (measured in RPUs - Redshift Processing Units) based on workload demands, making it cost-effective and easy to manage.

The Redshift Serverless dashboard in the AWS Management Console provides an overview of the namespace and workgroup status, compute capacity usage, available storage, and access to query monitoring tools. It confirms that our data warehouse endpoint is active and ready to serve data to analysts and visualization tools like QuickSight.

![Redshift Serverless Console Dashboard](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/02e5d51690d6591362bb67c832588179a1574cf6/redshift.png)
### Verifying the Output: Redshift Query Results

After the pipeline successfully completes, the processed and aggregated data resides in the Gold layer tables within Amazon Redshift. We can verify the output by connecting to the Redshift database using the built-in Query Editor V2 or any standard SQL client.

Running queries against the final tables allows us to inspect the structured, analysis-ready data. This confirms that the ETL logic in the Glue jobs performed the correct transformations and that the data warehouse contains the expected information for reporting and visualization.

![Sample Query Result from Redshift Gold Table](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/e60d23b746dc8068396cf48be34cb1276e6da021/RedshiftQuery.png)
### Data Visualization: Amazon QuickSight Dashboard

The ultimate goal of this data pipeline is to provide actionable insights to business stakeholders. Amazon QuickSight is the AWS Business Intelligence (BI) service used to connect to our Amazon Redshift Gold layer data and create interactive visualizations and dashboards.

QuickSight allows users to easily explore the processed data, identify trends, track key performance indicators (KPIs), and gain a deeper understanding of the e-commerce operations (e.g., sales performance, customer behavior). The dashboards can be shared across the organization, providing a single source of truth for data-driven decision-making. This replaces the need for manual report generation in tools like Excel.

![E-commerce Insights Dashboard in QuickSight](https://github.com/anshpman/AWS-Data-Engineering-Project/blob/eba34557ec411daf8b047d55f4c5b9a7cf60ad43/quicksight.png)
