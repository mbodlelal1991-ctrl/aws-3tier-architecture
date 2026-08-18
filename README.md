# Highly Available 3-Tier Web Architecture on AWS

A hands-on portfolio project demonstrating core AWS Solutions Architect concepts: VPC networking, high availability, load balancing, auto scaling, and least-privilege security — built entirely within AWS Free Tier constraints.

## Architecture Overview

```
                        Internet
                            │
                    ┌───────▼────────┐
                    │  Internet GW   │
                    └───────┬────────┘
                            │
          ┌─────────────────┴─────────────────┐
          │         Public Subnets            │
          │   10.0.0.0/24    10.0.1.0/24      │
          │   (us-east-1a)   (us-east-1b)     │
          │                                    │
          │    ┌──────────────────────┐       │
          │    │  Application LB       │       │
          │    │  (ALB-SG: 80 open)    │       │
          │    └──────────┬────────────┘       │
          │               │                     │
          │    ┌──────────▼────────────┐       │
          │    │   NAT Gateway          │       │
          │    │  (outbound only)       │       │
          │    └──────────┬────────────┘       │
          └───────────────┼────────────────────┘
                           │
          ┌────────────────┼────────────────────┐
          │         Private App Subnets          │
          │   10.0.10.0/24     10.0.11.0/24     │
          │   (us-east-1a)     (us-east-1b)     │
          │                                       │
          │   ┌─────────┐        ┌─────────┐    │
          │   │  EC2     │        │  EC2     │    │
          │   │ (ASG,    │        │ (ASG,    │    │
          │   │  nginx)  │        │  nginx)  │    │
          │   │ App-SG   │        │ App-SG   │    │
          │   └────┬────┘        └────┬────┘    │
          └────────┼──────────────────┼──────────┘
                    │                  │
          ┌─────────┼──────────────────┼──────────┐
          │         Private DB Subnets            │
          │   10.0.20.0/24     10.0.21.0/24      │
          │   (us-east-1a)     (us-east-1b)      │
          │                                        │
          │        ┌─────────────────┐            │
          │        │  RDS PostgreSQL  │            │
          │        │     DB-SG        │            │
          │        └─────────────────┘            │
          └────────────────────────────────────────┘
```

## Components

| Layer | Service | Configuration |
|---|---|---|
| Network | VPC | `10.0.0.0/16`, 6 subnets across 2 AZs (public / app / db tiers) |
| Compute | EC2 Auto Scaling Group | `t3.micro`, min 1 / desired 2 / max 3, launched via Launch Template |
| Load Balancing | Application Load Balancer | Internet-facing, HTTP:80, spans both public subnets |
| Database | RDS PostgreSQL | `db.t3.micro`, private subnets, single-AZ (cost-optimized for demo) |
| Outbound Access | NAT Gateway | Single-AZ, enables private-subnet package installs (`dnf`) |
| Security | 3-tier Security Group chain | ALB-SG → App-SG → DB-SG, each scoped to reference the SG in front of it rather than open CIDRs |

## Security Design

Traffic can only flow one direction through the stack:

- **ALB-SG**: allows inbound `80` from `0.0.0.0/0` (public entry point)
- **App-SG**: allows inbound `80` **only from ALB-SG** — no direct public access to app instances
- **DB-SG**: allows inbound `5432` **only from App-SG**, with outbound rules removed entirely (RDS never initiates outbound connections)

This means even if an attacker discovered an EC2 instance's private IP, they couldn't reach it — App-SG only trusts traffic that's already passed through the ALB.

## Key Design Decisions

- **No NAT Gateway in the original design** — initially skipped to preserve Free Tier costs. Re-added after discovering private-subnet instances need outbound internet access for OS package installs (`dnf install nginx`) even when running fully private application logic. This is a common real-world gap between "textbook private subnet" and "instances still need to patch/install software."
- **Private DB subnet group spans 2 AZs** even though RDS itself is single-AZ, because AWS requires a subnet group with 2+ AZs regardless of Multi-AZ deployment status.
- **Security groups reference each other by ID, not CIDR** — the standard least-privilege pattern for internal AWS traffic.

## What I'd Add With More Time / Budget

- Multi-AZ RDS for real failover (not Free Tier eligible)
- HTTPS via ACM certificate + Route 53 custom domain
- CI/CD pipeline (GitHub Actions) deploying via Terraform/CloudFormation instead of manual console setup
- CloudWatch alarms + SNS notifications for ASG scaling events and RDS metrics
- S3 + CloudFront for static asset delivery in front of the app tier

## Troubleshooting Log (Real Issues Hit During This Build)

1. **ALB timing out** — root cause was an empty ALB-SG with zero inbound rules; fixed by adding HTTP:80 from `0.0.0.0/0`.
2. **502 Bad Gateway from ALB** — target health checks failing because EC2 instances in private subnets had no outbound internet route, so `dnf install nginx` in the user data script silently failed. Fixed by adding a NAT Gateway + updating the private route table.
3. **Confirmed via target group health status** (`0 healthy / 1 unhealthy`) before and after each fix — a good habit for isolating whether a problem is at the ALB, security group, or instance level.

---
*Built as a portfolio project to demonstrate AWS Solutions Architect Associate (SAA-C03) concepts in a working environment.*
