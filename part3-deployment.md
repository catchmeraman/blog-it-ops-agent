# Part 3: Deploying to AgentCore Runtime

---

## Step 4: Package Your Agent Code

AgentCore Runtime supports direct code deployment via S3. Package your agent as a zip file:

```bash
# Install dependencies into the package
pip install -r requirements.txt -t ./package/

# Copy your agent code
cp -r main.py agent.py tools/ package/

# Create the deployment zip
cd package && zip -r ../it-ops-agent.zip . && cd ..

# Upload to S3
aws s3 cp it-ops-agent.zip s3://event-agent-kb-114805761158/devops-outputs/it-ops-agent.zip
```

### `requirements.txt`

```txt
strands-agents>=0.1.0
boto3>=1.34.0
```

---

## Step 5: Create the AgentCore Runtime

Use the AWS CLI or SDK to create the runtime. Here's the CLI approach:

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

### Key Configuration Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `network-mode` | PUBLIC | Simplifies connectivity, secured via IAM |
| `idle-timeout` | 900s (15 min) | Keeps sessions alive for multi-turn diagnostics |
| `max-lifetime` | 28800s (8 hrs) | Supports long-running incident response |
| `server-protocol` | HTTP | Standard HTTP request/response pattern |
| `code-runtime` | PYTHON_3_13 | Latest supported Python runtime |

### What Happens Behind the Scenes

When you create a runtime, AgentCore:

1. **Provisions** a dedicated Firecracker microVM environment
2. **Installs** your code package and dependencies
3. **Starts** your HTTP server on port 8080
4. **Creates** a DEFAULT endpoint automatically
5. **Assigns** a workload identity for secure credential management

The runtime transitions through states: `CREATING` → `READY`

---

## Step 6: Create a Custom Endpoint

While the DEFAULT endpoint is created automatically, we create a named endpoint for better routing and management:

```bash
aws bedrock-agentcore create-agent-runtime-endpoint \
  --agent-runtime-id "it_ops_agent_v2-Od8Y3L7coD" \
  --name "itOpsEndpoint" \
  --description "Production endpoint for IT Operations agent"
```

### Our Endpoint Configuration

| Endpoint | Version | Status | Purpose |
|----------|---------|--------|---------|
| DEFAULT | v6 | READY | Auto-created, always latest |
| itOpsEndpoint | v6 | READY | Named production endpoint |

> **💡 Pro Tip**: Use named endpoints for production traffic and the DEFAULT endpoint for testing. This lets you deploy a new version, test it on DEFAULT, then update the named endpoint once validated.

---

## Step 7: Updating the Runtime (New Version)

Each update creates a new **immutable version**. This is how our agent evolved from v1 to v6:

```bash
# Update with new code
aws bedrock-agentcore update-agent-runtime \
  --agent-runtime-id "it_ops_agent_v2-Od8Y3L7coD" \
  --role-arn "arn:aws:iam::114805761158:role/event-agent-role" \
  --code-s3-bucket "event-agent-kb-114805761158" \
  --code-s3-prefix "devops-outputs/it-ops-agent.zip" \
  --code-entry-point "main.py" \
  --code-runtime "PYTHON_3_13"
```

### Version History (Real Deployment Timeline)

| Version | Deployed | Changes |
|---------|----------|---------|
| v1 | Mar 31, 12:48 UTC | Initial deployment - basic diagnostics |
| v2 | Mar 31, 12:58 UTC | Added SSM run command tools |
| v3 | Mar 31, 14:30 UTC | Added EC2 management capabilities |
| v4 | Mar 31, 17:06 UTC | Added Knowledge Base integration |
| v5 | Mar 31, 17:20 UTC | Improved error handling |
| v6 | Mar 31, 18:15 UTC | Added SNS alerting, EKS support |

### Rollback Strategy

If a new version has issues, update the endpoint to point to a previous version:

```bash
# Rollback itOpsEndpoint to version 5
aws bedrock-agentcore update-agent-runtime-endpoint \
  --agent-runtime-id "it_ops_agent_v2-Od8Y3L7coD" \
  --endpoint-name "itOpsEndpoint" \
  --agent-runtime-version "5"
```

---

## Step 8: Invoking the Agent

### Via AWS CLI

```bash
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD" \
  --qualifier "itOpsEndpoint" \
  --payload '{"prompt": "Check if there are any CloudWatch alarms in ALARM state and diagnose the root cause"}' \
  --runtime-session-id "incident-$(date +%s)"
```

### Via Python SDK (boto3)

```python
import boto3
import json

client = boto3.client("bedrock-agentcore", region_name="us-east-1")

response = client.invoke_agent_runtime(
    agentRuntimeArn="arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD",
    qualifier="itOpsEndpoint",
    payload=json.dumps({
        "prompt": "Instance i-0abc123 has high CPU. Diagnose and remediate."
    }),
    runtimeSessionId="incident-session-001"
)

result = json.loads(response["body"].read())
print(result["response"])
```

---

> **📸 Screenshot 6: Runtime Versions Page**
> Navigate to: **AgentCore → Runtimes → it_ops_agent_v2 → Versions**
> Shows: Version history v1 through v6 with timestamps
> File: `screenshots/06-version-history.png`

> **📸 Screenshot 7: Endpoint Details**
> Navigate to: **AgentCore → Runtimes → it_ops_agent_v2 → Endpoints → itOpsEndpoint**
> Shows: Endpoint status READY, pointing to v6
> File: `screenshots/07-endpoint-details.png`

---

*Continue to Part 4: CI/CD Pipeline Setup →*
