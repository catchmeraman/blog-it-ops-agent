# IT Ops Agent on Amazon Bedrock AgentCore Runtime

A production-grade IT Operations Agent built with Strands Agents framework, deployed on Amazon Bedrock AgentCore Runtime with automated CI/CD.

## 📖 Blog Post

Read the full 300-level technical blog: **[BLOG.md](./BLOG.md)**

## 🏗️ Architecture

![Architecture Diagram](generated-diagrams/it-ops-agent-architecture.png)

## ✨ Features

- **Diagnose**: CloudWatch metrics/alarms, log analysis, CloudTrail event correlation
- **Remediate**: SSM run commands, EC2 instance management, security group modifications
- **Alert**: SNS notifications to operations teams
- **Learn**: Knowledge Base queries for runbook guidance
- **Deploy**: Automated CI/CD with CodePipeline → CodeBuild → AgentCore Runtime

## 📁 Structure

```
├── BLOG.md                          # Complete blog post
├── generated-diagrams/              # Architecture diagrams
│   └── it-ops-agent-architecture.png
├── screenshots/                     # Console screenshot placeholders
│   └── README.md
├── part1-introduction.md            # Blog Part 1
├── part2-iam-and-code.md            # Blog Part 2
├── part3-deployment.md              # Blog Part 3
├── part4-cicd-pipeline.md           # Blog Part 4
└── part5-testing-conclusion.md      # Blog Part 5
```

## 🚀 Quick Start

```bash
# Clone
git clone https://github.com/<your-username>/blog-it-ops-agent.git

# Read the blog
open BLOG.md
```

## 📸 Screenshots

Add your AWS Console screenshots to the `screenshots/` directory following the naming convention in the blog post.

## License

MIT
