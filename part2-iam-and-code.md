# Part 2: IAM Role Setup and Agent Code Implementation

---

## Step 1: Create the IAM Execution Role

The AgentCore Runtime needs an IAM role that it can assume to execute your agent's operations. This role requires a trust policy allowing the `bedrock-agentcore.amazonaws.com` service principal.

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

### Create the Role

```bash
aws iam create-role \
  --role-name it-ops-agent-role \
  --assume-role-policy-document file://trust-policy.json \
  --description "Execution role for IT Ops Agent on AgentCore Runtime"
```

---

## Step 2: Attach Permission Policies

Our agent needs carefully scoped permissions. We use multiple inline policies for separation of concerns:

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
      "Action": [
        "bedrock:Retrieve",
        "bedrock:RetrieveAndGenerate"
      ],
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
        "cloudwatch:DescribeAlarms",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:GetMetricData",
        "cloudwatch:ListMetrics"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogsForRCA",
      "Effect": "Allow",
      "Action": [
        "logs:FilterLogEvents",
        "logs:GetLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudTrailForRCA",
      "Effect": "Allow",
      "Action": [
        "cloudtrail:LookupEvents",
        "cloudtrail:GetTrailStatus"
      ],
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
        "ssm:SendCommand",
        "ssm:GetCommandInvocation",
        "ssm:ListCommandInvocations",
        "ssm:DescribeInstanceInformation",
        "ssm:ListCommands",
        "ssm:CancelCommand"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EC2Ops",
      "Effect": "Allow",
      "Action": [
        "ec2:RebootInstances",
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:ModifyInstanceAttribute",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SNSOps",
      "Effect": "Allow",
      "Action": [
        "sns:Publish",
        "sns:ListTopics",
        "sns:CreateTopic",
        "sns:Subscribe"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchAlarmOps",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricAlarm",
        "cloudwatch:DeleteAlarms",
        "cloudwatch:EnableAlarmActions",
        "cloudwatch:DisableAlarmActions"
      ],
      "Resource": "*"
    }
  ]
}
```

### Policy 4: EKS Read Access (Optional)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EKSAccess",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "eks:ListNodegroups",
        "eks:DescribeNodegroup",
        "eks:AccessKubernetesApi"
      ],
      "Resource": "*"
    }
  ]
}
```

> **📸 Screenshot 4: IAM Role Console**
> Navigate to: **IAM Console → Roles → it-ops-agent-role → Permissions**
> Shows: All inline policies attached to the role
> File: `screenshots/04-iam-role-permissions.png`

> **📸 Screenshot 5: Trust Relationships**
> Navigate to: **IAM Console → Roles → it-ops-agent-role → Trust relationships**
> Shows: bedrock-agentcore.amazonaws.com as trusted entity
> File: `screenshots/05-iam-trust-relationships.png`

---

## Step 3: Agent Code Implementation

Our agent uses the **Strands Agents** framework with custom tools. Here's the complete implementation:

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
└── Dockerfile               # (Optional for container deploy)
```

### `main.py` — HTTP Server for AgentCore Runtime

```python
"""
IT Ops Agent - Main entry point for AgentCore Runtime
Exposes an HTTP endpoint that AgentCore routes requests to.
"""
import json
import logging
from http.server import HTTPServer, BaseHTTPRequestHandler
from agent import create_it_ops_agent

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize agent once at module level
agent = create_it_ops_agent()


class AgentHandler(BaseHTTPRequestHandler):
    """HTTP handler for AgentCore Runtime requests."""

    def do_POST(self):
        """Handle incoming agent invocation requests."""
        content_length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(content_length)

        try:
            request = json.loads(body)
            prompt = request.get("prompt", "")
            session_id = request.get("session_id", "default")

            logger.info(f"Received request - session: {session_id}")

            # Invoke the agent
            response = agent(prompt)

            # Return response
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()

            result = {
                "response": str(response),
                "session_id": session_id,
                "status": "success"
            }
            self.wfile.write(json.dumps(result).encode())

        except Exception as e:
            logger.error(f"Agent error: {e}", exc_info=True)
            self.send_response(500)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            error_result = {"error": str(e), "status": "error"}
            self.wfile.write(json.dumps(error_result).encode())

    def do_GET(self):
        """Health check endpoint."""
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps({"status": "healthy"}).encode())


def main():
    """Start the HTTP server."""
    port = 8080
    server = HTTPServer(("0.0.0.0", port), AgentHandler)
    logger.info(f"IT Ops Agent starting on port {port}")
    server.serve_forever()


if __name__ == "__main__":
    main()
```

### `agent.py` — Agent Definition

```python
"""
IT Ops Agent definition with system prompt and tool registration.
"""
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

## Response Format:
1. Acknowledgment of the issue
2. Diagnostic steps taken and findings
3. Root cause analysis
4. Remediation action (if applicable)
5. Verification that the fix worked
6. Recommendations for prevention
"""


def create_it_ops_agent() -> Agent:
    """Create and configure the IT Ops Agent."""

    model = BedrockModel(
        model_id="anthropic.claude-3-5-sonnet-20241022-v2:0",
        region_name="us-east-1"
    )

    agent = Agent(
        model=model,
        system_prompt=SYSTEM_PROMPT,
        tools=[
            get_alarms,
            get_metric_data,
            run_ssm_command,
            check_ssm_status,
            manage_instance,
            describe_instances,
            search_logs,
            get_recent_errors,
            send_notification,
            query_runbook,
        ]
    )

    return agent
```

---

*Continue to Part 3: Deployment to AgentCore Runtime →*
