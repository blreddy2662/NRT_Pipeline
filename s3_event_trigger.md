# 🏗️ Architecture Flow

### S3 Landing Zone

A file lands in `s3://landing-zone/...`

**Flow Overview:**
- **S3 Event → EventBridge → SQS FIFO**
  - Configure S3 Event Notification to send `ObjectCreated:*` events.
  - Events are routed into SQS FIFO Queue.
  - Each event uses the filename (or prefix) as `MessageGroupId`.
  - Ensures all events for the same file name are processed in order.

- **SQS FIFO → Lambda Consumer**
  - Lambda is subscribed to SQS FIFO.
  - It picks up one event at a time per `MessageGroupId`.
  - Lambda triggers the Glue Job.

- **Glue Job → Completion Signal**
  - Glue Job runs to completion.
  - When done, Lambda automatically processes the next SQS message for that same `MessageGroupId`.

---

## ⚙️ Detailed Components

### 🔹 1. SQS FIFO Queue Setup

- Create FIFO queue (name must end with `.fifo`).
- Enable `ContentBasedDeduplication = true` (avoids duplicates).
- **Key concepts:**
  - `MessageGroupId`: groups messages by filename → ensures sequential execution for that filename.
  - `MessageDeduplicationId`: ensures duplicate messages aren’t reprocessed.

### 🔹 2. Lambda Function (Consumer)

- Subscribed to SQS FIFO.
- For each message:
  - Parse S3 event.
  - Extract bucket, key, filename.
  - Start Glue Job with boto3:

```python
import boto3, json

glue = boto3.client("glue")

def lambda_handler(event, context):
    for record in event["Records"]:
        body = json.loads(record["body"])
        s3_event = json.loads(body["Message"]) if "Message" in body else body

        bucket = s3_event["Records"][0]["s3"]["bucket"]["name"]
        key = s3_event["Records"][0]["s3"]["object"]["key"]
        filename = key.split("/")[-1]

        print(f"Processing file: {filename} from {bucket}/{key}")

        # Start Glue Job
        response = glue.start_job_run(
            JobName="your-glue-job",
            Arguments={
                "--bucket": bucket,
                "--key": key,
                "--filename": filename
            }
        )
        print(f"Started Glue Job run: {response['JobRunId']}")
```

**Important:** Configure Lambda concurrency = 1 per MessageGroupId (SQS FIFO handles this automatically).

### 🔹 3. Glue Job

- Glue job reads file → processes → writes output.
- Arguments passed from Lambda (`--bucket`, `--key`, `--filename`).
- Glue job runs independently, but since Lambda is invoked per FIFO message, same file’s events won’t overlap.

### 🔹 4. Error Handling

- If Glue job fails, Lambda can:
  - Push event back to SQS with a retry.
  - Or move to DLQ (Dead Letter Queue) after N retries.

### 🔹 5. Monitoring

- Use CloudWatch Logs for Lambda + Glue.
- Use CloudWatch Metrics on SQS:
  - `ApproximateNumberOfMessagesVisible` → backlog monitoring.

---

## 📊 Example Flow

- File `orders_20250915_101500.csv` arrives → S3 event → SQS FIFO (`MessageGroupId=orders_20250915`).
- Lambda picks it up → starts Glue job.
- While Glue job is running, another file `orders_20250915_103000.csv` lands.
- Its event also goes to same `MessageGroupId`.
- SQS FIFO holds it until first Glue job finishes.
- Meanwhile, file `customers_20250915_102000.csv` lands → different `MessageGroupId` → processed in parallel.

---

## ✅ With this design:

- **Same filename** → sequential execution.
- **Different filenames** → parallel execution.
- **No collisions / overwrites.**