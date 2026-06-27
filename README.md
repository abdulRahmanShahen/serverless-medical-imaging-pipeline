![image alt](https://github.com/abdulRahmanShahen/serverless-medical-imaging-pipeline/blob/061b5f5de2282d3771c583a6b876c526cf0e8cc9/Blank%20diagram%20(30).png)
# Serverless Medical Imaging & Prescription Processing Pipeline
### AWS Solutions Architect - Associate (SAA-C03) Graduation Project
**Author:** [Abdul-Rahman Mohamad Shaheen]  
**Academic & Professional Background:** PharmD (Clinical Pharmacist) | Salesforce Developer 

---

## 1. Project Overview & Business Case

In modern healthcare systems, the rapid, secure, and cost-effective ingestion of medical assets (such as high-resolution X-Rays, MRI scans, and electronic/handwritten prescriptions) is a critical bottleneck. Traditional on-premises or EC2-based monolithic architectures suffer from structural inefficiencies: they require expensive, continuous provisioning to handle peak hospital hours, yet sit idle during off-peak times, incurring massive waste.

This project implements a **Cloud-Native, Production-Grade Serverless Pipeline** designed to ingest, decouple, process, and catalog medical imagery and clinical documentation at scale. By leveraging AWS Serverless primitives, the architecture eliminates infrastructure management overhead, ensures strict adherence to data availability, and scales dynamically from zero to thousands of concurrent requests.

### Key Medical Healthcare Use Cases:
* **Immediate Clinical Availability:** Generates optimized image configurations  instantly, allowing clinicians to view diagnostic images on lower-bandwidth mobile devices or tablets at the bedside.
* **Decoupled High-Volume Ingestion:** Prevents hospital system crashes during localized crises or peak morning rounds by absorbing bursts of incoming medical assets through an asynchronous messaging queue.
* **Structured Metadata Indexing:** Automatically extracts image transport payloads and logs patient asset transactions into a high-throughput, low-latency NoSQL database for seamless integration with downstream electronic health records (EHR) and Salesforce Health Cloud.

---

## 2. Solution Architecture Diagram

```
                       [ High-Resolution Images / Prescriptions ]
                                           │
                                           ▼
                                ┌──────────────────────┐
                                │  Amazon API Gateway  │ (Secure HTTPS REST Endpoints)
                                └──────────────────────┘
                                           │
                                           ▼
                                ┌──────────────────────┐
                                │   Amazon S3 Bucket   │ (Source: raw-medical-images)
                                └──────────────────────┘
                                           │
                                           │ (S3 Event Notification)
                                           ▼
                                ┌──────────────────────┐
                                │   Amazon SQS Queue   │ (Decoupling & Load Buffering)
                                └──────────────────────┘
                                           │
                                           │ (Event Source Mapping)
                                           ▼
                                ┌──────────────────────┐
                                │  AWS Lambda Function │ (Compute: Processing & Resizing)
                                └──────────────────────┘
                                     /            \
                       (Save Thumbnail)          (Write Metadata)
                                   /                \
                                  ▼                  ▼
                    ┌──────────────────┐    ┌──────────────────┐
                    │ Amazon S3 Bucket │    │ Amazon DynamoDB  │
                    │(Destination:     │    │ (Patient Asset   │
                    │ medical-thumbs)  │    │  Metadata Table) │
                    └──────────────────┘    └──────────────────┘
```

### Components and Connections (For Lucidchart Reproduction):
1.  **Ingestion:** Client requests interface with an **Amazon API Gateway** endpoint configured to generate secure S3 pre-signed URLs or proxy upload payloads.
2.  **Raw Storage:** Uploaded files land securely inside an **Amazon S3 Source Bucket** (`raw-medical-images`).
3.  **Decoupling:** S3 triggers an **Event Notification** upon a successful `s3:ObjectCreated:*` action, publishing a message payload to an **Amazon SQS Queue**.
4.  **Compute execution:** **AWS Lambda** polls the SQS queue via an Event Source Mapping, processing the messages in batches.
5.  **Multi-Target Output:** * Lambda generates an optimized image thumbnail using the `Pillow` library and writes it to a downstream **Amazon S3 Destination Bucket** (`processed-medical-thumbnails`).
    * Lambda extracts image parameters (`BucketName`, `ObjectKey`, `FileSize`, `Timestamp`) and commits an atomic item insertion into an **Amazon DynamoDB Table** (`MedicalAssetMetadata`).

---

## 3. Detailed AWS Services Breakdown

### 📊 Amazon S3 (Simple Storage Service)
* **Role:** Acts as both the secure landing zone for raw medical imagery and the durable hosting tier for optimized assets.
* **Design Considerations for SAA Exam:**
    * **Security:** Enforces Encryption-at-Rest using SSE-S3 or AWS KMS managed keys. Bucket policies are strictly configured to deny public read access.
    * **Lifecycle Policies:** Transition raw, unedited high-resolution scans to *S3 Standard-Infrequent Access (IA)* after 30 days, and automatically archive to *S3 Glacier Flexible Retrieval* after 90 days to minimize long-term storage costs while maintaining regulatory medical compliance.

### 🔄 Amazon SQS (Simple Queue Service)
* **Role:** Provides asynchronous decoupling between the ingestion layer and the compute engine.
* **Design Considerations for SAA Exam:**
    * **Resilience & Load-Shedding:** Ensures that if an unexpected spike occurs (e.g., thousands of diagnostic uploads within a few minutes), the backend storage and processing layers do not experience throttling or loss of data.
    * **Dead-Letter Queue (DLQ):** Configured with a Redrive Policy. If a corrupted image file causes the Lambda function to fail processing repeatedly (maxReceiveCount = 3), the message is routed to an SQS DLQ (`medical-pipeline-dlq`) for isolation and auditing.

### ⚡ AWS Lambda
* **Role:** Serverless event-driven compute engine that processes transactions without server provisioning.
* **Design Considerations for SAA Exam:**
    * **IAM Execution Role:** Follows the Principle of Least Privilege. The execution role explicitly allows `sqs:ReceiveMessage`, `s3:GetObject` (from source bucket), `s3:PutObject` (to destination bucket), and `dynamodb:PutItem`. It has no full administrative rights.
    * **Dependencies:** External image manipulation runtimes (`Pillow`) are packaged neatly inside a dedicated **AWS Lambda Layer**, keeping the deployment package small, modular, and rapid to load.

### 🔑 Amazon DynamoDB
* **Role:** Low-latency NoSQL database providing consistent, single-digit millisecond performance at any scale for patient asset tracking.
* **Design Considerations for SAA Exam:**
    * **Capacity Modes:** Utilizes *On-Demand Capacity Mode* to absorb variable, unpredictable medical traffic patterns without over-paying for pre-allocated Read/Write Capacity Units (RCUs/WCUs).
    * **Schema Layout:** Key structure uses `PatientID` (String) as the Partition Key (PK) and `AssetTimestamp` (Number/String) as the Sort Key (SK) to enable efficient, fast index querying per patient record.

---

## 4. Step-by-Step Implementation Guide

Follow these sequential steps to deploy this architecture within your AWS Console environment:

### Step 1: Create the Amazon S3 Buckets
1.  Navigate to the **S3 Console** and click **Create bucket**.
2.  Create the Input Bucket: Name it uniquely (e.g., `yourname-raw-medical-images`). Keep ACLs disabled and ensure **Block all public access** is checked.
3.  Create the Output Bucket: Name it uniquely (e.g., `yourname-processed-medical-thumbnails`). Keep public access blocked.

### Step 2: Create the Amazon SQS Queue & Dead Letter Queue
1.  Navigate to the **SQS Console** and click **Create queue**.
2.  Select **Standard Queue**. Name it `medical-pipeline-dlq`. Click **Create queue**.
3.  Click **Create queue** again to build the main queue. Select **Standard Queue**. Name it `medical-processing-queue`.
4.  Under *Dead-letter queue settings*, enable it, select your `medical-pipeline-dlq`, and set the *Maximum receives* to `3`. Click **Create queue**.

### Step 3: Configure S3 Event Notifications
1.  Return to your Input S3 Bucket (`yourname-raw-medical-images`), click the **Properties** tab, and scroll down to **Event notifications**.
2.  Click **Create event notification**.
3.  Name it `SendToSQS`. Select **All object creation events** (`s3:ObjectCreated:*`).
4.  Under *Destination*, choose **SQS queue** and select `medical-processing-queue`. Click **Save changes**.

### Step 4: Create the Amazon DynamoDB Table
1.  Navigate to the **DynamoDB Console** and click **Create table**.
2.  Table name: `MedicalAssetMetadata`.
3.  Partition key: `PatientID` (Type: String).
4.  Sort key: `AssetTimestamp` (Type: String).
5.  Under *Table settings*, choose **Customize settings**, change the capacity mode to **On-Demand**, and click **Create table**.

### Step 5: Provision the AWS Lambda Function
1.  Navigate to the **Lambda Console** and click **Create function**.
2.  Select **Author from scratch**. Function name: `MedicalImageProcessor`. Runtime: **Python 3.11** (or later).
3.  Under *Permissions*, expand *Change default execution role*. Choose *Create a new role with basic Lambda permissions*. Click **Create function**.
4.  **Update IAM Permissions:** * Go to the *Configuration* tab of the function, select *Permissions*, and click on the generated Execution Role name to open it in the IAM Console.
    * Click *Add permissions* -> *Attach policies*, and add the managed policy: `AWSLambdaSQSQueueExecutionRole`.
    * Add an inline policy allowing `s3:GetObject` on the raw bucket, `s3:PutObject` on the thumbnail bucket, and `dynamodb:PutItem` on the `MedicalAssetMetadata` table.
5.  **Add SQS Trigger:** * In the Lambda Function Designer visualization, click **Add trigger**.
    * Select **SQS** as the source, pick `medical-processing-queue`, set batch size to `10`, and click **Add**.

---

## 5. Deployment Verification & Testing

To verify the functional integrity of the decoupled pipeline:

1.  **Simulate Upload:** Upload a standard sample image file (e.g., `xray_patient_99.jpg`) directly into your source S3 bucket (`yourname-raw-medical-images`).
2.  **Trace Queue Execution:** Check the SQS metrics. You will observe a brief spike in *Messages Available* followed instantly by *Messages Consumed*, proving that S3 successfully fired the event notification and Lambda successfully processed the message out of the queue.
3.  **Verify DB Records:** Navigate to your DynamoDB table `MedicalAssetMetadata` and click **Explore table items**. You will see a newly minted metadata entry containing the specific file details, operational timestamps, and references to the patient identifier.
4.  **Confirm Thumbnail Creation:** Check the destination S3 bucket (`yourname-processed-medical-thumbnails`) to confirm that a corresponding processed, downscaled thumbnail asset resides safely within the target folder structure.
