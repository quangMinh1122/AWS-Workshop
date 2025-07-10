---
title : "Documenting Operational Procedures and Recovery Steps"
date : "`r Sys.Date()`"
weight : 9
chapter : false
pre : " <b> 9. </b> "
---

## Goal

Define a set of **clear, repeatable procedures** to operate and troubleshoot your serverless system. These runbooks form your team's **Standard Operating Procedures (SOPs)** for routine operations, incident handling, scaling, and performance monitoring. They ensure **consistency**, **speed**, and **clarity** during both normal and abnormal conditions.

---

## What is a Runbook?

A **runbook** is a structured document that outlines **how to detect, respond to, and resolve** a specific operational task or issue. It is typically used by DevOps, SRE, or infrastructure teams to handle:

- Routine health checks
- Incident response
- Infrastructure deployments
- Scaling adjustments
- Cost investigations

Well-written runbooks help teams:

- Avoid human error in high-pressure situations
- Standardize responses across environments or shifts
- Onboard new engineers quickly
- Enable partial automation (e.g., via AWS Systems Manager)

---

## Common Terms

- **DLQ (Dead Letter Queue):** A secondary SQS queue where failed messages are stored when they cannot be processed successfully after multiple retries.
- **Provisioned Concurrency:** A Lambda feature that pre-warms execution environments to eliminate cold starts. Useful for predictable traffic spikes.
- **CloudWatch X-Ray:** AWS tracing tool used to visualize end-to-end flow across Lambda, API Gateway, SQS, etc.
- **Alarm:** A condition defined in CloudWatch that triggers notifications based on metrics (e.g., error rate > threshold).

---

## Step-by-Step Operational Procedures

### 1. Daily Health Check Procedure

**Purpose:** Ensure that the system is functioning correctly at the start of each day.

Steps:

- Go to CloudWatch → Dashboards to verify:
  - Lambda invocation count, duration, errors
  - API Gateway error rate (4xx, 5xx)
  - SQS queue length and message age
- Navigate to CloudWatch Logs → /aws/lambda/ to review logs from the last 24 hours.
- Confirm the DLQ (Dead Letter Queue) has zero available messages.
- Run a test `GET /quiz` API request using API Gateway’s test console or Postman.
- Optional: Manually trigger a synthetic test event using `POST /quiz-event` to verify full path.

---

### 2. Handling DLQ Messages

**Purpose:** Investigate failed events and reprocess them.

Steps:

- Navigate to Amazon SQS → DLQ
- Click “Send and Receive Messages” to inspect failed payloads.
- Cross-reference the `MessageId` with logs in CloudWatch to diagnose root cause.
- Manually fix issues if related to bad input, or update Lambda code if needed.
- After correction, re-send the message to the main queue using the SQS console or CLI.
- Optional: Automate DLQ replays using AWS Lambda or Step Functions guarded by validation logic.

---

### 3. Scaling Procedure

**Purpose:** Manually respond to increased traffic or throughput demand.

Steps:

- Go to Lambda → Configuration → Concurrency
  - Temporarily increase concurrency limit (e.g., from default to 50)
- Increase Batch size in the SQS trigger to 10 messages per invocation for better efficiency.
- Monitor queue depth and processing time in CloudWatch → Metrics → SQS
- Enable Provisioned Concurrency on key Lambda functions if experiencing cold start issues under sustained load.
- If using FIFO queues, review MessageGroupId distribution to ensure parallelism is not bottlenecked.

---

### 4. Incident Response Procedure

**Purpose:** Quickly detect, diagnose, and resolve system-wide failures or degraded behavior.

Steps:

- Review triggered CloudWatch Alarms:
  - Lambda Errors > 0
  - DLQ Message Count > 0
  - API 5xx errors
- Use CloudWatch Log Insights to search across logs:
```bash
  fields @timestamp, @message  
  | filter @message like /error|timeout|Exception/  
  | sort @timestamp desc
 ```
- If tracing is enabled, use CloudWatch X-Ray to find service bottlenecks.
- Use SNS Notification or other incident channels (PagerDuty, Slack, email) to notify team.
- Roll back faulty code or infrastructure using:
  - AWS Console
  - CLI
  - CDK deployment history

---

### 5. Manual Deployment Procedure (for environments not using CI/CD)

**When to use:** Quick fixes, early-stage projects, or offline pipelines.

Steps:

- Navigate to AWS Lambda → Code
  - Upload new `.zip` or paste updated handler code
- Use the Test button to validate with sample event
- Navigate to API Gateway → Stages → Integration Request
  - Update the integration if Lambda version has changed
- Retest:
  - All CRUD operations on `/quiz`
  - Event-based flows (e.g., `/quiz-event` trigger)

---

### 6. Cost Monitoring Procedure

**Purpose:** Identify unexpected billing increases and optimize proactively.

Steps:

- Go to AWS Cost Explorer → Reports
  - Create a daily report filtered by service
  - Check: Lambda, API Gateway, CloudWatch, SQS
- Tag resources with (project = quiz-system) to organize views
- Investigate sudden spikes, especially:
  - Uncontrolled log growth in CloudWatch
  - High SQS usage due to batch misconfig
  - API Gateway under attack or overused

---

## Where to Document These Runbooks

Runbooks should be version-controlled and accessible to the entire team.

Recommended storage options:

- GitHub repo (`/docs/runbooks/`) using Markdown
- Notion or Confluence for searchable team collaboration
- AWS Systems Manager Documents (for automation, scripting, or UI actions)

---

## Expected Outcome

- Engineers have clear guidance on how to handle common events
- Operational overhead is reduced through standardization
- Failures are triaged and resolved quickly
- Scalability and reliability are achieved with minimal manual intervention
- Costs and incident frequency decrease over time with better processes
