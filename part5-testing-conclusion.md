# Part 5: Console Screenshots Guide, Testing, and Conclusion

---

## Console Screenshots Reference Guide

To accompany this blog post with real implementation evidence, capture the following screenshots from your AWS Console. Each screenshot demonstrates a key aspect of the deployed agent.

### Screenshot Capture Checklist

| # | Console Location | What to Capture | Filename |
|---|-----------------|-----------------|----------|
| 1 | Bedrock Console → AgentCore → Runtimes | Runtime list showing `it_ops_agent_v2` with READY status | `01-agentcore-runtime-dashboard.png` |
| 2 | AgentCore → Runtimes → it_ops_agent_v2 → Configuration | Network mode, timeouts, workload identity ARN | `02-runtime-configuration.png` |
| 3 | AgentCore → Runtimes → it_ops_agent_v2 → Endpoints | Both DEFAULT and itOpsEndpoint showing READY/v6 | `03-endpoints-tab.png` |
| 4 | IAM Console → Roles → event-agent-role → Permissions | Inline policies (8 total) and managed policies | `04-iam-role-permissions.png` |
| 5 | IAM Console → Roles → event-agent-role → Trust | Trust relationship with bedrock-agentcore.amazonaws.com | `05-iam-trust-relationships.png` |
| 6 | AgentCore → Runtimes → it_ops_agent_v2 → Versions | All 6 versions with timestamps | `06-version-history.png` |
| 7 | AgentCore → Endpoints → itOpsEndpoint details | Endpoint ARN, live version, created/updated dates | `07-endpoint-details.png` |
| 8 | CodePipeline → it-ops-agent-pipeline | Pipeline stages all green | `08-codepipeline-overview.png` |
| 9 | CodeBuild → it-ops-agent-build → Latest build | Build logs showing successful deployment | `09-codebuild-logs.png` |
| 10 | EventBridge → Rules → it-ops-agent-code-change | Rule pattern and CodePipeline target | `10-eventbridge-trigger.png` |
| 11 | S3 → event-agent-kb-114805761158 → devops-outputs/ | The uploaded it-ops-agent.zip artifact | `11-s3-artifact.png` |
| 12 | Bedrock → Knowledge Bases → (runbook KB) | Knowledge base used by the agent for runbook queries | `12-bedrock-knowledge-base.png` |

### Recommended Architecture Diagram (for Blog Header)

Generate a polished architecture diagram using one of these approaches:

1. **AWS Architecture Center icons** — Use draw.io with AWS icon pack
2. **Diagrams-as-code** — Use the `diagrams` Python library:

```python
from diagrams import Diagram, Cluster, Edge
from diagrams.aws.ml import Sagemaker
from diagrams.aws.management import Cloudwatch, SystemsManager
from diagrams.aws.compute import EC2
from diagrams.aws.integration import SNS, Eventbridge
from diagrams.aws.devtools import Codebuild, Codepipeline, Codecommit
from diagrams.aws.storage import S3

with Diagram("IT Ops Agent on AgentCore", show=False, direction="LR"):
    user = EC2("User/Trigger")

    with Cluster("Amazon Bedrock AgentCore"):
        runtime = Sagemaker("it_ops_agent_v2\n(Runtime)")

    with Cluster("Diagnostic Tools"):
        cw = Cloudwatch("CloudWatch\nMetrics & Logs")
        ct = Cloudwatch("CloudTrail\nEvents")

    with Cluster("Remediation Tools"):
        ssm = SystemsManager("SSM\nRun Command")
        ec2 = EC2("EC2\nManagement")
        sns = SNS("SNS\nAlerts")

    with Cluster("CI/CD Pipeline"):
        cc = Codecommit("CodeCommit")
        cb = Codebuild("CodeBuild")
        s3 = S3("S3 Artifact")
        eb = Eventbridge("EventBridge")

    user >> runtime
    runtime >> cw
    runtime >> ct
    runtime >> ssm
    runtime >> ec2
    runtime >> sns

    cc >> Edge(label="trigger") >> eb >> Codepipeline("Pipeline") >> cb >> s3
```

---

## Testing the Agent

### Test Scenario 1: CloudWatch Alarm Diagnosis

```bash
# Invoke the agent with a diagnostic question
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD" \
  --qualifier "itOpsEndpoint" \
  --payload '{"prompt": "Are there any CloudWatch alarms currently in ALARM state? If so, investigate the root cause."}' \
  --runtime-session-id "test-diag-001"
```

**Expected Agent Behavior:**
1. Calls `get_alarms` tool to list active alarms
2. For each alarm, queries `get_metric_data` to understand the trend
3. Searches CloudTrail for recent changes that may have caused the issue
4. Provides root cause analysis and remediation recommendation

### Test Scenario 2: Instance Remediation

```bash
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD" \
  --qualifier "itOpsEndpoint" \
  --payload '{"prompt": "Instance i-0abc123def has high memory usage (95%). Check what process is consuming memory and restart the application service."}' \
  --runtime-session-id "test-remediate-001"
```

**Expected Agent Behavior:**
1. Runs SSM command: `ps aux --sort=-%mem | head -20`
2. Identifies the problematic process
3. Runs SSM command to restart the service: `systemctl restart <service>`
4. Verifies the fix by checking memory again
5. Sends SNS notification to the ops team

### Test Scenario 3: Multi-Turn Conversation

```python
import boto3, json

client = boto3.client("bedrock-agentcore", region_name="us-east-1")
session = "multi-turn-test-001"
runtime_arn = "arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD"

# Turn 1: Initial question
r1 = client.invoke_agent_runtime(
    agentRuntimeArn=runtime_arn,
    qualifier="itOpsEndpoint",
    payload=json.dumps({"prompt": "Show me all EC2 instances with status check failures"}),
    runtimeSessionId=session
)
print(json.loads(r1["body"].read())["response"])

# Turn 2: Follow-up remediation (same session!)
r2 = client.invoke_agent_runtime(
    agentRuntimeArn=runtime_arn,
    qualifier="itOpsEndpoint",
    payload=json.dumps({"prompt": "Reboot the first instance that failed and notify the team via SNS"}),
    runtimeSessionId=session
)
print(json.loads(r2["body"].read())["response"])
```

---

## Operational Metrics & Monitoring

### Key Metrics to Track

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Agent response latency | CloudWatch (AgentCore metrics) | > 30 seconds |
| Session idle timeouts | CloudWatch Logs | > 10 per hour |
| Tool execution failures | Application logs | Any |
| Runtime restarts | AgentCore events | > 2 per day |

### Setting Up Monitoring

```bash
# Create a CloudWatch alarm for agent errors
aws cloudwatch put-metric-alarm \
  --alarm-name "it-ops-agent-errors" \
  --namespace "AgentCore/CustomMetrics" \
  --metric-name "AgentErrors" \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:us-east-1:114805761158:ops-alerts"
```

---

## Lessons Learned

### 1. Session Management Matters
The 15-minute idle timeout (900s) works well for incident response. Longer incidents should use periodic keep-alive pings.

### 2. Immutable Versions Are Your Safety Net
With 6 versions deployed in a single day during initial development, the ability to instantly rollback was invaluable.

### 3. Tool Design is Critical
Keep tools atomic and well-scoped. A tool that does "check metrics AND restart" is harder to debug than two separate tools.

### 4. System Prompt Engineering
The system prompt acts as your agent's SOP. Iterate on it heavily — our v3→v4 update was primarily prompt improvements.

### 5. Start with Read-Only, Add Write Later
Our version progression shows this pattern: v1-v3 were diagnostics-only, v4-v6 added remediation capabilities.

---

## Cost Considerations

| Component | Estimated Monthly Cost |
|-----------|----------------------|
| AgentCore Runtime (sessions) | Based on session duration — no cost when idle |
| Bedrock Claude 3.5 Sonnet v2 | ~$3/1M input tokens, ~$15/1M output tokens |
| S3 storage (artifacts) | < $1 |
| CodeBuild (CI/CD) | ~$5 (10 builds/month × 5 min) |
| CloudWatch Logs | Based on volume |

> **💡 Key insight**: AgentCore Runtime is session-based — you only pay for active sessions. The 900s idle timeout auto-terminates unused sessions, keeping costs low.

---

## Conclusion

We've built a production-grade IT Operations Agent that:

✅ **Diagnoses** infrastructure issues using CloudWatch, CloudTrail, and application logs
✅ **Remediates** common problems via SSM commands and EC2 management
✅ **Evolves** through immutable versions with instant rollback capability
✅ **Deploys automatically** through a CI/CD pipeline triggered by code pushes
✅ **Scales** with AgentCore Runtime's session-based compute model

### What's Next

- **EventBridge integration**: Automatically trigger the agent when CloudWatch alarms fire
- **Multi-agent orchestration**: Connect with `observability_rca_agent` for deeper root cause analysis
- **Approval workflows**: Add human-in-the-loop for destructive remediation actions
- **Memory integration**: Use AgentCore Memory for long-term incident knowledge

---

## Resources

- [Amazon Bedrock AgentCore Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [Strands Agents Framework](https://github.com/strands-agents/strands-agents)
- [AWS Systems Manager Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html)
- [AgentCore Runtime CLI Reference](https://docs.aws.amazon.com/cli/latest/reference/bedrock-agentcore/)

---

*Published by the AIOps Engineering Team | Built on Amazon Bedrock AgentCore Runtime*
