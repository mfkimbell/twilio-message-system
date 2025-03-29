# twilio-message-system

# Overview

<img width="1143" alt="Screenshot 2025-03-28 at 7 07 51 PM" src="https://github.com/user-attachments/assets/33d914a8-12c6-4dd3-a133-2993fb119751" />


SMS MMS VOICE EMAIL ETC

## 🧠 What is Twilio?

### **Twilio** is a cloud-based platform that provides **communication APIs** for **SMS, MMS, voice, email (SendGrid), WhatsApp, and video**.
### Twilio is a **tool** to help solve certain communication prolems in produciton applications
- It lets you **send/receive messages and calls via code** — no telecom hardware required.
- Used for 2FA, notifications, support bots, alerts, appointment reminders, and more.

## Customers like **AirB&B** Lyft and UBER

<img width="671" alt="Screenshot 2025-03-28 at 7 14 26 PM" src="https://github.com/user-attachments/assets/f5f20720-b719-48f3-ba16-6c13366c7fd4" />

---

## 📦 Key APIs You Should Know

### ✅ **SMS / MMS API**
- Send SMS with `messages.create()`
- Optional: add `mediaUrl` for MMS (e.g., images, PDFs)
- Can include a `statusCallback` URL to get delivery updates

### ✅ **Voice API**
- Twilio calls your backend to fetch TwiML (XML instructions)
- You can play audio, gather digits (IVR), or forward calls

### ✅ **Webhooks**
- Twilio sends HTTP POSTs to your server for:
  - Inbound SMS (`/twilio/incoming`)
  - Message status (`/twilio/status`)
  - Call events (answered, ended, failed, etc.)

---

## 🔁 Twilio Status Callbacks

- When you send a message, you can provide a `statusCallback` URL.
- Twilio will notify you as the message goes from:
  - `queued` → `sent` → `delivered` or `failed`

### Example Payload:
```json
{
  "MessageSid": "SMxxxx",
  "MessageStatus": "delivered",
  "To": "+1234567890"
}
```

You’d store this in a `NotificationLog` table.

---

## ⚠️ Handling Failures

- **Retries**: Twilio will retry webhook calls if you don’t return a `200 OK`.
- **Dead Letter Queue**: Use SQS DLQ to store failed messages for investigation/retry.
- Add logs + monitoring to track bounce rates or carrier issues.

---

## 📈 Scaling with Twilio

- **ECS Workers** or Lambdas send messages in bulk (scale with SQS queue depth).
- Respect Twilio’s **rate limits** (based on country/carrier).
- Use **worker-side throttling** or rate limiting (token bucket, Redis, etc.)

---

## 📸 Media (MMS)

- To **send MMS**, use a `mediaUrl` that points to a public (or pre-signed) file.
- To **receive MMS**, Twilio posts media URLs to your webhook (in `MediaUrl0`, `ContentType0`).

---

## 🧠 Pro Tips for Interviews

✅ Be ready to explain:
- How your app sends messages using Twilio
- How delivery status is tracked and logged
- How you handle failures (retries, DLQ)
- How you decouple sending with a queue + worker
- How Twilio talks to your backend (webhooks)

---

## 🧪 Sample Flow

1. User triggers event (e.g. file uploaded)
2. Backend sends message via Twilio SMS API
3. Includes `statusCallback` to `/twilio/status`
4. Worker logs `queued` → `sent` → `delivered`
5. Failed? Push to DLQ, alert, or retry

Worker logs it from the status api route:

<img width="771" alt="Screenshot 2025-03-28 at 7 06 08 PM" src="https://github.com/user-attachments/assets/be0a0058-a1bc-434f-ada7-ea6a1f1d0cae" />

---


# Project specific

### Twilio Message SID
When you send a message via Twilio, you get back a unique MessageSid, like:

```json

{
  "sid": "SMXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
```

You can use this SID to:
- Track the message status
- Fetch details from Twilio's API
- Listen for status updates

### 🖼️ **What is MMS?**

MMS = **Multimedia Messaging Service**

Unlike SMS (text-only), **MMS supports media**, like:
- Images
- Videos
- PDFs
- Audio

#### In Twilio:
You can send MMS by including a `MediaUrl` when sending a message:
```json
{
  "to": "+123456789",
  "from": "+TwilioNumber",
  "body": "Here's the image!",
  "mediaUrl": "https://mycdn.com/image.png"
}
```

You can also receive MMS messages — Twilio will POST media data (URLs, content types) to your webhook.

### Webhooks

Webhooks are like notification systems so that APIs can send data to configured endpoints on your application. 

#### 1. **Inbound Message Webhook**  
To receive **incoming SMS or MMS messages** (i.e., when a user texts your Twilio number):

```js
app.post('/twilio/incoming', (req, res) => {
  const message = req.body.Body;     // message text
  const from = req.body.From;        // sender phone number

  console.log(`Received message from ${from}: ${message}`);

  res.status(200).send(); // Acknowledge Twilio
});
```

- You configure this URL in your Twilio Console (or via API).
- Twilio hits it **when someone sends a message to you**.
- you **MUST RETURN 200** or twilio will retry sending

---

#### 2. **Status Callback Webhook**  
To receive **delivery status updates** for **outbound** messages you send via Twilio:

```js
app.post('/twilio/status', (req, res) => {
  const status = req.body.MessageStatus; // e.g. 'delivered', 'failed'
  const sid = req.body.MessageSid;

  if (status === 'failed' || status === 'undelivered') {
    enqueueToDLQ({ sid, status });
  }

  res.status(200).send();
});
```

- You pass this URL as `statusCallback` **when sending a message**
- Twilio will notify you as the message progresses from `queued` → `sent` → `delivered` or `failed`
- you **MUST RETURN 200** or twilio will retry sending



## Failure Strategies?

### Retries
- retries with exponential backoff
- 3-5 is pretty standard

### Dead Letter Queues

dead letter queues keep track of requests that fail all the retries, they keep the system running while storing the original payload (optionally add errors that went with it)

Twilio has a "status callback that you send in api requests to get the response" the incoming message URL is assigned in your twilio console via API. 

`AWS SQS` is perfectly fine to use for a dead letter queue

```json
{
  "to": "+123456789",
  "body": "Your order has shipped!"
}
```
gets sent to DLQ
```json
{
  "payload": {
    "to": "+123456789",
    "body": "Your order has shipped!"
  },
  "status": "undelivered",
  "error": "Invalid number format",
  "retryCount": 3,
  "failedAt": "2025-03-28T15:00:00Z"
}
```


```javascript
app.post('/twilio/status', (req, res) => {
  const status = req.body.MessageStatus;

  if (status === 'failed' || status === 'undelivered') {
    // Send this to a Dead Letter Queue for retry/investigation
    enqueueToDLQ(req.body);
  }

  res.status(200).send();
});
```

## Scaling Strategies?
- we use a queue for messages, workers pick them up and do work
- containerization and replilcation (also helps with failures) and replication in multiple availibility zones (disaster recovery)

Frontend – where users trigger messages (scales with CDN or loadbalancer)

Backend API – receives the request, sends to queue (scales with containerization and K8s or ECS, or API Gateway which auto-scales and then loadbalancing like NGINX, ALB, or K8s again)

Queue – stores message jobs temporarily (use SQS, RabbitMQ, or Kafka)

Worker Service – pulls from the queue, sends message via Twilio

It is a docker container (or multiple, but less common) that runs inside:

* A Pod (in Kubernetes)

* A Task (in ECS/Fargate)

Status Webhook API – receives delivery status updates from Twilio (can scale with loadbalancer and container autoscaling)

Optional DLQ – for failed jobs (can monitor with Prometheus + Grafana or Cloudwatch)

Database – stores messages, users, delivery logs (Write Sharding (a.k.a. horizontal partitioning), read replicas, or just using NoSQL for built in horizontal paritioning)

Monitoring + Alerts – logs, alerts, dashboards

## Rate Limiting?

### IP-Based Rate Limiting
- good for public APIS
- Can help block bot abuse or brute-force attacks

### User-Based Rate Limiting
- Tracks limits per authenticated user

### Global Request Rate Limiting 
- limits total number of requests per minute

### Implementing a Rate Limiter using Redit

### Bucket Agorithm for rate limiting approaches

#### Token Bucket
- **Allows for bursts**, and should always be preferred unless the system can't handle bursts
<img width="816" alt="Screenshot 2025-03-28 at 5 22 48 PM" src="https://github.com/user-attachments/assets/cf3ad3b8-a651-4078-98a0-aa724b8e43d1" />

In a typical Token Bucket setup, We check and refill tokens on every request — but only as much as time allows. we could use **UserID or IP**

<img width="763" alt="Screenshot 2025-03-28 at 5 26 52 PM" src="https://github.com/user-attachments/assets/c8f5c3a6-6932-4928-aafb-8e24dad55535" />

<img width="762" alt="Screenshot 2025-03-28 at 5 45 37 PM" src="https://github.com/user-attachments/assets/7208ed16-c5d7-46e8-b24e-65a4d0e31a27" />


#### Leaky Bucket
- Doesn't allow for bursts
- “You can send 1 message every second — period.”
→ You never earn extra. You must wait before the next one.

<img width="784" alt="Screenshot 2025-03-28 at 5 52 39 PM" src="https://github.com/user-attachments/assets/1b2e8e85-18ba-4fb8-9715-30edfc736e35" />

  
<img width="792" alt="Screenshot 2025-03-28 at 5 23 18 PM" src="https://github.com/user-attachments/assets/2097b62c-d912-48b1-9274-f42439e369c3" />

<img width="788" alt="Screenshot 2025-03-28 at 5 50 58 PM" src="https://github.com/user-attachments/assets/33305fbc-9d95-4dfa-88ab-53c8104d74df" />

#### Sliding Window 

Sliding Window Rate Limiting ensures that:

“No more than N requests are allowed in the last X seconds, regardless of exactly when those requests occurred.”

Instead of counting tokens or spacing requests, it tracks actual timestamps of each request and counts how many are within the current rolling window.

<img width="772" alt="Screenshot 2025-03-28 at 6 01 00 PM" src="https://github.com/user-attachments/assets/05f01548-a359-4a73-bd77-05bec8b637ce" />

<img width="775" alt="Screenshot 2025-03-28 at 6 01 23 PM" src="https://github.com/user-attachments/assets/9c270b47-f108-4eef-9721-678a708c7835" />



### Worker Side Throttling

```javascript
setInterval(() => {
  processMessageFromQueue(); // one message per tick
}, 200); // roughly 5 per second
```

## Media Handling

### Sending Media
We pass twilio an S3 URL (public pre-signed URL)
* CDN (like CloudFront) can be used to serve fast + regionally cached media

```json
{
  "to": "+123456789",
  "from": "+TwilioNumber",
  "body": "Here's your receipt!",
  "mediaUrl": "https://your-cdn.com/media/receipt.pdf"
}
```

### 📩 Receiving Media (Inbound MMS)
Twilio sends media metadata to your inbound webhook:

```json
NumMedia=1
MediaUrl0=https://api.twilio.com/2010-04-01/Accounts/.../Media/AB123...
MediaContentType0=image/jpeg
```

Perfect — this is a great architecture. Here’s the **full logic flow** based on what you're building, including upload, notification, search, and scaling details:

---

## 📁 File Upload & Notification System – Full Logic Flow

---

### 🧱 1. **Frontend (React App on ECS)**
- User selects a file and clicks "Upload"
- React app:
  - Requests a **pre-signed S3 URL** from backend
  - **A pre-signed URL is a temporary, permissioned URL that allows someone else to perform an S3 action (like upload or download) without needing AWS credentials.**
  - Uploads the file directly to S3 via that URL
  - Sends metadata to API Gateway (e.g., file name, UUID, type, upload time)

---

### 🔁 2. **Backend (API Gateway → Lambda or ECS API service)**
- Receives metadata from frontend
- Adds a message to **SQS Upload Queue**:

```json
{
  "uuid": "abc123",
  "fileName": "resume.pdf",
  "s3Url": "https://bucket.s3.amazonaws.com/abc123.pdf",
  "uploadedAt": "2025-03-28T12:00:00Z",
  "userId": "user_42"
}
```

---

### 🏗️ 3. **ECS Worker Service (Polls SQS)**
- Runs on a schedule or long-polling from the queue
- When a message arrives:
  1. Validates and transforms the data
  2. Saves the file metadata into the **database** (e.g., Postgres or DynamoDB)
  Here you go — the **`FileMetadata`** table with a couple of sample entries shown in plain table format:

---

#### 📄 `FileMetadata`

| id        | user_id | file_name     | file_type | s3_url                                                  | size_kb | uploaded_at          | tags                    |
|-----------|---------|---------------|-----------|----------------------------------------------------------|---------|-----------------------|--------------------------|
| file-001  | user-42 | invoice.pdf   | pdf       | https://s3.amazonaws.com/bucket/invoice.pdf             | 210     | 2025-03-28 12:00:00   | {finance, invoice}       |
| file-002  | user-88 | profile.jpg   | jpg       | https://cdn.example.com/user/profile.jpg                | 140     | 2025-03-28 12:05:23   | {avatar, profile}        |
| file-003  | user-42 | tax_doc.pdf   | pdf       | https://s3.amazonaws.com/bucket/tax_doc_2025.pdf        | 520     | 2025-03-28 13:15:10   | {finance, taxes, upload} |

---

  4. Checks **notification rules table** (optional feature)
     - e.g., “If file type = .pdf → send SMS”
  5. If a match, adds a new message to a **notification queue** for processing

 #### Lambda can poll (but only behind the scenes)
 
<img width="530" alt="Screenshot 2025-03-28 at 6 48 59 PM" src="https://github.com/user-attachments/assets/834df220-d9c2-4905-afe4-a6ac6bb6e09e" />


  #### Long polling example

  ```javascript
  app.get('/api/notifications/longpoll', async (req, res) => {
  const userId = req.query.userId;
  const since = parseInt(req.query.since);

  const startTime = Date.now();
  const timeout = 30000;         // max wait time (30s)
  const checkInterval = 1000;    // check every 1s

  const checkForData = async () => {
    const newNotifications = await getNewNotifications(userId, since);

    if (newNotifications.length > 0) {
      // 🟢 🎯 NEW DATA FOUND → respond immediately
      return res.json(newNotifications);
    }

    if (Date.now() - startTime > timeout) {
      // 🟡 Timeout hit → respond with nothing
      return res.json([]);
    }

    // 🔁 Wait 1s and check again
    setTimeout(checkForData, checkInterval);
  };

  // 🚀 Start the recursive wait loop
  checkForData();
  });
  ```

### 3.5 Rules table

<img width="669" alt="Screenshot 2025-03-28 at 6 39 15 PM" src="https://github.com/user-attachments/assets/03ea49d0-f617-497c-9269-35b0c2714299" />

🧾 `NotificationRules` Table (Real Layout)

| id        | user_id | event_type  | file_type | file_name_regex     | min_size_kb | notify_via | target         | is_active |
|-----------|---------|-------------|-----------|----------------------|--------------|-------------|----------------|-----------|
| rule-1    | user-42 | file_upload | pdf       | `^invoice.*\.pdf$`   | 100          | sms         | +1234567890     | true      |
| rule-2    | user-42 | file_upload | jpg       | `null`               | null         | email       | user@mail.com   | true      |
| rule-3    | user-88 | file_upload | null      | `.*\.zip$`           | 5000         | push        | user-88-token   | true      |

🔹 **Each row is one rule**.  
🔹 **Each column defines a piece of that rule’s logic**.

No — `min_size_kb` would **NOT** prevent the file upload.
It only affects whether a notification gets triggered after the upload.

---

### 📩 4. **Notification System**
- Separate ECS worker or Lambda polls **notification queue**
- For each message:
  - Sends SMS/email using **Twilio, SES**, etc.
  - Tracks status via webhook callbacks (optional)
  - Stores log in the DB

Webhook data can either be stored in a notitifcation logs table:

<img width="768" alt="Screenshot 2025-03-28 at 6 45 38 PM" src="https://github.com/user-attachments/assets/19395e83-dbf9-453b-9454-af89e35e45ee" />

Or we can store it in the metadata table of the image

---

### 🔍 5. **Search Service (Lambda or ECS microservice)**
- Called by the frontend
- Queries the **file metadata DB**
- Filter by filename, tags, upload date, user
- Optionally generates **pre-signed S3 URLs** so the frontend can show/download the file

---

### 🧠 6. **Database Schema (example)**

**📄 FileMetadata**
| Field         | Type          |
|---------------|---------------|
| uuid          | UUID (PK)     |
| fileName      | String        |
| s3Url         | String        |
| uploadedAt    | Timestamp     |
| userId        | String        |
| tags          | Array<String> |

**📣 NotificationRules**
| Field         | Type          |
|---------------|---------------|
| userId        | String        |
| fileType      | String        |
| notifyVia     | String (SMS/Email) |
| criteria      | JSON or fields |

**📬 NotificationLog**
| Field         | Type          |
|---------------|---------------|
| uuid          | UUID          |
| status        | String        |
| sentAt        | Timestamp     |

---

## 📈 Scaling Components

| Component          | Scaling Method                       |
|--------------------|--------------------------------------|
| React Frontend     | ECS with Load Balancer (ALB)         |
| API Gateway        | Serverless, autoscaling              |
| S3                 | Scales automatically                 |
| SQS                | Scales with volume                   |
| ECS Workers        | Auto-scaled based on queue depth     |
| Search Lambda      | Serverless, on-demand                |

---

## Architechture

```
┌────────────────────────────┐
│       👤 User (Frontend)   │
└────────────┬───────────────┘
             │
             ▼
┌──────────────────────────────────────────────┐
│ React App (ECS Fargate / ALB / CloudFront)   │
│ - Requests pre-signed S3 URL                 │
│ - Uploads file directly to S3                │
│ - Sends metadata to API Gateway              │
└────────────┬─────────────────────────────────┘
             │
             ▼
┌───────────────────────────────┐
│     🌐 API Gateway            │
│ - Validates request           │
│ - Forwards to Lambda or ECS   │
└────────────┬──────────────────┘
             │
             ▼
┌────────────────────────────────────┐
│ Lambda / ECS Upload Handler        │
│ - Enqueues file metadata to SQS    │
└────────────────┬───────────────────┘
                 │
                 ▼
        ┌──────────────────┐
        │  📬 SQS Queue     │
        └──────┬────────────┘
               │ (polling)
               ▼
      ┌────────────────────────────┐
      │ ECS Worker Service         │
      │ - Dequeues metadata        │
      │ - Stores in DB (FileMetadata)   │
      │ - Checks NotificationRules │
      │ - If match: push to notifQ │
      └────────────┬───────────────┘
                   │
                   ▼
           ┌──────────────────────┐
           │  📨 Notification SQS │
           └────────┬─────────────┘
                    │ (polling)
                    ▼
          ┌────────────────────────────┐
          │ ECS Notification Worker    │
          │ - Sends SMS/Email via API  │
          │ - Stores status in DB      │
          │ - Twilio statusCallback →  │
          │     /twilio/status webhook │
          └────────────┬───────────────┘
                       │
                       ▼
            ┌──────────────────────┐
            │ Twilio Sends Webhook │
            │ (status update)      │
            └────────┬─────────────┘
                     ▼
        ┌────────────────────────────┐
        │ /twilio/status endpoint    │
        │ - Updates NotificationLog  │
        └────────────────────────────┘

   ⬇ Optional: NotificationLog Table
┌────────────────────────────────────────────┐
│ NotificationLog                            │
│ - message_sid                              │
│ - status (queued/sent/delivered/failed)    │
│ - sentAt / updatedAt                       │
│ - linked file_id or user_id                │
└────────────────────────────────────────────┘

   ⬇ File Metadata Table
┌────────────────────────────────────────────┐
│ FileMetadata                               │
│ - file_id, file_name, s3_url               │
│ - uploadedAt, user_id, tags                │
└────────────────────────────────────────────┘

               ▲
               │
      ┌───────────────────────┐
      │ 🔍 Search Service     │
      │ Lambda or ECS        │
      │ - Queries DB         │
      │ - Returns file list  │
      │ - Optionally adds    │
      │   pre-signed S3 URLs │
      └───────────────────────┘
```


