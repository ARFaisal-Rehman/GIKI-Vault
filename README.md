<div align="center">

```
  ██████╗ ██╗██╗  ██╗██╗██╗   ██╗ █████╗ ██╗   ██╗██╗  ████████╗
 ██╔════╝ ██║██║ ██╔╝██║██║   ██║██╔══██╗██║   ██║██║  ╚══██╔══╝
 ██║  ███╗██║█████╔╝ ██║██║   ██║███████║██║   ██║██║     ██║   
 ██║   ██║██║██╔═██╗ ██║╚██╗ ██╔╝██╔══██║██║   ██║██║     ██║   
 ╚██████╔╝██║██║  ██╗██║ ╚████╔╝ ██║  ██║╚██████╔╝███████╗██║   
  ╚═════╝ ╚═╝╚═╝  ╚═╝╚═╝  ╚═══╝  ╚═╝  ╚═╝ ╚═════╝ ╚══════╝╚═╝   
```

### An Intelligent, Event-Driven Campus Cloud Platform on AWS

[![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com)
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://postgresql.org)
[![Flask](https://img.shields.io/badge/Flask-Backend-000000?style=for-the-badge&logo=flask&logoColor=white)](https://flask.palletsprojects.com)
[![CloudFront](https://img.shields.io/badge/CloudFront-CDN-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudfront)

**CE308 — Introduction to Cloud Computing · Fall 2024 · GIKI, Topi**

🌐 **Live Demo:** [https://d3jqcqbhgw2313.cloudfront.net](https://d3jqcqbhgw2313.cloudfront.net)

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [AWS Services](#-aws-services-used)
- [Data Flow](#-data-flow)
- [API Reference](#-api-reference)
- [Database Schema](#-database-schema)
- [Security Model](#-security-model)
- [Monitoring & Alerting](#-monitoring--alerting)
- [Deployment](#-deployment--configuration)
- [Challenges & Solutions](#-challenges--solutions)
- [Team](#-team)

---

## 🌟 Overview

GIKIVault is a **fully serverless, multi-tier cloud platform** that acts as a central intelligence hub for GIKI students and faculty. It automatically ingests live data from **three external public APIs**, processes it through an **event-driven pipeline**, and surfaces results via a **globally distributed web dashboard**.

### What makes GIKIVault different?

| Feature | Basic Assignment | GIKIVault |
|---------|-----------------|-----------|
| AWS Services | 4 (EC2, S3, ELB, IAM) | **14+** (+ Lambda, SQS, RDS, Cognito, CloudFront, API GW, CloudWatch, CloudTrail, SNS, SSM) |
| Data Entry | Manual | **Fully Automated** (EventBridge) |
| External APIs | 1 | **3 live APIs** with decoupled ingestion |
| Authentication | None | **Amazon Cognito** (JWT + role-based) |
| Secret Management | Hardcoded | **SSM Parameter Store** (KMS encrypted) |
| Fault Tolerance | Basic ELB | **ALB + DLQs + Lambda retries** |
| CDN | None | **CloudFront** (global edge) |
| Monitoring | None | **CloudWatch + SNS + CloudTrail** |

---

## 🏗️ Architecture

### High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL WORLD                               │
│                                                                     │
│   ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐    │
│   │ Ticketmaster │  │  OpenWeatherMap   │  │  Campus RSS Feed │    │
│   │     API      │  │       API        │  │                  │    │
│   └──────┬───────┘  └────────┬─────────┘  └────────┬─────────┘    │
└──────────┼──────────────────┼────────────────────────┼─────────────┘
           │                  │                        │
           ▼                  ▼                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS CLOUD (us-east-1)                       │
│                                                                     │
│  ┌────────────────────── INGESTION LAYER ─────────────────────┐    │
│  │                                                             │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │    │
│  │  │ λ ticketmstr │  │  λ weather   │  │  λ rss-campus    │ │    │
│  │  │  -ingestion  │  │   -fetch     │  │     -feed        │ │    │
│  │  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘ │    │
│  │         │   EventBridge triggers (every 15 min)│           │    │
│  └─────────┼──────────────────────────────────────┼───────────┘    │
│            │                                      │                 │
│            ▼                                      ▼                 │
│  ┌──────────────────── MESSAGING LAYER ───────────────────────┐    │
│  │                                                             │    │
│  │  ┌─────────────────────┐    ┌──────────────────────────┐  │    │
│  │  │  SQS: events-queue  │    │   SQS: news-queue        │  │    │
│  │  │  DLQ: events-dlq    │    │   DLQ: news-dlq          │  │    │
│  │  └──────────┬──────────┘    └────────────┬─────────────┘  │    │
│  └─────────────┼────────────────────────────┼────────────────┘    │
│                │                            │                       │
│                ▼                            ▼                       │
│  ┌──────────────────── PROCESSING LAYER ──────────────────────┐    │
│  │                                                             │    │
│  │              ┌──────────────────────┐                       │    │
│  │              │   λ SQS Processor    │                       │    │
│  │              │  (reads & writes DB) │                       │    │
│  │              └──────────┬───────────┘                       │    │
│  └─────────────────────────┼─────────────────────────────────┘    │
│                             │                                       │
│                             ▼                                       │
│  ┌──────────────────────── VPC ───────────────────────────────┐    │
│  │  ┌─────────────────────────────────────────────────────┐   │    │
│  │  │                  PUBLIC SUBNETS                      │   │    │
│  │  │  ┌──────────────────────────────────────────────┐   │   │    │
│  │  │  │   ALB (gikivault-alb)                        │   │   │    │
│  │  │  └──────────────────┬───────────────────────────┘   │   │    │
│  │  │                     │                                │   │    │
│  │  │  ┌──────────────────▼───────────────────────────┐   │   │    │
│  │  │  │   EC2 t3.micro (Flask App)                   │   │   │    │
│  │  │  │   /api/events  /api/weather  /health         │   │   │    │
│  │  │  └──────────────────────────────────────────────┘   │   │    │
│  │  └─────────────────────────────────────────────────┘   │   │    │
│  │  ┌─────────────────────────────────────────────────┐   │   │    │
│  │  │                  PRIVATE SUBNETS                 │   │   │    │
│  │  │  ┌──────────────────────────────────────────┐   │   │    │
│  │  │  │   RDS PostgreSQL 15 (db.t3.micro)        │   │   │    │
│  │  │  │   events | weather_cache | news           │   │   │    │
│  │  │  └──────────────────────────────────────────┘   │   │    │
│  │  └─────────────────────────────────────────────────┘   │   │    │
│  └─────────────────────────────────────────────────────────┘   │    │
│                             │                                       │
│  ┌──────────────────── DELIVERY LAYER ────────────────────────┐    │
│  │                                                             │    │
│  │   API Gateway  ──────►  ALB  ──────►  Flask EC2            │    │
│  │   (throttle + auth)                                         │    │
│  │                                                             │    │
│  │   S3 (frontend) ──────► CloudFront ──────► 🌐 Users        │    │
│  │   (static HTML/JS)       (HTTPS + CDN)                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌──────────────────── SECURITY & OPS ────────────────────────┐    │
│  │  Cognito (Auth)  │  SSM (Secrets)  │  CloudWatch + SNS     │    │
│  │  CloudTrail (Audit)  │  IAM (RBAC) │  EventBridge (Cron)   │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ☁️ AWS Services Used

<details>
<summary><b>🌐 Networking — VPC</b></summary>

```
VPC: gikivault-vpc
CIDR: 10.0.0.0/16
Region: us-east-1

┌─────────────────────────────────────────┐
│              gikivault-vpc              │
│                                         │
│  AZ-1 (us-east-1a)  AZ-2 (us-east-1b)  │
│  ┌─────────────┐    ┌─────────────┐    │
│  │Public Subnet│    │Public Subnet│    │
│  │ 10.0.1.0/24 │    │ 10.0.2.0/24 │    │
│  └─────────────┘    └─────────────┘    │
│  ┌─────────────┐    ┌─────────────┐    │
│  │Priv. Subnet │    │Priv. Subnet │    │
│  │ 10.0.3.0/24 │    │ 10.0.4.0/24 │    │
│  └─────────────┘    └─────────────┘    │
│                                         │
│  Internet Gateway → Public Route Table  │
└─────────────────────────────────────────┘
```

- 2 Public Subnets across 2 Availability Zones
- 2 Private Subnets for RDS (never internet-exposed)
- Internet Gateway for public subnet egress
- Security Groups as stateful firewalls

</details>

<details>
<summary><b>⚡ Serverless — AWS Lambda (Python 3.12)</b></summary>

```
EventBridge (every 15 min)
        │
        ├──► λ ticketmaster-ingestion
        │         └──► Ticketmaster API (Islamabad)
        │               └──► SQS: gikivault-events-queue
        │
        ├──► λ weather-fetch
        │         └──► OpenWeatherMap API (Topi, PK)
        │               └──► RDS: weather_cache table
        │
        └──► λ rss-campus-feed
                  └──► Campus RSS Feed
                        └──► SQS: gikivault-news-queue
```

All Lambda functions:
- Attached to `gikivault-lambda-role` (least privilege)
- Retrieve API keys from SSM Parameter Store at runtime
- Scheduled via Amazon EventBridge (rate: 15 minutes)

</details>

<details>
<summary><b>📨 Messaging — Amazon SQS</b></summary>

```
Producer                Queue                  Consumer
────────               ───────                 ────────
λ ticketmaster  ──►  events-queue  ──►  λ SQS Processor ──► RDS
                        │
                     events-dlq  (captures failures after 3 retries)

λ rss-campus    ──►  news-queue    ──►  λ SQS Processor ──► RDS
                        │
                     news-dlq    (captures failures after 3 retries)
```

Dead Letter Queues ensure **zero data loss** on processing failures.

</details>

<details>
<summary><b>🗄️ Database — Amazon RDS PostgreSQL 15</b></summary>

- Instance: `db.t3.micro` in **private subnet** (no internet exposure)
- Credentials stored in SSM Parameter Store (KMS encrypted)
- Connected only from EC2 via Security Group rule (port 5432)

</details>

<details>
<summary><b>🔐 Auth — Amazon Cognito</b></summary>

- User Pool: `GIKIVaultUsers` (`ciaxjk`)
- Email-based sign-up & login
- JWT tokens for stateless API auth
- Roles: `Student` | `Admin`

</details>

---

## 🔄 Data Flow

```
Step 1 ─── SCHEDULE
           EventBridge fires every 15 minutes
                │
                ▼
Step 2 ─── INGESTION
           Lambda fetches from external APIs
           (keys pulled from SSM at runtime)
                │
                ▼
Step 3 ─── QUEUING
           Raw JSON pushed to SQS queues
           (decouples ingestion from processing)
                │
                ▼
Step 4 ─── PROCESSING
           SQS processor Lambda reads messages
           Writes structured rows to RDS tables
                │
                ▼
Step 5 ─── SERVING
           Flask on EC2 reads from RDS
           Exposes REST endpoints via ALB
                │
                ▼
Step 6 ─── GATEWAY
           API Gateway applies throttling + API key auth
           Routes to ALB → EC2
                │
                ▼
Step 7 ─── FRONTEND
           React/HTML dashboard on S3
           Served via CloudFront (HTTPS, global CDN)
           Authenticated users see live data
```

---

## 📡 API Reference

**Base URL:** `https://xnwj8a9gl4.execute-api.us-east-1.amazonaws.com/prod`

> All endpoints require the `x-api-key` header (Usage Plan: 100 req/sec, 10,000 req/day)

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/events` | Returns processed event data from RDS | API Key |
| `GET` | `/weather` | Returns current weather for Topi, PK | API Key |
| `GET` | `/health` | EC2 health check (via ALB) | None |

### Example Response — `/weather`

```json
{
  "id": 42,
  "temp": 28.5,
  "humidity": 60,
  "description": "partly cloudy",
  "fetched_at": "2024-11-15T14:30:00Z"
}
```

### Example Response — `/events`

```json
[
  {
    "id": 101,
    "title": "Tech Expo Islamabad 2024",
    "source": "ticketmaster",
    "event_date": "2024-12-01T18:00:00Z",
    "location": "Islamabad",
    "category": "Technology",
    "created_at": "2024-11-15T14:30:00Z"
  }
]
```

---

## 🗃️ Database Schema

```sql
-- Stores events ingested from Ticketmaster API
CREATE TABLE events (
    id          SERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    source      TEXT,
    event_date  TIMESTAMP,
    location    TEXT,
    category    TEXT,
    raw_json    JSONB,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Caches weather data from OpenWeatherMap
CREATE TABLE weather_cache (
    id          SERIAL PRIMARY KEY,
    temp        NUMERIC(5,2),
    humidity    INTEGER,
    description TEXT,
    fetched_at  TIMESTAMP DEFAULT NOW()
);

-- Stores news items from campus RSS feed
CREATE TABLE news (
    id          SERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    link        TEXT,
    published   TIMESTAMP,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

---

## 🔒 Security Model

```
┌─────────────────────── SECURITY LAYERS ───────────────────────────┐
│                                                                    │
│  LAYER 1 — Network                                                 │
│  ├── RDS in private subnet (no internet route)                     │
│  ├── EC2 SG: inbound port 5000 from ALB SG only                   │
│  └── ALB SG: inbound ports 80/443 from 0.0.0.0/0                  │
│                                                                    │
│  LAYER 2 — Identity & Access                                       │
│  ├── MFA on root account                                           │
│  ├── Root never used; gikivault-admin for all ops                  │
│  ├── Lambda role: SQS + RDS + SSM + CloudWatch only                │
│  └── EC2 role: RDS + S3 read + SQS only                           │
│                                                                    │
│  LAYER 3 — Secrets                                                 │
│  ├── /gikivault/ticketmaster-key  (SecureString, KMS)              │
│  ├── /gikivault/openweather-key   (SecureString, KMS)              │
│  └── /gikivault/db-password       (SecureString, KMS)              │
│      └── Fetched at runtime via boto3 — zero hardcoded creds       │
│                                                                    │
│  LAYER 4 — Application Auth                                        │
│  ├── Cognito User Pool — email/password signup                     │
│  ├── JWT tokens issued on login (stateless)                        │
│  └── API Gateway key + usage plan throttling                       │
│                                                                    │
│  LAYER 5 — Audit                                                   │
│  └── CloudTrail logs ALL API calls → S3 bucket                     │
│      (IAM changes, Lambda invocations, S3 ops, etc.)               │
└────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Monitoring & Alerting

```
CloudWatch Alarms
      │
      ├── ticketmaster-errors
      │       Metric : Lambda Errors (ticketmaster-ingestion)
      │       Alarm  : > 5 errors in 5 minutes
      │       Action : SNS → gikivault-alerts → 📧 Email
      │
      ├── weather-errors
      │       Metric : Lambda Errors (weather-fetch)
      │       Alarm  : > 5 errors in 5 minutes
      │       Action : SNS → gikivault-alerts → 📧 Email
      │
      └── rss-errors
              Metric : Lambda Errors (rss-campus-feed)
              Alarm  : > 5 errors in 5 minutes
              Action : SNS → gikivault-alerts → 📧 Email
```

> **CloudTrail** (`gikivault-trail`) records every AWS API call and stores logs in `aws-cloudtrail-logs-470379384049-d05b59aa` for forensic audit capability.

---

## 🚀 Deployment & Configuration

### Lambda Environment Variables

| Function | Variable | Source |
|----------|----------|--------|
| `ticketmaster-ingestion` | `SQS_EVENTS_URL` | Hardcoded env var |
| `rss-campus-feed` | `SQS_NEWS_URL` | Hardcoded env var |
| All functions | API Keys | SSM Parameter Store |
| EC2 Flask app | `RDS_HOST` | Environment variable |

### Key Resource Identifiers

| Resource | Identifier |
|----------|------------|
| VPC | `gikivault-vpc` (`10.0.0.0/16`) |
| RDS Endpoint | `gikivault-db.xxxx.us-east-1.rds.amazonaws.com` |
| API Gateway | `https://xnwj8a9gl4.execute-api.us-east-1.amazonaws.com/prod` |
| CloudFront | `https://d3jqcqbhgw2313.cloudfront.net` |
| CloudTrail Bucket | `aws-cloudtrail-logs-470379384049-d05b59aa` |
| Cognito User Pool | `User pool - ciaxjk` |
| SNS Topic | `gikivault-alerts` |
| Region | `us-east-1` (N. Virginia) |

### Fetch a Secret (boto3)

```python
import boto3

ssm = boto3.client('ssm', region_name='us-east-1')

def get_secret(name: str) -> str:
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response['Parameter']['Value']

TICKETMASTER_KEY = get_secret('/gikivault/ticketmaster-key')
WEATHER_KEY      = get_secret('/gikivault/openweather-key')
DB_PASSWORD      = get_secret('/gikivault/db-password')
```

### Lambda Ingestion Pattern

```python
import boto3, json, requests

sqs = boto3.client('sqs')
SQS_URL = os.environ['SQS_EVENTS_URL']

def handler(event, context):
    api_key = get_secret('/gikivault/ticketmaster-key')
    resp = requests.get(
        'https://app.ticketmaster.com/discovery/v2/events.json',
        params={'apikey': api_key, 'city': 'Islamabad', 'size': 20}
    )
    events = resp.json().get('_embedded', {}).get('events', [])
    for e in events:
        sqs.send_message(
            QueueUrl=SQS_URL,
            MessageBody=json.dumps(e)
        )
    return {'statusCode': 200, 'count': len(events)}
```

---

## ⚠️ Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| EC2 in private subnet had no internet access (no NAT Gateway budget) | Moved EC2 to public subnet with Elastic IP; Lambda kept outside VPC so it can call external APIs without NAT Gateway costs |
| EC2 Instance Connect not working | Created EC2 Instance Connect Endpoint inside VPC; assigned Elastic IP to EC2 |
| RDS refusing EC2 connections | Added inbound PostgreSQL rule (port 5432) to RDS security group, sourced from EC2 security group (not IP) |
| S3 returning 403 via CloudFront | Disabled Block Public Access on frontend bucket; added bucket policy granting public `s3:GetObject` |
| CloudFront serving blank page | File uploaded as `index (1).html`; deleted and re-uploaded with correct `index.html` name |

---

## 👥 Team

| Role | Name | Student ID |
|------|------|------------|
| **Team Lead** | Abdul Rehman Faisal | 2023901 |
| **Member** | Batool Fatima | 2023159 |

**Course:** CE308 — Introduction to Cloud Computing  
**Semester:** Fall 2024  
**Institute:** GIKI, Topi, KPK, Pakistan  
**Instructor:** Ms. Safia Baloch  

---

<div align="center">

**Built with ❤️ on AWS · GIKI Fall 2024**

[![Live Demo](https://img.shields.io/badge/🌐_Live_Demo-d3jqcqbhgw2313.cloudfront.net-FF9900?style=for-the-badge)](https://d3jqcqbhgw2313.cloudfront.net)

</div>
