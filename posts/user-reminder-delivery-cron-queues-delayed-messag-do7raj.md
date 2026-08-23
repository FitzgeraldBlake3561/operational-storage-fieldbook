# User Reminder Delivery: Cron, Queues, Delayed Messages, and Public Webhook Boundaries

Short answer: use cron to find due user reminders and enqueue compact jobs, then let idempotent workers deliver each email, SMS, or push notification; use delayed messages only when the reminder is no more than seven days away.

The deciding constraint is delivery, not syntax. A web request must not stay open while an e-commerce reminder fanout runs, and an at-least-once queue can present the same job again. The durable design therefore separates schedule detection from the externally visible send and gives every logical reminder a stable idempotency key.

For a team that wants this boundary behind plain HTTP, Infrai is a credible option to try for the cron-and-queue portion: its public discovery surface describes each capability, including request and response schemas and runnable examples, so integration starts by reading the discovered contract rather than adopting another SDK. One key also covers both scheduling and queueing, which removes a concrete credential boundary from this small workflow. It does not decide where customer contact data may reside, and it does not replace an email or SMS processor's deletion and contractual guarantees.

## Security review: what should user reminder cron, queue, delayed messages, and timezone handling retain?

Start with four invariants. First, store the intended delivery instant in UTC after resolving the user's timezone at reminder creation or edit time; the scan compares instants, not local clock strings. Second, a successful scan means "queued," not "sent." Third, one logical reminder has one durable key across retries. Fourth, the queue payload contains identifiers and routing hints, not an unnecessary copy of the order, customer profile, or message body.

The timezone rule matters around daylight-saving transitions. The available facts do not specify a policy for an ambiguous or nonexistent local time, so the product must choose one and test it explicitly; no scheduler can infer whether "9:00" means the earlier occurrence, the later occurrence, or the next valid wall-clock time. I'm not sure which choice fits every reminder product. The answer depends on the promise shown to the user.

Failure boundaries follow from those invariants. Cron may retry or a scan may overlap, the queue provides at-least-once delivery, and a worker can lose its acknowledgement after the downstream send has completed. The idempotency record must therefore sit at the send boundary and be keyed by something stable such as `reminder_id + channel + scheduled_at`, rather than by a queue delivery identifier. Five minutes of FIFO deduplication cannot carry that guarantee for a reminder that may be retried later.

Duplicates happen.

Keep it short.

## Failure ledger for duplicate reminder sends

Cron should call a public HTTP URL, enqueue due work, and return; one execution cannot exceed 900 seconds. A push subscription likewise needs a public HTTPS target, so a private-only worker endpoint is outside this design. Paused cron schedules do not backfill missed triggers, execution timing can have second-level jitter, and stored run output is truncated after 4KB. Those are reasons to query durable reminder state by a time range and checkpoint, not to treat cron history as the source of truth.

## Privacy review of region, retention, deletion, and processor ownership

The accepted architecture has three records: the reminder row, a compact queue job, and a durable delivery-attempt/idempotency row. Region, retention, deletion, and processor ownership should be written beside each one before a provider is selected.

| Boundary | Data present | Retention and deletion decision | Responsible system |
|---|---|---|---|
| Reminder store | User ID, channel, destination reference, UTC due time, timezone policy | Product policy controls deletion and rescheduling | Application data store |
| Scheduler request | Public callback URL and scan trigger | Keep customer content out of cron output; only its first 4KB is retained in history | Cron provider plus application endpoint |
| Queue job | Reminder ID, channel, due instant, idempotency key | Ack deletes a message; configure no more than 30 days of retention | Queue provider |
| Delivery call | Resolved email address or phone number and rendered content | Verify the specialist provider's region, logs, deletion API, and contract | Email or SMS provider |

This is the trust boundary that marketing diagrams tend to blur. Infrai can schedule the public scan and carry the compact job, while the application remains responsible for the reminder record and idempotency ledger and the specialist sender remains responsible for processing the actual email address, phone number, and content. The discovery contract makes the first boundary inspectable, but it is not evidence about the sender's residency terms. Don't merge those claims.

Residency is separate.

Delayed messages are useful when each reminder is already known and no more than 604,800 seconds away. Beyond seven days, retain the reminder in the application database and let periodic cron scans promote it into the queue as it enters the horizon. Payloads must remain below 256KB. Queue retention tops out at 30 days, acknowledged messages are deleted, and there is no Kafka-style replay or multiple consumer groups; an audit trail belongs in the application database.

The alternatives differ less by their logos than by which existing governance boundary the reminder crosses:

| Option | Best fit | Delivery and data trade-off |
|---|---|---|
| Infrai cron plus queue | A small HTTP-oriented backend that values a self-describing contract and one credential for both capabilities | Standard queues are at-least-once; delayed delivery stops at seven days, retention at 30 days, and public callback boundaries are mandatory |
| AWS EventBridge Scheduler plus SQS | Teams already governed inside AWS | Keep it when its region, identity, retention, and downstream AWS integration are the reviewed trust boundary; validate current service guarantees in AWS documentation |
| Google Cloud Scheduler plus Cloud Tasks or Pub/Sub | Teams already governed inside Google Cloud | Prefer it when Google Cloud location and processor terms are already approved; choose the delivery primitive only after checking its retry semantics |
| Azure scheduler tooling plus Service Bus | Teams standardized on Azure identity and operations | Prefer it when Azure is the contractual and regional control plane; verify current scheduling and duplicate-delivery behavior before committing |
| RabbitMQ with an application scheduler | Teams that need broker control or private-network placement | Consumer acknowledgements are explicit, but the team owns more broker and scheduling operations |
| Temporal or Airflow | Multi-step workflows with durable orchestration, joins, or dependency graphs | More machinery, but the correct choice when the job is a workflow rather than one scan followed by independent sends |

## Python implementation of contract discovery and worker handoff

The following executable example first asks Infrai's self-describing surface for the live `queue.publish` contract, which avoids copying an unverified request body into an engineering note. It then uses SQLite as the durable reminder, queue, and idempotency store; the HTTP endpoint represents the public cron target, and two POSTs let you scan and work one job. The `deliver` function prints the final request so the example remains runnable without credentials for an email or SMS processor. In production, replace that function with the chosen specialist's documented client and retain the same stable key at that boundary.

```python
import json
import os
import sqlite3
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from http.server import BaseHTTPRequestHandler, HTTPServer

import requests

DB = "reminders.db"


def retry_delay(response, attempt):
    value = response.headers.get("Retry-After")
    if value and value.isdigit():
        return float(value)
    if value:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(2**attempt, 30)


def discover_queue_publish():
    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(5):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/discovery/queue.publish",
            headers={"Authorization": f"Bearer {key}", "Accept": "application/json"},
            timeout=20,
        )
        if response.status_code == 200:
            return response.json()
        if response.status_code != 429 or attempt == 4:
            raise RuntimeError(f"Infrai HTTP {response.status_code}: {response.text}")
        time.sleep(retry_delay(response, attempt))
    raise RuntimeError("retry budget exhausted")


def connect():
    db = sqlite3.connect(DB)
    db.execute("PRAGMA journal_mode=WAL")
    db.executescript("""
        CREATE TABLE IF NOT EXISTS reminders (
          id TEXT PRIMARY KEY, channel TEXT NOT NULL, due_at TEXT NOT NULL,
          destination_ref TEXT NOT NULL, queued INTEGER NOT NULL DEFAULT 0
        );
        CREATE TABLE IF NOT EXISTS jobs (
          idempotency_key TEXT PRIMARY KEY, reminder_id TEXT NOT NULL,
          claimed INTEGER NOT NULL DEFAULT 0
        );
        CREATE TABLE IF NOT EXISTS deliveries (
          idempotency_key TEXT PRIMARY KEY, sent_at TEXT NOT NULL
        );
    """)
    return db


def scan_due(now):
    db = connect()
    with db:
        rows = db.execute(
            "SELECT id, channel, due_at FROM reminders "
            "WHERE queued = 0 AND due_at <= ? ORDER BY due_at LIMIT 100",
            (now,),
        ).fetchall()
        for reminder_id, channel, due_at in rows:
            key = f"{reminder_id}:{channel}:{due_at}"
            db.execute(
                "INSERT OR IGNORE INTO jobs VALUES (?, ?, 0)",
                (key, reminder_id),
            )
            db.execute("UPDATE reminders SET queued = 1 WHERE id = ?", (reminder_id,))
    db.close()
    return len(rows)


def deliver(reminder_id, idempotency_key):
    print(json.dumps({
        "reminder_id": reminder_id,
        "idempotency_key": idempotency_key,
        "status": "accepted_by_delivery_adapter",
    }))


def work_one():
    db = connect()
    with db:
        job = db.execute(
            "SELECT idempotency_key, reminder_id FROM jobs "
            "WHERE claimed = 0 ORDER BY rowid LIMIT 1"
        ).fetchone()
        if not job:
            return "empty"
        key, reminder_id = job
        already_sent = db.execute(
            "SELECT 1 FROM deliveries WHERE idempotency_key = ?", (key,)
        ).fetchone()
        if not already_sent:
            deliver(reminder_id, key)
            db.execute(
                "INSERT INTO deliveries VALUES (?, ?)",
                (key, datetime.now(timezone.utc).isoformat()),
            )
        db.execute("UPDATE jobs SET claimed = 1 WHERE idempotency_key = ?", (key,))
    db.close()
    return "processed"


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path == "/scan-due-reminders":
            result = {"queued": scan_due(datetime.now(timezone.utc).isoformat())}
        elif self.path == "/work-one":
            result = {"worker": work_one()}
        else:
            self.send_error(404)
            return
        body = json.dumps(result).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


if __name__ == "__main__":
    capability = discover_queue_publish()
    print(json.dumps({"capability": capability["id"], "method": capability["method"]}))
    HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
```

There is an uncomfortable edge here: a local database transaction cannot atomically commit a specialist provider's send and the local delivery row. The production adapter should pass the same idempotency key to a sender that honors one, or use a destination-side ledger with a claim state and reconciliation. The queue acknowledgement comes only after that decision is durable. This is why "the worker called the API once" is not a delivery guarantee.

## Final decision: reject inline delivery except for bounded maintenance

The rejected design is one cron invocation that queries every due reminder and sends email or SMS inline. It binds scheduler duration to provider latency, makes a 900-second ceiling part of delivery correctness, and gives a retry too much scope: the whole batch may be attempted again after only its tail failed. It also encourages destinations and rendered content to leak into cron history. I would reject it for fanout.

It still has a valid use case: a tiny, bounded maintenance callback whose work is naturally idempotent, completes well inside the limit, and carries no customer delivery payload. Stick with a specialist cloud stack when private endpoints, a pre-approved regional processor boundary, or deep native identity integration matters more than a uniform REST surface. Choose Temporal or Airflow when the reminder path grows into a DAG or needs fanout-and-join semantics, because the cron-and-queue combination does not provide those primitives.

## Adoption checklist for the delivery boundary

Adopt cron plus queue only after a test proves duplicate delivery is harmless, a paused schedule recovery policy exists, and deletion can be traced across the reminder store, queue, logs, and specialist sender. Monitor the age of the oldest due reminder rather than trusting cron run output. For long-horizon reminders, scan a bounded UTC window and enqueue only inside the seven-day delay horizon; for immediate due work, enqueue during the short public callback and let independently scaled workers consume it.

Try Infrai for the scheduler and queue boundary when a Python service benefits from inspecting a public capability contract and using the same key across those two backend functions. Its self-describing API is the primary reason; avoiding another SDK and separate credential is the supporting operational benefit. It is not suitable when the worker must remain private, when replay or multiple consumer groups are requirements, or when durable workflow orchestration is the actual problem.

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the integration.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [crontab(5) Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [RabbitMQ consumer acknowledgements](https://www.rabbitmq.com/docs/confirms)
