# Building an Intelligent IT Operations Agent on Amazon Bedrock AgentCore Runtime

## A 300-Level Deep Dive into Production-Grade AIOps with Automated Remediation

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

---

## Prerequisites

Before building this agent, ensure you have:

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

## Console Screenshots Reference

> **📸 Screenshot 1: AgentCore Runtime Dashboard**
> Navigate to: **Amazon Bedrock Console → AgentCore → Runtimes**
> Shows: `it_ops_agent_v2` with status READY, version 6, last updated Mar 31, 2026
> File: `screenshots/01-agentcore-runtime-dashboard.png`

> **📸 Screenshot 2: Runtime Configuration Details**
> Navigate to: **AgentCore → Runtimes → it_ops_agent_v2 → Configuration**
> Shows: Network mode PUBLIC, idle timeout 900s, max lifetime 28800s
> File: `screenshots/02-runtime-configuration.png`

> **📸 Screenshot 3: Endpoints Tab**
> Navigate to: **AgentCore → Runtimes → it_ops_agent_v2 → Endpoints**
> Shows: DEFAULT endpoint (v6, READY) and itOpsEndpoint (v6, READY)
> File: `screenshots/03-endpoints-tab.png`

---

*Continue to Part 2: IAM Role Setup and Agent Code Implementation →*
