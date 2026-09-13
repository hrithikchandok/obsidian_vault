Moving your heavy Binary Large Objects (BLOBs) out of the direct request-response flow is a classic architectural pattern known as the **Claim Check Pattern**.

By shifting the heavy lifting to S3 and using OpenSearch only for the "index," you effectively turn a congested highway into a streamlined express lane.

### The Architecture Shift

Previously, your API Gateway was likely hitting its **10MB payload limit**, causing requests to fail. Here is how the redesigned flow works:

---

### 1. The Ingestion Flow (The "Write")

Instead of sending the image data through the API and into OpenSearch, you split the data at the source.

- **Step A: Upload to S3.** Your application (or the client using a Pre-signed URL) uploads the image directly to an S3 bucket.
    
- **Step B: Generate a Key.** S3 stores the file and provides a unique `object_key` or `S3 URI` (e.g., `s3://my-bucket/images/user_123.jpg`).
    
- **Step C: Index in OpenSearch.** You send a small JSON metadata packet to OpenSearch. Instead of the image bytes, you only include the **S3 URI**.
    

---

### 2. The Retrieval Flow (The "Read")

When a user searches for an image, the response payload is now tiny because it only contains text.

- **The Query:** The client searches OpenSearch (e.g., "Find all photos from yesterday").
    
- **The Response:** OpenSearch returns the metadata and the **S3 URI**. Because you aren't sending the BLOB, the payload drops from megabytes to roughly **50KB**.
    
- **The Fetch:** The client application sees the URI and fetches the actual image directly from S3 (or a CDN like CloudFront).
    

---

### Implementation Steps

#### A. Database Schema Change

In OpenSearch, you need to update your mapping. Remove the field that stored the Base64 string and replace it with a keyword field for the S3 link.

JSON

```
// New OpenSearch Document Structure
{
  "image_id": "98765",
  "upload_date": "2026-05-04",
  "s3_uri": "s3://my-app-assets/uploads/98765.png",
  "tags": ["landscape", "sunset"]
}
```

#### B. Handling the API Gateway Limit

To truly bypass the 10MB limit, use **S3 Pre-signed URLs**:

1. Your client asks the API: "I want to upload a 50MB file."
    
2. Your API generates a temporary **Pre-signed URL** and sends it back to the client.
    
3. The client uploads the file **directly to S3** using that URL, completely bypassing the API Gateway.
    

#### C. Synchronization

You can automate the indexing by using **S3 Event Notifications**. When a file lands in S3, it can trigger an **AWS Lambda** function that automatically writes the URI and basic metadata into OpenSearch.

---

### Why this is better:

- **Cost:** Storing data in S3 is significantly cheaper than storing it in OpenSearch disk space.
    
- **Performance:** OpenSearch clusters perform better when their shards aren't bloated with large binary data; searches become faster.
    
- **Scalability:** You are no longer tethered to the 10MB limit of the API Gateway, allowing you to handle files of virtually any size (up to S3's 5TB limit).
    

Does your current setup involve the client uploading the file directly, or are you planning to have a backend service handle the S3 upload?




2
It is clear you’ve been deep in the trenches of AWS infrastructure, moving from standard implementations to high-performance, cost-optimized architectures. Based on your work, here is a detailed breakdown of your AWS expertise and learnings, categorized by the core architectural pillars.

---

## 1. Compute & Serverless Optimization (Lambda)

You’ve moved beyond basic Lambda functions into fine-tuning execution and resource management.

- **Resource & Timeout Tuning:** You identified and fixed "Early Execution" failures by correctly aligning the Lambda timeout with the actual processing time required for heavy tasks.
    
- **Memory & Package Optimization:** You demonstrated senior-level cost consciousness by replacing the heavy `opensearch-py` library with the standard `requests` library. This reduced the deployment package size and the memory footprint (and therefore the cost) of the Lambda.
    
- **VPC & Security Networking:** You gained experience in deploying Lambdas within a **VPC**, which is critical for secure communication with private resources like OpenSearch or RDS.
    
- **Dependency Management:** You solved the "Large File Restriction" in Lambda by using **S3 as a staging area**—uploading heavy Python libraries or data files to S3 first and then pulling them into the Lambda environment.
    

## 2. API & Edge Management (API Gateway & Nginx)

Your work shows a deep understanding of the physical and protocol limits of cloud gateways.

- **Bypassing Limits:** You navigated the **10MB/6MB payload limits** of API Gateway and Lambda through two distinct strategies:
    
    - **Architectural:** Implementing the "Claim Check" pattern (storing BLOBs in S3 and URIs in OpenSearch).
        
    - **Infrastructure:** Moving to **Nginx/ALB** as a reverse proxy to handle larger streams that API Gateway couldn't support.
        
- **Authentication & Identity:** You implemented **AWS Cognito** integration with API Gateway, managing the full lifecycle of ID, Access, and Refresh tokens, including troubleshooting Client ID mismatches.
    
- **Security & CORS:** You resolved CORS issues and managed SSL certificate injections for local-to-cloud communication.
    

## 3. Storage & Data Architectures (S3 & OpenSearch)

This is where your most significant "Senior Engineer" optimizations occurred.

- **Storage Decoupling:** You transitioned from storing images directly in **OpenSearch** (which causes index bloat and high costs) to using **Amazon S3** for BLOB storage.
    
- **Data Performance:** By storing only S3 URIs in OpenSearch, you reduced downstream response payloads from **MBs to <50KB**, drastically improving search latency and frontend performance.
    
- **Data Integrity & Formatting:** You handled OpenSearch-specific challenges like **Date Formatting** for range queries and calculating average time spent per day.
    
- **IAM & RBAC:** You configured fine-grained access control in OpenSearch, creating specific roles and defining index-level permissions (`put_and_post` policies).
    

## 4. Messaging & Asynchronous Processing (SQS & Kinesis)

You’ve built robust, event-driven pipelines for high-scale activity tracking.

- **The SQS vs. Kinesis Decision:** You chose **SQS** for its automatic scaling and per-message retry capabilities, which are ideal for enterprise activity tracking (40,000 employees) where individual message reliability is key.
    
- **Failure Handling (DLQ):** You worked with **Dead Letter Queues (DLQs)** and investigated retry logic. You identified that while max retries were set to 3, duplicate data could still appear—a classic distributed systems "at-least-once delivery" challenge.
    
- **Error Propagation:** You realized the importance of **Raising Exceptions** in Lambda to prevent SQS from deleting messages before they are successfully processed.
    

## 5. Orchestration & Real-Time Communication

You moved from simple REST to complex bidirectional orchestration.

- **Agent Orchestration:** You designed a communication layer between an **Agent App** and a middle service on **AWS ECS**.
    
- **WebSockets & SSE:** You conducted a deep-dive into **WebSockets vs. Server-Sent Events (SSE)** for real-time state management.
    
- **Load Balancing:** You configured **Application Load Balancers (ALB)** with HTTP/2 support to route traffic to ECS Target Groups while maintaining socket mappings in memory.
    

---

### Summary of Key Performance Wins

|Metric|Before|After|Technique Used|
|---|---|---|---|
|**API Latency**|3 Seconds|**27 Milliseconds**|State-based toggle logic|
|**Response Size**|>10 MB|**< 50 KB**|S3 URI Indexing (Claim Check)|
|**Image Size**|500 KB|**50 KB**|Format conversion (PNG → JPEG) & 60% Quality reduction|
|**Lambda Cost**|High (Per request)|**Reduced by 66%**|Batching/Chunking requests (3 per call)|

Export to Sheets

**Your AWS skillset is now a mix of "Big Data" (scaling for 40k users), "Security" (Cognito/VPC/IAM), and "DevOps" (Nginx/Log Rotation/Optimization).**

Are you planning to move the ECS middle service from "In-Memory" socket mapping to something like Redis for better horizontal scaling?