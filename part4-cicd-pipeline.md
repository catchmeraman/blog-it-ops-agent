# Part 4: CI/CD Pipeline for Continuous Agent Deployment

---

## Why CI/CD for AI Agents?

Agent code evolves rapidly — new tools, improved prompts, bug fixes in tool logic. Without CI/CD, deploying updates requires manual zip packaging, S3 upload, and runtime update commands. Our pipeline automates the entire flow:

**Code Push → Build & Test → Package → Deploy to AgentCore → Validate**

---

## Pipeline Architecture

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

---

## Step 9: Set Up CodeCommit Repository

```bash
# Create the repository
aws codecommit create-repository \
  --repository-name it-ops-agent \
  --repository-description "IT Operations Agent for AgentCore Runtime"

# Clone and set up
git clone codecommit://it-ops-agent
cd it-ops-agent

# Add your agent code
cp -r ../it-ops-agent/* .
git add . && git commit -m "Initial agent code"
git push origin main
```

---

## Step 10: CodeBuild Project (buildspec.yml)

Create the build specification that handles testing, packaging, and deployment:

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
      - echo "Installing dependencies..."
      - pip install -r requirements.txt -t ./package/
      - pip install pytest boto3

  pre_build:
    commands:
      - echo "Running unit tests..."
      - python -m pytest tests/ -v --tb=short
      - echo "Tests passed ✓"

  build:
    commands:
      - echo "Packaging agent code..."
      - cp -r main.py agent.py tools/ package/
      - cd package && zip -r ../it-ops-agent.zip . && cd ..
      - echo "Package size: $(du -h it-ops-agent.zip | cut -f1)"

  post_build:
    commands:
      - echo "Uploading to S3..."
      - aws s3 cp it-ops-agent.zip s3://${S3_BUCKET}/${S3_KEY}

      - echo "Updating AgentCore Runtime..."
      - |
        aws bedrock-agentcore update-agent-runtime \
          --agent-runtime-id ${RUNTIME_ID} \
          --role-arn ${ROLE_ARN} \
          --code-s3-bucket ${S3_BUCKET} \
          --code-s3-prefix ${S3_KEY} \
          --code-entry-point "main.py" \
          --code-runtime "PYTHON_3_13" \
          --network-mode "PUBLIC"

      - echo "Waiting for runtime to be READY..."
      - |
        for i in $(seq 1 30); do
          STATUS=$(aws bedrock-agentcore get-agent-runtime \
            --agent-runtime-id ${RUNTIME_ID} \
            --query 'runtimeStatus' --output text)
          echo "Attempt $i: Status = $STATUS"
          if [ "$STATUS" = "READY" ]; then
            echo "Runtime is READY ✓"
            break
          fi
          sleep 10
        done

      - echo "Updating endpoint to latest version..."
      - |
        LATEST_VERSION=$(aws bedrock-agentcore get-agent-runtime \
          --agent-runtime-id ${RUNTIME_ID} \
          --query 'agentRuntimeVersion' --output text)
        echo "Deploying version: $LATEST_VERSION"
        aws bedrock-agentcore update-agent-runtime-endpoint \
          --agent-runtime-id ${RUNTIME_ID} \
          --endpoint-name ${ENDPOINT_NAME} \
          --agent-runtime-version ${LATEST_VERSION}

      - echo "Deployment complete ✓"

artifacts:
  files:
    - it-ops-agent.zip
  discard-paths: yes

cache:
  paths:
    - '/root/.cache/pip/**/*'
```

---

## Step 11: Create the CodeBuild Project

```bash
aws codebuild create-project \
  --name "it-ops-agent-build" \
  --source "type=CODECOMMIT,location=https://git-codecommit.us-east-1.amazonaws.com/v1/repos/it-ops-agent,buildspec=buildspec.yml" \
  --artifacts "type=NO_ARTIFACTS" \
  --environment "type=LINUX_CONTAINER,computeType=BUILD_GENERAL1_SMALL,image=aws/codebuild/standard:7.0" \
  --service-role "arn:aws:iam::114805761158:role/codebuild-it-ops-agent-role"
```

### CodeBuild IAM Role Permissions

The CodeBuild service role needs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::event-agent-kb-114805761158/devops-outputs/*"
    },
    {
      "Sid": "AgentCoreDeployment",
      "Effect": "Allow",
      "Action": [
        "bedrock-agentcore:UpdateAgentRuntime",
        "bedrock-agentcore:GetAgentRuntime",
        "bedrock-agentcore:UpdateAgentRuntimeEndpoint",
        "bedrock-agentcore:GetAgentRuntimeEndpoint"
      ],
      "Resource": "arn:aws:bedrock-agentcore:us-east-1:114805761158:runtime/it_ops_agent_v2-Od8Y3L7coD*"
    },
    {
      "Sid": "PassRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::114805761158:role/event-agent-role"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CodeCommitAccess",
      "Effect": "Allow",
      "Action": [
        "codecommit:GitPull",
        "codecommit:GetBranch",
        "codecommit:GetCommit"
      ],
      "Resource": "arn:aws:codecommit:us-east-1:114805761158:it-ops-agent"
    }
  ]
}
```

---

## Step 12: Create CodePipeline

```bash
aws codepipeline create-pipeline --cli-input-json file://pipeline-definition.json
```

### `pipeline-definition.json`

```json
{
  "pipeline": {
    "name": "it-ops-agent-pipeline",
    "roleArn": "arn:aws:iam::114805761158:role/codepipeline-it-ops-agent-role",
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "SourceAction",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeCommit",
              "version": "1"
            },
            "configuration": {
              "RepositoryName": "it-ops-agent",
              "BranchName": "main",
              "PollForSourceChanges": "false"
            },
            "outputArtifacts": [{"name": "SourceOutput"}]
          }
        ]
      },
      {
        "name": "Build-Test-Deploy",
        "actions": [
          {
            "name": "BuildAndDeploy",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "it-ops-agent-build"
            },
            "inputArtifacts": [{"name": "SourceOutput"}],
            "outputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      }
    ]
  }
}
```

### Enable Automatic Triggers with EventBridge

```bash
# Create EventBridge rule to trigger pipeline on code push
aws events put-rule \
  --name "it-ops-agent-code-change" \
  --event-pattern '{
    "source": ["aws.codecommit"],
    "detail-type": ["CodeCommit Repository State Change"],
    "resources": ["arn:aws:codecommit:us-east-1:114805761158:it-ops-agent"],
    "detail": {
      "event": ["referenceCreated", "referenceUpdated"],
      "referenceName": ["main"]
    }
  }'

aws events put-targets \
  --rule "it-ops-agent-code-change" \
  --targets "Id=Pipeline,Arn=arn:aws:codepipeline:us-east-1:114805761158:it-ops-agent-pipeline,RoleArn=arn:aws:iam::114805761158:role/events-codepipeline-role"
```

---

## Deployment Workflow (Day-to-Day)

Once the pipeline is configured, deploying updates is a single `git push`:

```bash
# Make changes to your agent
vim tools/ssm_tools.py  # Add new remediation capability

# Commit and push
git add .
git commit -m "feat: add disk cleanup remediation via SSM"
git push origin main

# Pipeline automatically:
# 1. Detects code change via EventBridge
# 2. Runs tests in CodeBuild
# 3. Packages and uploads to S3
# 4. Updates AgentCore Runtime (creates new version)
# 5. Updates itOpsEndpoint to new version
```

### Blue-Green Deployment Pattern (Advanced)

For zero-downtime deployments with validation:

```bash
# 1. Push code (DEFAULT endpoint gets new version automatically)
# 2. Test on DEFAULT endpoint
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "..." \
  --qualifier "DEFAULT" \
  --payload '{"prompt": "Run diagnostic health check"}'

# 3. If healthy, promote to production endpoint
aws bedrock-agentcore update-agent-runtime-endpoint \
  --agent-runtime-id "it_ops_agent_v2-Od8Y3L7coD" \
  --endpoint-name "itOpsEndpoint" \
  --agent-runtime-version "7"

# 4. If issues, rollback by pointing to previous version
aws bedrock-agentcore update-agent-runtime-endpoint \
  --agent-runtime-id "it_ops_agent_v2-Od8Y3L7coD" \
  --endpoint-name "itOpsEndpoint" \
  --agent-runtime-version "6"
```

---

> **📸 Screenshot 8: CodePipeline Console**
> Navigate to: **CodePipeline Console → Pipelines → it-ops-agent-pipeline**
> Shows: Source → Build-Test-Deploy stages, both green/succeeded
> File: `screenshots/08-codepipeline-overview.png`

> **📸 Screenshot 9: CodeBuild Logs**
> Navigate to: **CodeBuild → Build projects → it-ops-agent-build → Build history → Latest**
> Shows: Build logs with test results, packaging, S3 upload, and AgentCore update
> File: `screenshots/09-codebuild-logs.png`

> **📸 Screenshot 10: EventBridge Rule**
> Navigate to: **EventBridge → Rules → it-ops-agent-code-change**
> Shows: Rule pattern matching CodeCommit push to main, target = CodePipeline
> File: `screenshots/10-eventbridge-trigger.png`

---

*Continue to Part 5: Testing, Validation, and Conclusion →*
