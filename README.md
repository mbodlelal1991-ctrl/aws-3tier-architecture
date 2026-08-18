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
