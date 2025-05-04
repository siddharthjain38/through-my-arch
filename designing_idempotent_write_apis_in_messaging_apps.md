# 🔁 Designing Idempotent Write APIs: Preventing Duplicates in Messaging Apps

When building messaging apps (or any system that handles write operations), **idempotency** is essential. Network retries, dropped connections, and user impatience can trigger repeated requests, leading to duplicate messages or inconsistent system states. Ensuring **idempotent write APIs** helps mitigate these issues, so repeated requests don’t cause unintended side effects.

In this post, we’ll dive into how to design **idempotent write APIs**, especially for tasks like sending messages, ensuring that repeated requests produce consistent outcomes.

---

## 💡 What Is Idempotency?

An operation is **idempotent** if performing it **multiple times results in the same outcome** as performing it once.

For instance, if a client retries a request to send a message due to a timeout or poor network conditions, the system should **store the message only once**, no matter how many times the request is made.

---

## ✅ Strategy for Idempotent Write APIs

### 1. **Use an Idempotency Key**

Each client should generate a unique **idempotency key** for every write operation (usually a UUID). This key is sent with the request to help the server identify retries.

```http
POST /sendMessage
Headers:
  Idempotency-Key: abc123xyz

Body:
{
  "from": "user123",
  "to": "user456",
  "message": "Hey there!"
}
```

### 2. **Cache the Request Result on the Server**

The backend should check if the idempotency key has been previously seen:

* If it has, return the **same response** as the first request.
* If not, process the request and **store the response** alongside the idempotency key for future reference.

### 3. **Deduplicate Based on Business Logic (Optional)**

For additional safety, you can also deduplicate messages based on:

* Message hash + timestamp
* Sender ID + recipient ID + message content
* A client-generated `message_id`

This step ensures that duplicates are identified even if idempotency keys are missing or inconsistent.

### 4. **Design Your Storage Layer Defensively**

For added reliability:

* Use **unique constraints** on message IDs or deduplication keys.
* Implement logic to return the existing message rather than inserting a duplicate.

This guarantees consistency even when app logic fails to detect a retry.

---

## 🧪 Example (Python Pseudocode)

```python
def send_message(request):
    idempotency_key = request.headers.get('Idempotency-Key')

    if idempotency_key in idempotency_store:
        return idempotency_store[idempotency_key]  # Return cached result

    # Process the message
    message_id = store_message(request.body)

    response = {
        "status": "sent",
        "message_id": message_id
    }

    # Store the result for future retries
    idempotency_store[idempotency_key] = response

    return response
```

---

## ⏲ TTL for Idempotency Keys — Why It Matters

Idempotency keys are crucial for managing retries in systems like messaging apps. However, how long should these keys be stored? The answer lies in **TTL (Time-To-Live)**.

### 🔁 Why Store Idempotency Keys Temporarily?

Most duplicates arise from **network retries, timeouts, or user impatience**, often occurring within **minutes of the original request**. There's no need to store these keys indefinitely—just long enough to handle retries.

**Benefits of TTL:**

* **Reduces memory usage** by expiring old keys.
* **Improves system performance** by keeping the cache lean.
* **Automates cleanup**, so you don’t have to manually manage old keys.

---

### 🕒 How Long Should TTL Be?

TTL duration depends on the nature of the operation:

| Use Case               | Recommended TTL      |
| ---------------------- | -------------------- |
| Messaging / Chat       | 5–15 minutes         |
| File Upload / Sync Ops | 15–60 minutes        |
| Payment / Transaction  | 1–24 hours (or more) |

Set TTL based on how long you expect clients to retry the same request.

---

### ⚙️ Where to Store Keys and TTL

There are a few options for storing idempotency keys:

#### ✅ **1. Cache (Fast Path)**

Use a fast cache like **Redis** to store idempotency keys along with the response.

* **Pros**: Quick read/write operations, built-in TTL support, ideal for short-lived retries (5–30 minutes).

#### ✅ **2. Database with Unique Constraints (Reliable Fallback)**

You can store idempotency keys in a **database** with a **unique constraint** on the `idempotency_key` column. This ensures that duplicate requests are automatically rejected.

* **Pros**: Safeguard against cache misses.
* **How it works**: If the key already exists, the **insert will fail** (due to a unique constraint violation), and you can **catch the exception** to return the existing response.

#### 🧠 Best Practice: **Cache First, DB as Backup**

Use a **cache-first** approach for speed, and rely on the database as a **safety net** to ensure reliability.

---

### 🧱 Example Database Schema

```sql
CREATE TABLE idempotency_keys (
  idempotency_key VARCHAR(64) PRIMARY KEY,
  response_body JSONB,
  created_at TIMESTAMP DEFAULT now()
);
```

Use the `created_at` field to clean up expired keys via background jobs or scheduled tasks.

### 📊 Flow Diagram (Cache + DB Fallback)

```plaintext
+-------------------+
|  Client Sends     |
|  POST /sendMessage|
|  with Idempotency |
|  Key (UUID)       |
+-------------------+
          |
          v
+-----------------------------+
| Check Cache (e.g., Redis)  |
| for Idempotency Key        |
+-----------------------------+
        |       |
   Key Found   Key Not Found
      |             |
Return Cached    Try Inserting
  Response        into DB with
                  Unique Constraint
                    |
                    v
          +----------------------------+
          |  DB Insert Succeeds?       |
          +----------------------------+
                |          |
            Yes          No (Duplicate)
             |               |
     Process Request   Fetch Existing Response
     Store in Cache     from DB
     Return Response    Return Response
```

### 🔚 In Summary

* **TTL** helps manage how long idempotency keys are stored.
* **Cache-first** strategies improve retry performance.
* **Database constraints** ensure safety, even when cache fails.
* You don’t need to pre-check the DB—just handle constraint violations gracefully.

---

## 🏁 Takeaways

* **Idempotency** is vital for systems with retryable write operations.
* Use **idempotency keys** to ensure safe retries.
* Combine with **business-level deduplication** for additional safety.
* Design storage with **constraints** and conflict resolution in mind.

---

Have you faced similar challenges in your messaging or transaction-based systems? What strategies worked best for you? Let's Discuss...