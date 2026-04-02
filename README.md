# 📸 Serverless Media Processing Pipeline (AWS)

## 🚀 Project Overview

This project implements a **fully serverless image processing pipeline** using AWS services. The system is designed to handle large-scale image uploads, automatically process them, and distribute globally.

### ✅ Key Features

* Upload images securely using **Pre-Signed URLs**
* Automatic **image resizing & watermarking** via Lambda
* Store processed images in a separate bucket
* Global content delivery using **CloudFront**
* Lifecycle policies for cost optimization
* Fault tolerance using **SQS Dead Letter Queue (DLQ)**
* Cross-region replication for durability

---

## 🏗️ Architecture Flow

```
Browser
   ↓
PreSigned URL
   ↓
S3 Source Bucket (rawuploads/)
   ↓ (S3 Event Trigger)
Lambda (Resize + Watermark)
   ↓
S3 Processed Bucket (processed/)
   ↓
CloudFront Distribution (+ Lambda@Edge)
   ↓
Global End Users
```

---

## ⚙️ Implementation Steps

---

## 1️⃣ S3 Setup

### 1.1 Create Source Bucket

* Go to AWS Console → S3
* Click **Create bucket**
* Name: `rawuploadsbucket`
* Enable **Versioning**
* Enable **Default Encryption (SSE-S3)**
* Create bucket
<img width="692" height="349" alt="image" src="https://github.com/user-attachments/assets/ba70784a-fa81-4cd1-a2b0-56c9e77c873f" />
<img width="691" height="310" alt="image" src="https://github.com/user-attachments/assets/803f6da2-7c0b-4b6b-b89e-2aef343dca8f" />
<img width="691" height="312" alt="image" src="https://github.com/user-attachments/assets/64259531-303c-4017-8830-28ca5f4255d3" />

---

### 1.2 Configure CORS

* Open `rawuploadsbucket`
* Go to **Permissions → CORS configuration**
* Add:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["PUT", "GET"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": []
  }
]
```
<img width="691" height="243" alt="image" src="https://github.com/user-attachments/assets/bc267f1e-b223-48cf-a8df-bf656c646820" />


---

### 1.3 Create Processed Bucket

* Go to S3 → Create bucket
* Name: `processedimagesbucket`
* Enable **Default Encryption → SSE-KMS**
* Select/create KMS key
* Create bucket

---

## 2️⃣ Cross-Region Replication (CRR)

* Create destination bucket in another region (e.g., EU-West-1)
* Enable **Versioning** on both buckets
* Open `rawuploadsbucket`
* Go to **Management → Replication rules**
* Configure:

  * Source prefix: `rawuploads/`
  * Destination: EU bucket
* Create IAM role
* Save rule
<img width="691" height="287" alt="image" src="https://github.com/user-attachments/assets/2be0e769-5cf6-4b3f-82b8-b95e60be1200" />
<img width="691" height="282" alt="image" src="https://github.com/user-attachments/assets/bb1ec606-4b87-4f5b-a933-4481ba052095" />
---

## 3️⃣ Lifecycle Rules

* Open `rawuploadsbucket`
* Go to **Management → Lifecycle rules**
* Create rule:

  * Prefix: `rawuploads/`
* Add transitions:

  * Standard → IA (30 days)
  * IA → Glacier (90 days)
  * Glacier → Deep Archive (365 days)
<img width="691" height="313" alt="image" src="https://github.com/user-attachments/assets/799fd2ab-a629-4e2a-9d22-b086e065214c" />
<img width="691" height="269" alt="image" src="https://github.com/user-attachments/assets/5dd80a41-7a76-4b23-83c5-afc99c71ad0c" />
<img width="691" height="255" alt="image" src="https://github.com/user-attachments/assets/04936eb8-877d-4145-92dd-250092e22847" />

---

## 4️⃣ Pre-Signed URL Setup

### Lambda Code for Pre-Signed URL

```python
from botocore.client import Config
import boto3
import json

s3 = boto3.client('s3', config=Config(signature_version='s3v4'))

SOURCE_BUCKET = 'rawuploadsbucket'

def lambda_handler(event, context):
    qs = event.get('queryStringParameters') or {}
    filename = qs.get('filename', 'upload.jpg')
    key = "logs/" + filename

    url = s3.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': SOURCE_BUCKET,
            'Key': key,
            'ContentType': 'image/jpeg'
        },
        ExpiresIn=900
    )

    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': '*',
            'Content-Type': 'application/json'
        },
        'body': json.dumps({
            'upload_url': url,
            'key': key
        })
    }
```
<img width="691" height="298" alt="image" src="https://github.com/user-attachments/assets/e6b8dbce-0ba0-4e27-bd7f-3af98f8eff15" />
<img width="691" height="295" alt="image" src="https://github.com/user-attachments/assets/0901ed59-80b2-41a6-b11d-c80057257ffb" />
<img width="691" height="282" alt="image" src="https://github.com/user-attachments/assets/5afbb2ad-9211-4cfd-8f88-da519b1203ce" />

---

## 5️⃣ Lambda Setup

### 5.1 Create Function

* Go to AWS Lambda
* Click **Create function**
* Name: `imageprocessingfunction`
* Runtime: Python 3.11

---

### 5.2 Configure Settings

* Memory: **512 MB**
* Timeout: **30 seconds**
<img width="691" height="300" alt="image" src="https://github.com/user-attachments/assets/4a89db22-761a-4250-8d3e-61f8a16bc8a6" />

---

### 5.3 Add S3 Trigger

* Add trigger:

  * Source: S3
  * Bucket: `rawuploadsbucket`
  * Event: Object Created
<img width="691" height="303" alt="image" src="https://github.com/user-attachments/assets/a330cd73-11c1-44d1-b785-3c04e56ac969" />
---

### 5.4 Create Lambda Layer (for Pillow)

```bash
mkdir python
pip install pillow -t python/
zip -r pillow-layer.zip python
```

* Go to Lambda → Layers → Create layer
* Upload zip
* Select Python 3.11
<img width="691" height="283" alt="image" src="https://github.com/user-attachments/assets/592584e6-bed3-44f1-8f1b-e3dc0a8f19ca" />
<img width="691" height="304" alt="image" src="https://github.com/user-attachments/assets/aff036f5-336e-488c-ad06-8642ea7ac93e" />
<img width="691" height="143" alt="image" src="https://github.com/user-attachments/assets/04e40537-2d86-4fb9-a7c8-3790d03e3b42" />

---

### 5.5 Attach Layer

* Open Lambda function
* Go to Layers → Add layer
* Select custom layer

---

### 5.6 Add Permissions

Attach policy to execution role:

```
AmazonS3FullAccess
```

---

## 6️⃣ Image Processing Logic (Lambda)

```python
from PIL import Image
import boto3
import io

s3 = boto3.client('s3')

SOURCE_BUCKET = 'rawuploadsbucket'
DEST_BUCKET = 'processedimagesbucket'

def lambda_handler(event, context):
    for record in event['Records']:
        key = record['s3']['object']['key']

        response = s3.get_object(Bucket=SOURCE_BUCKET, Key=key)
        image_content = response['Body'].read()

        image = Image.open(io.BytesIO(image_content))
        image = image.resize((300, 300))

        buffer = io.BytesIO()
        image.save(buffer, 'JPEG')
        buffer.seek(0)

        s3.put_object(
            Bucket=DEST_BUCKET,
            Key=key,
            Body=buffer,
            ContentType='image/jpeg'
        )

    return {"status": "processed"}
```
<img width="1920" height="1080" alt="Screenshot (479)" src="https://github.com/user-attachments/assets/5ca3c33e-3eaf-49a0-9df0-59f8e08ba927" />

---

## 7️⃣ Final Validation

### Step 1: Generate Pre-Signed URL

Open your Lambda Function URL in browser.

---

### Step 2: Copy `upload_url`

---

### Step 3: Upload via CLI

```bash
curl -X PUT "PASTE_UPLOAD_URL" \
  -H "Content-Type: image/jpeg" \
  --upload-file test.jpg
```

---

### Step 4: Verify

* Check:

  * `rawuploadsbucket/logs/`
  * `processedimagesbucket/logs/`

---

## ✅ Expected Outcome

* Image uploaded to **raw bucket**
* Lambda triggers automatically
* Image resized & processed
* Output stored in **processed bucket**
<img width="691" height="315" alt="image" src="https://github.com/user-attachments/assets/bfc3a043-db45-481e-9d43-36d1ead93885" />

---

## 🎯 Services Used

* AWS S3
* AWS Lambda
* AWS CloudFront
* AWS IAM
* AWS SQS (DLQ)
* AWS KMS


