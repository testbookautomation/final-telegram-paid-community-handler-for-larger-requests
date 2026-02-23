# Telegram Invite Service (Cloud Tasks + Firestore + WebEngage)

Scalable Telegram single-use invite link generator with:

* ✅ **Asynchronous processing via Cloud Tasks**
* ✅ **Retry with exponential backoff (Telegram rate-limit aware)**
* ✅ **Firestore persistence**
* ✅ **WebEngage event tracking**
* ✅ **Telegram webhook for join tracking**
* ✅ Designed for production (GCP Cloud Run / GKE / Compute)

---

## 🏗 Architecture Overview

```
Client → /v1/invite/request
            ↓
        Firestore (QUEUED)
            ↓
      Cloud Tasks enqueue
            ↓
       /v1/invite/worker
            ↓
  Telegram createChatInviteLink
            ↓
    Save invite hash lookup
            ↓
  Fire WebEngage "link_created"
            ↓
User joins Telegram
            ↓
Telegram Webhook
            ↓
 Fire WebEngage "joined"
```

---

## 📁 Firestore Collections

### 1️⃣ `invite_requests`

Tracks request lifecycle.

| Field            | Type                                | Description          |
| ---------------- | ----------------------------------- | -------------------- |
| requestId        | string                              | UUID                 |
| userId           | string                              | Internal user ID     |
| transactionId    | string                              | Payment reference    |
| status           | QUEUED / PROCESSING / DONE / FAILED |                      |
| attempts         | number                              | Retry count          |
| inviteLink       | string                              | Telegram invite link |
| weLinkEventFired | boolean                             | Link event sent      |
| joinEventFired   | boolean                             | Join event sent      |
| telegramUserId   | string                              | Telegram user ID     |
| createdAt        | ISO string                          |                      |
| updatedAt        | ISO string                          |                      |

---

### 2️⃣ `invite_lookup`

Maps invite hash → request.

| Field         | Type       |
| ------------- | ---------- |
| inviteLink    | string     |
| requestId     | string     |
| userId        | string     |
| transactionId | string     |
| createdAt     | ISO string |

Invite link is stored by:

```
sha256(inviteLink)
```

---

# 🚀 API Endpoints

---

## 1️⃣ Create Invite Request

### `POST /v1/invite/request`

### Body

```json
{
  "userId": "12345",
  "transactionId": "txn_abc"
}
```

### Response

```json
{
  "ok": true,
  "status": "queued",
  "requestId": "uuid"
}
```

This:

* Saves request
* Enqueues Cloud Task
* Returns immediately (<100ms)

---

## 2️⃣ Worker Endpoint (Cloud Tasks Only)

### `POST /v1/invite/worker`

⚠️ Secured by Cloud Tasks header:

```
x-cloudtasks-queuename
```

Flow:

* Increment attempts
* Create Telegram invite link
* If rate limited → reschedule using retry_after
* Save invite hash lookup
* Fire WebEngage event:

  * `pass_paid_community_telegram_link_created`

---

## 3️⃣ Telegram Webhook

### `POST /v1/telegram/webhook`

Triggered on user join.

Flow:

* Extract invite link
* Hash lookup in Firestore
* Fire WebEngage event:

  * `pass_paid_community_telegram_joined`

---

# 🌎 Environment Variables

```bash
# Telegram
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHANNEL_ID=

# WebEngage
WEBENGAGE_LICENSE_CODE=
WEBENGAGE_API_KEY=

# GCP
GCP_PROJECT=
GCP_LOCATION=asia-south1
TASKS_QUEUE=tg-invite-queue

# Service
BASE_URL=https://your-service-url
PORT=8080
```

---

# ⚙️ Setup Instructions

---

## 1️⃣ Enable APIs

```bash
gcloud services enable \
  cloudtasks.googleapis.com \
  firestore.googleapis.com
```

---

## 2️⃣ Create Cloud Tasks Queue

```bash
gcloud tasks queues create tg-invite-queue \
  --location=asia-south1
```

---

## 3️⃣ Deploy to Cloud Run

```bash
gcloud run deploy telegram-invite-service \
  --source . \
  --region asia-south1 \
  --allow-unauthenticated
```

---

## 4️⃣ Set Telegram Webhook

```bash
curl -X POST \
  https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/setWebhook \
  -H "Content-Type: application/json" \
  -d '{"url":"https://your-service-url/v1/telegram/webhook"}'
```

---

# 🔁 Retry Logic

* `MAX_ATTEMPTS = 50`
* If Telegram returns:

  ```json
  {
    "parameters": {
      "retry_after": 30
    }
  }
  ```
* Worker reschedules task after `retry_after` seconds.

---

# 🔐 Security Recommendations (Production)

* Require OIDC authentication from Cloud Tasks.
* Restrict webhook endpoint by secret path or IP validation.
* Enable Firestore IAM restricted access.
* Rotate WebEngage & Telegram API tokens regularly.

---

# 📊 Events Sent to WebEngage

### 1️⃣ Link Created

```
pass_paid_community_telegram_link_created
```

Payload:

```json
{
  "transactionId": "...",
  "inviteLink": "..."
}
```

---

### 2️⃣ User Joined

```
pass_paid_community_telegram_joined
```

Payload:

```json
{
  "transactionId": "...",
  "inviteLink": "...",
  "telegramUserId": "..."
}
```

---

# 🧠 Design Decisions

* **Hashing invite links** avoids Firestore key length issues.
* **Cloud Tasks** prevents blocking API response.
* **Idempotent join handling** prevents duplicate WebEngage events.
* **Single-use invites (`member_limit: 1`)** ensure private access.

---

# 🛠 Local Development

```bash
npm install
node index.js
```

Use `.env` with `dotenv` if needed.

---

# 📈 Scaling Notes

* Cloud Run auto scales workers.
* Cloud Tasks ensures backpressure.
* Firestore handles high write throughput.
* Suitable for high payment volume systems.

---

# 🧯 Failure Scenarios

| Scenario              | Behaviour                |
| --------------------- | ------------------------ |
| Telegram rate limit   | Delayed retry            |
| Telegram API error    | Retry                    |
| Max attempts exceeded | Mark FAILED              |
| WebEngage failure     | Event flag remains false |

---

# 📦 Suggested Improvements

* Add structured logging (Winston / Pino)
* Add OpenTelemetry tracing
* Add request signature verification for Telegram
* Add Dead Letter Queue (DLQ)
* Add metrics via Cloud Monitoring

---

# 🏁 Status Lifecycle

```
QUEUED → PROCESSING → DONE
                    ↘ FAILED
```

---

# 📄 License

MIT

---

Just tell me what you want next 🚀
