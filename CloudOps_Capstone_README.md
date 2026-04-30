# AWS Cloud Operations — Capstone Hands-on Lab

Welcome! This repo contains everything you need for the **end-of-course capstone lab** for *Cloud Operations on AWS* (DODEVA). In about **90 minutes** you will deploy, secure, scale, monitor, audit, and tear down a small public-facing web application — touching every one of the 14 course modules in a single connected scenario.

> **Scenario.** You're a new member of Example Corp.'s CloudOps team. Marketing wants the public storefront live by end of day. You will provision the network, deploy the app with infrastructure as code, lock it down, make it highly available and auto-scaling, instrument it with monitoring and audit, and put a budget on it.

---

## Why this lab exists

Across the course we covered concepts module by module — IAM, VPC, CloudFormation, Auto Scaling, CloudWatch, S3, and so on. **This lab is the moment everything clicks together.** You'll see how the pieces compose into a real, working, operationally sound system, instead of 14 disconnected demos.

By the end you will have practiced:

- Sign in, set Region, drive AWS from CloudShell (Mod 01, 03)
- Create IAM users and roles with least-privilege policies (Mod 02)
- Deploy a multi-AZ web application from a single CloudFormation template (Mod 04, 05)
- Connect to instances with Systems Manager Session Manager — no SSH key, no port 22 (Mod 06)
- Configure S3 with versioning and a tiered lifecycle rule (Mod 13)
- Snapshot an EBS volume and discuss AWS Backup (Mod 12)
- Watch a multi-AZ ALB rotate traffic across healthy targets (Mod 07)
- Add a target-tracking Auto Scaling policy and trigger a scale-out (Mod 08)
- Build a CloudWatch dashboard and an alarm (Mod 09)
- Inspect CloudTrail, KMS, and IAM Access Analyzer findings (Mod 10)
- Turn on AWS Config recording and add one managed rule (Mod 03 part 2)
- Use Cost Explorer and create a Budget with email alerts (Mod 14)
- **Clean up everything** — the most underrated operations skill there is

---

## What's in this repo

| File | What it is |
|------|-----------|
| `CloudOps_Capstone_Lab.docx` | The 30-page lab guide. Read this start to finish. Every section maps to a course module and includes console clicks, CLI commands, and instructor notes. |
| `cfn-template.yaml` | The CloudFormation template you'll deploy in Section 5. Creates VPC, ALB, ASG, EC2 with httpd, IAM role, security groups, and the `example-corp:*` tag taxonomy. |
| `cleanup.sh` | Fallback teardown script. Use it if any resource refuses to delete from the console. |
| `README.md` | This file. |

---

## Before you start

You will need:

1. **An AWS account with administrative permissions.** A personal sandbox or your AWS Academy account works. *Do not run this lab against a production account.*
2. **A modern web browser.**
3. **About USD 0.20 of resource charges** — small, but real. Cleanup at the end is mandatory.
4. **Region: `us-east-1` (N. Virginia)** is recommended. Some services like AWS Budgets default there.
5. **No local AWS CLI required** — we use AWS CloudShell from the browser.

> ⚠️ **Cleanup is not optional.** The number-one reason students get a surprise bill is forgetting to delete a NAT gateway, an EIP, or a Config recorder. The lab guide ends with Section 15 — work through it, or run `cleanup.sh`.

---

## How to run the lab

### Option A — instructor-led (the way it's designed)

Your instructor demos each section live. You follow along in your own AWS account, pausing the instructor when you fall behind. The .docx guide is your script — every command and click is in there.

### Option B — self-paced

Open `CloudOps_Capstone_Lab.docx`, open the AWS Console + CloudShell side by side, and work top-to-bottom. Skip the "Instructor note" callouts on the first read.

### Section pacing (90 minutes total)

```
 1. Sign in & set up         5 min   Mod 01
 2. Access management        7 min   Mod 02
 3. System discovery         6 min   Mod 03
 4. Network foundation       8 min   Mod 11
 5. Deploy with IaC         10 min   Mod 04 + 05
 6. Manage with SSM          8 min   Mod 06
 7. Object storage (S3)      6 min   Mod 13
 8. EBS snapshots            4 min   Mod 12
 9. High availability        5 min   Mod 07
10. Auto Scaling             7 min   Mod 08
11. Monitoring               9 min   Mod 09
12. Audit & data security    6 min   Mod 10
13. AWS Config inventory     4 min   Mod 03
14. Cost management          6 min   Mod 14
15. Cleanup                  5 min   —
```

---

## Architecture you'll build

```
                       Internet
                          │
                    ┌─────▼─────┐
                    │    IGW    │
                    └─────┬─────┘
                          │
        ┌─────────────────┴─────────────────┐
        │           ALB SG (80)             │
        │  ┌───────────────────────────┐    │
        │  │ Application Load Balancer │    │
        │  └──────┬──────────────┬─────┘    │
        │         │              │          │
        │  ┌──────▼─────┐ ┌──────▼─────┐    │
        │  │ Subnet A   │ │ Subnet B   │    │
        │  │ (us-east-1a)│ (us-east-1b)│    │
        │  │ ┌────────┐ │ │ ┌────────┐ │    │
        │  │ │  EC2   │ │ │ │  EC2   │ │    │
        │  │ │ httpd  │ │ │ │ httpd  │ │    │
        │  │ └────────┘ │ │ └────────┘ │    │
        │  └────────────┘ └────────────┘    │
        │     Auto Scaling Group (2..4)     │
        │     App SG: port 80 from ALB SG   │
        │            VPC 10.20.0.0/16       │
        └───────────────────────────────────┘
                          │
        CloudWatch ─ CloudTrail ─ AWS Config
        Systems Manager (Session / Patch / Param Store)
        S3 (lifecycle)   EBS Snapshots   IAM
        Budgets / Cost Explorer
```

---

## Quick start (the 30-second version)

If you'd rather see the deployed app first and read later, run this in CloudShell:

```bash
# 1. Get the template
curl -sLO https://raw.githubusercontent.com/sheldton/AWS-CloudOps/main/cfn-template.yaml

# 2. Validate
aws cloudformation validate-template --template-body file://cfn-template.yaml

# 3. Deploy (about 3 minutes)
aws cloudformation deploy \
  --stack-name example-corp-web \
  --template-file cfn-template.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides ProjectName=example-corp-web EnvironmentType=Lab

# 4. Get the URL
aws cloudformation describe-stacks --stack-name example-corp-web \
  --query 'Stacks[0].Outputs[?OutputKey==`WebsiteURL`].OutputValue' --output text
```

…then go back and work through the lab guide for the why and how.

---

## When you're done

```bash
# Pull the cleanup script and run it
curl -sLO https://raw.githubusercontent.com/sheldton/AWS-CloudOps/main/cleanup.sh
bash cleanup.sh
```

Then check the **Billing** dashboard 24 hours later to confirm no resources are still racking up charges.

---

## Troubleshooting

**ALB returns 502 / 503**
Targets are still warming. Wait 60 seconds; check `EC2 → Target Groups → Targets` for "healthy" state.

**`CREATE_FAILED` on the CloudFormation stack**
Almost always a service quota (running EC2 instances) or a name collision with an existing VPC. Look at the **Events** tab — the first failure tells you exactly why.

**Session Manager won't connect**
Give it 60 seconds for the SSM agent to register. If it still fails, check that the instance role has `AmazonSSMManagedInstanceCore` attached.

**S3 bucket creation fails with `BucketAlreadyExists`**
S3 names are globally unique. Add a timestamp or your account ID as a suffix.

**Auto Scaling didn't scale out**
Target tracking takes ~3 minutes to react. Use `aws autoscaling set-desired-capacity` to demonstrate manually.

More in **Appendix C** of the lab guide.

---

## Concepts cheat sheet

| Concept | One-line definition |
|---------|---------------------|
| **Operational Excellence** | The Well-Architected pillar that governs how you prepare, operate, and evolve workloads. |
| **IAM role** | An identity assumed by a service (EC2, Lambda) — no long-lived credentials. |
| **VPC** | Your private logical network in AWS — you control IP ranges, subnets, routing. |
| **Security Group vs NACL** | SG = stateful, attached to ENIs, *allow only*. NACL = stateless, attached to subnets, allow + deny. |
| **CloudFormation stack** | A set of AWS resources created from a single template. Delete the stack, delete the resources. |
| **Session Manager** | Browser-based shell into EC2 with no SSH key and no inbound port 22. |
| **ALB** | Application Load Balancer — Layer 7, routes HTTP/HTTPS, multi-AZ by design. |
| **Auto Scaling Group** | A managed group of EC2 instances that scales in and out based on demand or schedule. |
| **CloudWatch alarm** | A rule that watches a metric and triggers an action when the metric breaches a threshold. |
| **CloudTrail** | The audit log of every API call made in your account. |
| **AWS Config** | The "what does my resource configuration look like over time?" service. |
| **Budget** | A spending threshold with optional email or SNS alerts. |

---

## Feedback

Spot a typo or have a suggestion? Open an issue or PR on this repo. The lab evolves from class to class.

Happy operating! ☁️
