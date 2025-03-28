# twilio-message-system

## twilio api info

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

Backend API – receives the request, sends to queue (scales with containerization and K8s or ECS, and then loadbalancing like NGINX, ALB, or K8s again)

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

## Architechture
