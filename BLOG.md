# Building an Intelligent IT Operations Agent on Amazon Bedrock AgentCore Runtime

## A 300-Level Deep Dive into Production-Grade AIOps with Automated Remediation

---

> **Level**: 300 (Advanced) | **Services**: Amazon Bedrock AgentCore, CloudWatch, SSM, EC2, CodePipeline, CodeBuild  
> **Time to Build**: 4-6 hours | **Region**: us-east-1

---

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [IAM Role Setup](#step-1-create-the-iam-execution-role)
5. [Agent Code Implementation](#step-3-agent-code-implementation)
6. [Deploying to AgentCore Runtime](#step-4-package-your-agent-code)
7. [CI/CD Pipeline](#part-4-cicd-pipeline-for-continuous-agent-deployment)
8. [Testing & Validation](#testing-the-agent)
9. [Operational Monitoring](#operational-metrics--monitoring)
10. [Lessons Learned & Conclusion](#lessons-learned)

---

## Introduction

Modern IT operations teams face an ever-growing challenge: infrastructure complexity scales exponentially while team size remains constant. Traditional runbook-based approaches cannot keep pace with the volume and velocity of operational incidents across hybrid cloud environments.

In this blog post, we walk through how we built **it_ops_agent_v2** — a production-grade IT Operations Agent deployed on **Amazon Bedrock AgentCore Runtime** that can:

- **Diagnose** infrastructure issues using CloudWatch metrics, logs, and CloudTrail events
- **Remediate** common problems via AWS Systems Manager (SSM) run commands
- **Manage** EC2 instances (reboot, stop/start, security group modifications)
- **Alert** teams through SNS notifications
- **Query** an internal knowledge base for runbook guidance using Amazon Bedrock Knowledge Bases

This agent has been running in production since March 2026, currently at **version 6**, serving requests through a custom endpoint with a fully automated CI/CD pipeline for updates.

---

## Architecture Overview

![IT Ops Agent Architecture](generated-diagrams/it-ops-agent-architecture.png)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        IT Ops Agent Architecture                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────┐     ┌─────────────────────────────────────┐       │
│  │   User /     │     │   Amazon Bedrock AgentCore Runtime   │       │
│  │   Event      │────▶│                                     │       │
│  │   Trigger    │     │   ┌─────────────────────────────┐   │       │
│  └──────────────┘     │   │     it_ops_agent_v2         │   │       │
│                        │   │   (Strands Agent Framework)  │   │       │
│                        │   │                             │   │       │
│                        │   │   ┌─────────┐ ┌─────────┐  │   │       │
│                        │   │   │ Tools   │ │  LLM    │  │   │       │
│                        │   │   └────┬────┘ └────┬────┘  │   │       │
│                        │   └────────┼───────────┼────────┘   │       │
│                        └────────────┼───────────┼────────────┘       │
│                                     │           │                     │
│  ┌──────────────────────────────────┼───────────┼──────────────────┐ │
│  │              AWS Services        │           │                  │ │
│  │                                  ▼           ▼                  │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │ │
│  │  │CloudWatch│  │CloudTrail│  │   SSM    │  │ Bedrock FM   │   │ │
│  │  │Metrics & │  │  Events  │  │Run Cmds  │  │Claude 3.5    │   │ │
│  │  │  Logs    │  │          │  │          │  │Sonnet v2     │   │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │ │
│  │                                                                 │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │ │
│  │  │   EC2    │  │   SNS    │  │   EKS    │  │ Bedrock KB   │   │ │
│  │  │ Manage   │  │ Alerts   │  │ Cluster  │  │ (Runbooks)   │   │ │
│  │  │          │  │          │  │  Info    │  │              │   │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                    CI/CD Pipeline                                │ │
│  │  CodeCommit ──▶ CodeBuild ──▶ S3 Upload ──▶ AgentCore Update    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Architecture Decisions

1. **AgentCore Runtime over Lambda**: We chose AgentCore Runtime for its session persistence (up to 8 hours), eliminating cold starts for multi-turn diagnostic conversations.
2. **Strands Agent Framework**: Provides a clean tool-use pattern with Claude 3.5 Sonnet v2 as the reasoning engine.
3. **Public Network Mode**: The agent operates in PUBLIC network mode for simplicity, with IAM-based access control securing the endpoint.
4. **Immutable Versioning**: Each deployment creates a new immutable version (currently v6), enabling instant rollbacks.

> 📸 **Screenshot Placeholder**: AgentCore Runtime Dashboard  
> ![AgentCore Runtime Dashboard](screenshots/01-agentcore-runtime-dashboard.png)  
> *Navigate to: Amazon Bedrock Console → AgentCore → Runtimes*

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| AWS Account | With access to Amazon Bedrock (Claude 3.5 Sonnet v2 enabled) |
| AgentCore Runtime | Access to Amazon Bedrock AgentCore service |
| IAM Role | With trust policy for `bedrock-agentcore.amazonaws.com` |
| Python 3.12+ | For the agent code |
| AWS CLI v2 | Configured with appropriate credentials |
| S3 Bucket | For storing agent code packages |
| Bedrock Knowledge Base | (Optional) For runbook retrieval |

### Service Quotas to Verify

- Bedrock AgentCore Runtime: Default limit of 10 runtimes per account
- Bedrock model access: Claude 3.5 Sonnet v2 must be enabled in your region
- SSM: Ensure target EC2 instances have SSM Agent installed and running

---

## Step 1: Create the IAM Execution Role

The AgentCore Runtime needs an IAM role that it can assume to execute your agent's operations.

### Trust Policy (`trust-policy.json`)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "bedrock-agentcore.amazonaws.com",
          "bedrock.amazonaws.com",
          "ec2.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name it-ops-agent-role \
  --assume-role-policy-document file://trust-policy.json \
  --description "Execution role for IT Ops Agent on AgentCore Runtime"
```

> 📸 **Screenshot Placeholder**: IAM Trust Relationships  
> ![IAM Trust Relationships](screenshots/05-iam-trust-relationships.png)  
> *Navigate to: IAM Console → Roles → event-agent-role → Trust relationships*

---

## Step 2: Attach Permission Policies

### Policy 1: Bedrock Model & Knowledge Base Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BedrockModelAccess",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*"
    },
    {
      "Sid": "KnowledgeBaseAccess",
      "Effect": "Allow",
      "Action": ["bedrock:Retrieve", "bedrock:RetrieveAndGenerate"],
      "Resource": "arn:aws:bedrock:us-east-1:<ACCOUNT_ID>:knowledge-base/<KB_ID>"
    },
    {
      "Sid": "AgentCoreAccess",
      "Effect": "Allow",
      "Action": "bedrock-agentcore:*",
      "Resource": "*"
    }
  ]
}
```

### Policy 2: CloudWatch & CloudTrail for Diagnostics

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchReadForRCA",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:DescribeAlarms", "cloudwatch:GetMetricStatistics",
        "cloudwatch:GetMetricData", "cloudwatch:ListMetrics"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogsForRCA",
      "Effect": "Allow",
      "Action": [
        "logs:FilterLogEvents", "logs:GetLogEvents",
        "logs:DescribeLogGroups", "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudTrailForRCA",
      "Effect": "Allow",
      "Action": ["cloudtrail:LookupEvents", "cloudtrail:GetTrailStatus"],
      "Resource": "*"
    }
  ]
}
```

### Policy 3: SSM & EC2 for Remediation

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMFull",
      "Effect": "Allow",
      "Action": [
        "ssm:SendCommand", "ssm:GetCommandInvocation",
        "ssm:ListCommandInvocations", "ssm:DescribeInstanceInformation",
        "ssm:ListCommands", "ssm:CancelCommand"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EC2Ops",
      "Effect": "Allow",
      "Action": [
        "ec2:RebootInstances", "ec2:StartInstances", "ec2:StopInstances",
        "ec2:DescribeInstances", "ec2:DescribeInstanceStatus",
        "ec2:ModifyInstanceAttribute", "ec2:CreateTags"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SNSOps",
      "Effect": "Allow",
      "Action": ["sns:Publish", "sns:ListTopics", "sns:CreateTopic", "sns:Subscribe"],
      "Resource": "*"
    }
  ]
}
```

> 📸 **Screenshot Placeholder**: IAM Role Permissions  
> ![IAM Role Permissions](screenshots/04-iam-role-permissions.png)  
> *Navigate to: IAM Console → Roles → event-agent-role → Permissions*

---

## Step 3: Agent Code Implementation

### Project Structure

```
it-ops-agent/
├── main.py                  # Agent entry point (HTTP server)
├── agent.py                 # Agent definition and system prompt
├── tools/
│   ├── __init__.py
│   ├── cloudwatch_tools.py  # CloudWatch metrics & alarms
│   ├── ssm_tools.py         # Systems Manager run commands
│   ├── ec2_tools.py         # EC2 instance management
│   ├── log_tools.py         # CloudWatch Logs analysis
│   ├── sns_tools.py         # SNS notifications
│   └── kb_tools.py          # Knowledge Base retrieval
├── requirements.txt
└── buildspec.yml            # CI/CD build specification
```

### `main.py` — HTTP Server for AgentCore Runtime

```python
"""
IT Ops Agent - Main entry point for AgentCore Runtime.
Exposes an HTTP endpoint that AgentCore routes requests to.
"""
import json
import logging
from http.server import HTTPServer, BaseHTTPRequestHandler
from agent import create_it_ops_agent

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

agent = create_it_ops_agent()


class AgentHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(content_length)
        try:
            request = json.loads(body)
            prompt = request.get("prompt", "")
            session_id = request.get("session_id", "default")
            logger.info(f"Received request - session: {session_id}")
            response = agent(prompt)
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            result = {"response": str(response), "session_id": session_id, "status": "success"}
            self.wfile.write(json.dumps(result).encode())
        except Exception as e:
            logger.error(f"Agent error: {e}", exc_info=True)
            self.send_response(500)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps({"error": str(e), "status": "error"}).encode())

    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps({"status": "healthy"}).encode())


if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", 8080), AgentHandler)
    logger.info("IT Ops Agent starting on port 8080")
    server.serve_forever()
```

### `agent.py` — Agent Definition with System Prompt

```python
from strands import Agent
from strands.models import BedrockModel
from tools.cloudwatch_tools import get_alarms, get_metric_data
from tools.ssm_tools import run_ssm_command, check_ssm_status
from tools.ec2_tools import manage_instance, describe_instances
from tools.log_tools import search_logs, get_recent_errors
from tools.sns_tools import send_notification
from tools.kb_tools import query_runbook

SYSTEM_PROMPT = """You are an expert IT Operations Agent responsible for 
monitoring, diagnosing, and remediating infrastructure issues on AWS.

## Your Capabilities:
1. **Diagnostics**: Query CloudWatch metrics/alarms, search logs, 
   check CloudTrail for recent changes
2. **Remediation**: Execute SSM commands on instances, reboot/stop/start 
   EC2 instances, modify security groups
3. **Alerting**: Send SNS notifications to operations teams
4. **Knowledge**: Query internal runbooks for standard operating procedures

## Operating Principles:
- Always DIAGNOSE before REMEDIATING
- Explain your reasoning and findings clearly
- For destructive actions (reboot, stop), confirm the action and explain why
- Log all remediation actions taken
- Escalate to human operators if the issue is outside your capability
"""


def create_it_ops_agent() -> Agent:
    model = BedrockModel(
        model_id="anthropic.claude-3-5-sonnet-20241022-v2:0",
        region_name="us-east-1"
    )
    return Agent(
        model=model,
        system_prompt=SYSTEM_PROMPT,
        tools=[
            get_alarms, get_metric_data, run_ssm_command, check_ssm_status,
            manage_instance, describe_instances, search_logs, get_recent_errors,
            send_notification, query_runbook,
        ]
    )
```

---

## Step 4: Package Your Agent Code

```bash
pip install -r requirements.txt -t ./package/
cp -r main.py agent.py tools/ package/
cd package && zip -r ../it-ops-agent.zip . && cd ..
aws s3 cp it-ops-agent.zip s3://event-agent-kb-114805761158/devops-outputs/it-ops-agent.zip
```

---

## Step 5: Create the AgentCore Runtime

```bash
aws bedrock-agentcore create-agent-runtime \
  --agent-runtime-name "it_ops_agent_v2" \
  --role-arn "arn:aws:iam::114805761158:role/event-agent-role" \
  --network-mode "PUBLIC" \
  --code-s3-bucket "event-agent-kb-114805761158" \
  --code-s3-prefix "devops-outputs/it-ops-agent.zip" \
  --code-entry-point "main.py" \
  --code-runtime "PYTHON_3_13" \
  --server-protocol "HTTP" \
  --region us-east-1
```

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `network-mode` | PUBLIC | Simplifies connectivity, secured via IAM |
| `idle-timeout` | 900s (15 min) | Keeps sessions alive for multi-turn diagnostics |
| `max-lifetime` | 28800s (8 hrs) | Supports long-running incident response |
| `server-protocol` | HTTP | Standard request/response pattern |
| `code-runtime` | PYTHON_3_13 | Latest supported Python runtime |

> 📸 **Screenshot Placeholder**: Runtime Configuration  
> ![Runtime Configuration](screenshots/02-runtime-configuration.png)  
> *Navigate to: AgentCore → Runtimes → it_ops_agent_v2 → Configuration*

---

## Step 6: Create a Custom Endpoint

```bash
aws bedrock-agentcore create-agent-runtime-endpoint \
  --agent-runtime-id "it_ops_agent_v2-Od8Y3L7coD" \
  --name "itOpsEndpoint" \
  --description "Production endpoint for IT Operations agent"
```

| Endpoint | Version | Status | Purpose |
|----------|---------|--------|---------|
| DEFAULT | v6 | READY | Auto-created, always latest |
| itOpsEndpoint | v6 | READY | Named production endpoint |

> 💡 **Pro Tip**: Use named endpoints for production traffic and the DEFAULT endpoint for testing.

> 📸 **Screenshot Placeholder**: Endpoints Tab  
> ![Endpoints Tab](screenshots/03-endpoints-tab.png)  
> *Navigate to: AgentCore → Runtimes → it_ops_agent_v2 → Endpoints*

---

## Step 7: Version History & Rollback

Each update creates a new **immutable version**:

| Version | Deployed | Changes |
|---------|----------|---------|
| v1 | Mar 31, 12:48 UTC | Initial deployment - basic diagnostics |
| v2 | Mar 31, 12:58 UTC | Added SSM run command tools |
| v3 | Mar 31, 14:30 UTC | Added EC2 management capabilities |
| v4 | Mar 31, 17:06 UTC | Added Knowledge Base integration |
| v5 | Mar 31, 17:20 UTC | Improved error handling |
| v6 | Mar 31, 18:15 UTC | Added SNS alerting, EKS support |

### Rollback Command

```bash
aws bedrock-agentcore update-agent-runtime-endpoint \
  --agent-runtime-id "it_ops_agent_v2-Od8Y3L7coD" \
  --endpoint-name "itOpsEndpoint" \
  --agent-runtime-version "5"
```

> 📸 **Screenshot Placeholder**: Version History  
> ![Version History](screenshots/06-version-history.png)  
> *Navigate to: AgentCore → Runtimes → it_ops_agent_v2 → Versions*

---

## Part 4: CI/CD Pipeline for Continuous Agent Deployment

### Pipeline Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  CodeCommit  │────▶│  CodeBuild   │────▶│  S3 Upload   │────▶│  AgentCore   │
│  (Source)    │     │  (Build &    │     │  (Artifact)  │     │  (Deploy)    │
│              │     │   Test)      │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │                     │
       │ git push           │ pip install        │ it-ops-agent.zip   │ update-runtime
       │ to main            │ pytest             │                     │ update-endpoint
       ▼                    ▼                    ▼                     ▼
  Triggers           Runs unit tests       Stores versioned       Creates new
  pipeline           Packages code         artifact in S3         immutable version
```

### `buildspec.yml`

```yaml
version: 0.2

env:
  variables:
    RUNTIME_ID: "it_ops_agent_v2-Od8Y3L7coD"
    S3_BUCKET: "event-agent-kb-114805761158"
    S3_KEY: "devops-outputs/it-ops-agent.zip"
    ROLE_ARN: "arn:aws:iam::114805761158:role/event-agent-role"
    ENDPOINT_NAME: "itOpsEndpoint"

phases:
  install:
    runtime-versions:
      python: 3.13
    commands:
      - pip install -r requirements.txt -t ./package/
      - pip install pytest boto3

  pre_build:
    commands:
      - echo "Running unit tests..."
      - python -m pytest tests/ -v --tb=short

  build:
    commands:
      - cp -r main.py agent.py tools/ package/
      - cd package && zip -r ../it-ops-agent.zip . && cd ..

  post_build:
    commands:
      - aws s3 cp it-ops-agent.zip s3://${S3_BUCKET}/${S3_KEY}
      - |
        aws bedrock-agentcore update-agent-runtime \
          --agent-runtime-id ${RUNTIME_ID} \
          --role-arn ${ROLE_ARN} \
          --code-s3-bucket ${S3_BUCKET} \
          --code-s3-prefix ${S3_KEY} \
          --code-entry-point "main.py" \
          --code-runtime "PYTHON_3_13" \
          --network-mode "PUBLIC"
      - |
        for i in $(seq 1 30); do
          STATUS=$(aws bedrock-agentcore get-agent-runtime \
            --agent-runtime-id ${RUNTIME_ID} \
            --query 'runtimeStatus' --output text)
          if [ "$STATUS" = "READY" ]; then break; fi
          sleep 10
        done
      - |
        LATEST_VERSION=$(aws bedrock-agentcore get-agent-runtime \
          --agent-runtime-id ${RUNTIME_ID} \
          --query 'agentRuntimeVersion' --output text)
        aws bedrock-agentcore update-agent-runtime-endpoint \
          --agent-runtime-id ${RUNTIME_ID} \
          --endpoint-name ${ENDPOINT_NAME} \
          --agent-runtime-version ${LATEST_VERSION}
      - echo "Deployment complete ✓"
```

### CodePipeline Definition

```json
{
  "pipeline": {
    "name": "it-ops-agent-pipeline",
    "roleArn": "arn:aws:iam::114805761158:role/codepipeline-it-ops-agent-role",
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "SourceAction",
          "actionTypeId": {"category": "Source", "owner": "AWS", "provider": "CodeCommit", "version": "1"},
          "configuration": {"RepositoryName": "it-ops-agent", "BranchName": "main"},
          "outputArtifacts": [{"name": "SourceOutput"}]
        }]
      },
      {
        "name": "Build-Test-Deploy",
        "actions": [{
          "name": "BuildAndDeploy",
          "actionTypeId": {"category": "Build", "owner": "AWS", "provider": "CodeBuild", "version": "1"},
          "configuration": {"ProjectName": "it-ops-agent-build"},
          "inputArtifacts": [{"name": "SourceOutput"}]
        }]
      }
    ]
  }
}
```

### EventBridge Auto-Trigger

```bash
aws events put-rule \
  --name "it-ops-agent-code-change" \
  --event-pattern '{
    "source": ["aws.codecommit"],
    "detail-type": ["CodeCommit Repository State Change"],
    "detail": {"event": ["referenceUpdated"], "referenceName": ["main"]}
  }'
```

### Day-to-Day Deployment Workflow

```bash
# Make changes → commit → push → done!
git add . && git commit -m "feat: add disk cleanup tool" && git push origin main
# Pipeline: detect → test → package → deploy → update endpoint (automatic)
```

> 📸 **Screenshot Placeholder**: CodePipeline Overview  
> ![CodePipeline](screenshots/08-codepipeline-overview.png)  
> *Navigate to: CodePipeline → Pipelines → it-ops-agent-pipeline*

> 📸 **Screenshot Placeholder**: CodeBuild Logs  
> ![CodeBuild Logs](screenshots/09-codebuild-logs.png)  
> *Navigate to: CodeBuild → it-ops-agent-build → Latest build*

---

## Testing the Agent

### Test Scenario 1: CloudWatch Alarm Diagnosis

```bash
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD" \
  --qualifier "itOpsEndpoint" \
  --payload '{"prompt": "Are there any CloudWatch alarms in ALARM state? Investigate root cause."}' \
  --runtime-session-id "test-diag-001"
```

### Test Scenario 2: Instance Remediation

```bash
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD" \
  --qualifier "itOpsEndpoint" \
  --payload '{"prompt": "Instance i-0abc123def has 95% memory. Find the process and restart it."}' \
  --runtime-session-id "test-remediate-001"
```

### Test Scenario 3: Multi-Turn Conversation

```python
import boto3, json

client = boto3.client("bedrock-agentcore", region_name="us-east-1")
session = "multi-turn-001"
arn = "arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD"

# Turn 1
r1 = client.invoke_agent_runtime(agentRuntimeArn=arn, qualifier="itOpsEndpoint",
    payload=json.dumps({"prompt": "Show EC2 instances with status check failures"}),
    runtimeSessionId=session)

# Turn 2 (same session — agent remembers context)
r2 = client.invoke_agent_runtime(agentRuntimeArn=arn, qualifier="itOpsEndpoint",
    payload=json.dumps({"prompt": "Reboot the first failed instance and notify team via SNS"}),
    runtimeSessionId=session)
```

---

## Operational Metrics & Monitoring

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Agent response latency | CloudWatch | > 30 seconds |
| Session idle timeouts | CloudWatch Logs | > 10 per hour |
| Tool execution failures | Application logs | Any |
| Runtime restarts | AgentCore events | > 2 per day |

---

## Lessons Learned

1. **Session Management Matters** — The 15-minute idle timeout works well for incident response.
2. **Immutable Versions Are Your Safety Net** — 6 versions in one day; instant rollback saved us.
3. **Tool Design is Critical** — Keep tools atomic. Don't combine "check + fix" into one tool.
4. **System Prompt Engineering** — The prompt IS your agent's SOP. Iterate heavily.
5. **Start Read-Only, Add Write Later** — v1-v3 were diagnostics-only, v4-v6 added remediation.

---

## Cost Considerations

| Component | Estimated Monthly Cost |
|-----------|----------------------|
| AgentCore Runtime (sessions) | Pay per session duration — free when idle |
| Bedrock Claude 3.5 Sonnet v2 | ~$3/1M input, ~$15/1M output tokens |
| S3 storage (artifacts) | < $1 |
| CodeBuild (CI/CD) | ~$5 (10 builds/month) |
| CloudWatch Logs | Based on volume |

---

## Conclusion

We've built a production-grade IT Operations Agent that:

✅ **Diagnoses** infrastructure issues using CloudWatch, CloudTrail, and application logs  
✅ **Remediates** common problems via SSM commands and EC2 management  
✅ **Evolves** through immutable versions with instant rollback capability  
✅ **Deploys automatically** through a CI/CD pipeline triggered by code pushes  
✅ **Scales** with AgentCore Runtime's session-based compute model  

### What's Next

- **EventBridge integration**: Auto-trigger agent when CloudWatch alarms fire
- **Multi-agent orchestration**: Connect with `observability_rca_agent` for deeper RCA
- **Approval workflows**: Human-in-the-loop for destructive remediation
- **Memory integration**: AgentCore Memory for long-term incident knowledge

---

## Resources

- [Amazon Bedrock AgentCore Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [Strands Agents Framework](https://github.com/strands-agents/strands-agents)
- [AWS Systems Manager Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html)
- [AgentCore Runtime CLI Reference](https://docs.aws.amazon.com/cli/latest/reference/bedrock-agentcore/)

---

*Published by the AIOps Engineering Team | Built on Amazon Bedrock AgentCore Runtime*
