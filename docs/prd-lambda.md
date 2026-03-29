# PRD: Lambda Dispatcher

**Service:** `lambda/`
**Language:** Python 3.11
**Runtime:** AWS Lambda
**Trigger:** SQS standard queue
**Region:** us-east-1

---

## Overview

The Lambda dispatcher is a small, stateless function that sits between the Fastify API and ECS Fargate. When the API marks a job complete, it publishes to SQS. Lambda reads SQS, inspects the dump size, and calls `ECS RunTask` with the appropriately-sized Fargate task definition. It also creates the `analyses` record in Postgres.

Lambda was chosen over direct API→Fargate calls to:
1. Decouple the API from Fargate cold starts (~30-60s)
2. Provide automatic retry on transient RunTask failures (SQS visibility timeout)
3. Keep Fargate task sizing logic in one place

---

## Goals

- Read SQS message, determine correct worker size, call ECS RunTask
- Create `analyses` DB record with status = 'running' before launching task
- Pass job metadata to Fargate via environment variables
- Complete in < 1s (excluding ECS RunTask latency ~200ms)

## Non-goals

- Analysis logic (that's the worker)
- Retry of failed analyses (operator triggers a new job from UI)

---

## Module structure

```
lambda/
├── dispatcher.py      Handler + sizing logic
└── requirements.txt   boto3, psycopg2-binary
```

---

## Environment variables

```
DATABASE_URL        Postgres connection string
SQS_QUEUE_URL       Source queue URL (for DLQ config reference)
ECS_CLUSTER         ARN of the ECS cluster
TASK_DEF_SMALL      ARN of worker-small task definition
TASK_DEF_MEDIUM     ARN of worker-medium task definition
TASK_DEF_LARGE      ARN of worker-large task definition
VPC_SUBNETS         Comma-separated subnet IDs (private subnets with NAT)
SECURITY_GROUP      Security group ID for worker tasks
ECR_IMAGE           Full ECR image URI (passed to task override if needed)
AWS_REGION          us-east-1
```

---

## SQS message format

Published by Fastify API on `POST /jobs/:id/complete`:

```json
{
  "job_id":     "uuid",
  "s3_key":     "dumps/org_id/machine_id/job_id.lime",
  "size_bytes": 4294967296
}
```

---

## Implementation

```python
# dispatcher.py
import boto3, json, os
import psycopg2

ecs = boto3.client('ecs', region_name=os.environ['AWS_REGION'])

TASK_DEFS = {
    'small':  os.environ['TASK_DEF_SMALL'],
    'medium': os.environ['TASK_DEF_MEDIUM'],
    'large':  os.environ['TASK_DEF_LARGE'],
}

def select_size(size_bytes: int) -> str:
    gb = size_bytes / (1024 ** 3)
    if gb < 4:    return 'small'
    if gb <= 16:  return 'medium'
    return 'large'

def handler(event, context):
    for record in event['Records']:
        body = json.loads(record['body'])
        job_id     = body['job_id']
        s3_key     = body['s3_key']
        size_bytes = body['size_bytes']

        worker_size = select_size(size_bytes)
        task_def    = TASK_DEFS[worker_size]

        # Create analyses record before launching task
        analysis_id = create_analysis_record(job_id, worker_size)

        ecs.run_task(
            cluster=os.environ['ECS_CLUSTER'],
            taskDefinition=task_def,
            launchType='FARGATE',
            networkConfiguration={
                'awsvpcConfiguration': {
                    'subnets':        os.environ['VPC_SUBNETS'].split(','),
                    'securityGroups': [os.environ['SECURITY_GROUP']],
                    'assignPublicIp': 'DISABLED',
                }
            },
            overrides={
                'containerOverrides': [{
                    'name': 'worker',
                    'environment': [
                        {'name': 'JOB_ID',      'value': job_id},
                        {'name': 'S3_KEY',       'value': s3_key},
                        {'name': 'ANALYSIS_ID',  'value': analysis_id},
                    ]
                }]
            }
        )

def create_analysis_record(job_id: str, worker_size: str) -> str:
    conn = psycopg2.connect(os.environ['DATABASE_URL'])
    cur  = conn.cursor()
    cur.execute(
        """
        INSERT INTO analyses (dump_id, worker_size, status, started_at)
        SELECT d.id, %s, 'running', NOW()
        FROM dumps d WHERE d.job_id = %s
        RETURNING id
        """,
        (worker_size, job_id)
    )
    analysis_id = str(cur.fetchone()[0])
    conn.commit()
    conn.close()
    return analysis_id
```

---

## SQS configuration

- Queue type: Standard (at-least-once delivery)
- Visibility timeout: 300s (5 min — covers Lambda execution + ECS RunTask latency)
- Max receive count: 3 (before DLQ)
- DLQ: separate SQS queue (`memoryinsight-dispatcher-dlq`)
- Lambda trigger: batch size 1 (one job per invocation)

DLQ messages are not automatically retried — operator investigates via CloudWatch and re-triggers from UI if needed.

---

## IAM permissions required

Lambda execution role needs:
- `ecs:RunTask` on the cluster
- `iam:PassRole` for the ECS task execution role
- `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes` on the source queue
- `rds-db:connect` (or VPC access to Postgres)
- `ec2:DescribeSubnets`, `ec2:DescribeSecurityGroups` (for VPC config validation)

---

## Deployment

GitHub Actions deploys on merge to main:
```bash
zip -r lambda.zip dispatcher.py requirements.txt
pip install -r requirements.txt -t package/
cd package && zip -r ../lambda.zip .
aws lambda update-function-code \
  --function-name memoryinsight-dispatcher \
  --zip-file fileb://lambda.zip \
  --region us-east-1
```

---

## Acceptance criteria (Phase 1)

- [ ] SQS message with `size_bytes < 4GB` triggers `worker-small` task definition
- [ ] SQS message with `size_bytes 4-16GB` triggers `worker-medium`
- [ ] SQS message with `size_bytes > 16GB` triggers `worker-large`
- [ ] `analyses` record with `status = 'running'` is created before `RunTask` call
- [ ] If `RunTask` fails (e.g. capacity unavailable): SQS message returns to queue (Lambda throws), DLQ receives after 3 failures
- [ ] Lambda executes in < 2s (p99) excluding ECS RunTask API latency
- [ ] GitHub Actions deploys on merge to main
