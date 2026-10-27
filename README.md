# Scalable Web Application with ALB and Auto Scaling on AWS

A production-grade, highly available and scalable web application on AWS using EC2 in a multi-AZ VPC, fronted by CloudFront, WAF and an Application Load Balancer, with a Multi-AZ RDS database and bastion-free access through Systems Manager.

---

## 1. Solution Architecture

![Architecture Diagram](architecture/architecture-diagram.png)

> Regenerate the diagram with `pip install diagrams && python architecture/generate_diagram.py` (requires Graphviz).

### Request flow
1. A user resolves the domain through **Route 53** (alias record + health checks).
2. **CloudFront** serves cached static assets from **S3**; dynamic requests are forwarded to the origin.
3. **AWS WAF** filters traffic (OWASP Top 10 managed rules, rate limiting) before it reaches the ALB.
4. The **ALB** (public subnets, both AZs) routes requests to healthy targets using listener rules.
5. **EC2 instances** in the **private app subnets**, managed by an **Auto Scaling Group**, process the request.
6. The app reads and writes to **RDS Multi-AZ** in **private DB subnets**.
7. Outbound traffic (patches, package installs) goes through a **NAT Gateway** per AZ.
8. Admins reach instances only via **Systems Manager Session Manager** (no SSH, no bastion, no open port 22).
9. **CloudWatch** collects metrics and logs; alarms notify the team through **SNS**.

---

## 2. AWS Services Used

| Service | Role in the solution |
|---|---|
| **VPC** | Isolated network `10.0.0.0/16` across 2 AZs, public / private app / private DB tiers |
| **Internet Gateway** | Internet access for public subnets |
| **NAT Gateway** (x2) | Outbound-only internet for private instances (one per AZ for HA) |
| **EC2 + Launch Template** | Web/app servers (Amazon Linux 2023, IMDSv2, SSM agent) |
| **Auto Scaling Group** | Maintains capacity across AZs, target tracking on CPU and ALB requests per target |
| **ALB** | Layer 7 load balancing, health checks, path-based routing |
| **AWS WAF** | AWS Managed Rules (Core, SQLi, Known Bad Inputs), rate-based rule |
| **CloudFront** | CDN caching static assets, TLS termination, reduced latency |
| **S3** | Static asset origin (private, accessed via Origin Access Control) |
| **RDS Multi-AZ** | MySQL/PostgreSQL with synchronous standby and automatic failover |
| **Route 53** | Alias record to CloudFront/ALB, health checks |
| **Systems Manager** | Session Manager for secure shell access |
| **CloudWatch + SNS** | Dashboards, alarms, email/SMS notifications |
| **IAM** | Instance profile with `AmazonSSMManagedInstanceCore`, least privilege |

---

## 3. Network Design

| Subnet | AZ | CIDR | Route table |
|---|---|---|---|
| Public-A | us-east-1a | 10.0.1.0/24 | `0.0.0.0/0 -> IGW` |
| Public-B | us-east-1b | 10.0.2.0/24 | `0.0.0.0/0 -> IGW` |
| Private-App-A | us-east-1a | 10.0.11.0/24 | `0.0.0.0/0 -> NAT-A` |
| Private-App-B | us-east-1b | 10.0.12.0/24 | `0.0.0.0/0 -> NAT-B` |
| Private-DB-A | us-east-1a | 10.0.21.0/24 | local only |
| Private-DB-B | us-east-1b | 10.0.22.0/24 | local only |

*(Replace the region/AZs with the ones you deploy to.)*

### Security Groups (stateful)

| SG | Inbound | Outbound |
|---|---|---|
| `alb-sg` | 80/443 from `0.0.0.0/0` (or CloudFront prefix list) | to `app-sg` on app port |
| `app-sg` | App port (80/8080) from `alb-sg` only | 443 to internet (via NAT), 3306/5432 to `db-sg` |
| `db-sg` | 3306/5432 from `app-sg` only | none |

### Network ACLs (stateless, defense in depth)

| NACL | Allow inbound | Allow outbound |
|---|---|---|
| Public | 80, 443, ephemeral 1024-65535 | all |
| Private App | App port from public subnets, ephemeral 1024-65535 | 443, DB port to DB subnets, ephemeral |
| Private DB | DB port from app subnets only | ephemeral to app subnets |

---

## 4. Compute and Scaling

- **Launch Template:** `t3.micro`/`t3.small`, Amazon Linux 2023, IMDSv2 required, IAM instance profile with SSM, user data installs and starts the web server.
- **ASG:** min 2, desired 2, max 6, spread across both private app subnets, ELB health checks, 300s grace period.
- **Scaling policies:**
  - *Target tracking:* average CPU at 50%.
  - *Target tracking:* `ALBRequestCountPerTarget` at 1000.
  - *Step scaling (optional):* +2 instances when CPU > 80% for 2 minutes, -1 when CPU < 30%.
- **ALB target group:** health check `GET /health`, healthy threshold 3, unhealthy threshold 2, interval 15s, deregistration delay 30s.

---

## 5. Security

- All compute and database tiers are in **private subnets** with no public IPs.
- **WAF** web ACL with AWS Managed Rules (Core Rule Set, SQL Database, Known Bad Inputs) and a rate-based rule (e.g. 2000 req / 5 min / IP).
- **Security Group chaining** (ALB -> App -> DB), never CIDR-based between tiers.
- **No SSH:** port 22 closed, access via **Session Manager** with IAM-based auth and session logging to CloudWatch/S3.
- **Encryption:** TLS via ACM on CloudFront and ALB, RDS and EBS encrypted with KMS, S3 bucket private with OAC.
- **Secrets:** DB credentials in AWS Secrets Manager (or SSM Parameter Store).
- **IMDSv2** enforced on all instances.

---

## 6. High Availability and Resilience

| Failure | Mitigation |
|---|---|
| EC2 instance fails | ALB health check removes it, ASG replaces it |
| Entire AZ fails | Traffic shifts to the other AZ; RDS fails over to the standby (typically 60-120s) |
| NAT Gateway failure | Each AZ has its own NAT Gateway |
| Traffic spike | Target tracking scales out automatically |
| Region-level DNS/endpoint issue | Route 53 health checks |

---

## 7. Monitoring and Alerting

**CloudWatch dashboard** widgets: ALB request count and 5xx, target response time, healthy host count, ASG instance count, EC2 CPU, RDS CPU / connections / free storage, CloudFront cache hit rate.

**Alarms -> SNS topic -> email subscription:**

| Alarm | Condition |
|---|---|
| High CPU | ASG average CPU > 80% for 5 min |
| Unhealthy targets | `UnHealthyHostCount` >= 1 for 3 min |
| ALB 5xx | `HTTPCode_Target_5XX_Count` > 10 in 5 min |
| RDS CPU | > 80% for 10 min |
| RDS storage | `FreeStorageSpace` < 2 GB |
| Route 53 health check | Status = unhealthy |

---

## 8. Deployment Steps (AWS Console)

1. **VPC:** create VPC, 6 subnets, IGW, 2 NAT Gateways, route tables, NACLs.
2. **Security Groups:** `alb-sg`, `app-sg`, `db-sg`.
3. **IAM:** create role `EC2-SSM-Role` with `AmazonSSMManagedInstanceCore` and `CloudWatchAgentServerPolicy`.
4. **RDS:** DB subnet group (private DB subnets), Multi-AZ instance, encryption on, `db-sg`.
5. **Launch Template:** AMI, instance profile, `app-sg`, user data (below).
6. **ALB:** internet-facing, public subnets, target group with `/health` check, HTTP->HTTPS redirect.
7. **ASG:** attach to target group, private app subnets, add scaling policies.
8. **WAF:** create web ACL, add managed rules, associate with the ALB.
9. **S3 + CloudFront:** private bucket for static assets, distribution with two origins (S3 for `/static/*`, ALB for default), ACM certificate.
10. **Route 53:** alias A record to the CloudFront distribution, add health check.
11. **CloudWatch + SNS:** create topic, confirm email subscription, create alarms and dashboard.
12. **Test** (section 9).

### Sample user data
```bash
#!/bin/bash
dnf update -y
dnf install -y httpd php php-mysqlnd
systemctl enable --now httpd
echo "OK" > /var/www/html/health
cat > /var/www/html/index.php <<'PHP'
<?php
echo "<h1>Scalable Web App</h1>";
echo "<p>Instance: " . gethostname() . "</p>";
PHP
```

---

## 9. Testing and Validation

| Test | How | Expected result |
|---|---|---|
| Load balancing | Refresh the site repeatedly | Hostname alternates between instances in both AZs |
| Auto Scaling | `sudo dnf install -y stress && stress --cpu 4 --timeout 600` via Session Manager | New instances launch when CPU > 50% |
| Self-healing | Terminate an instance | ASG launches a replacement |
| RDS failover | Reboot with failover | App recovers after ~1-2 min |
| WAF | `curl "https://yourdomain/?id=1' OR '1'='1"` | HTTP 403 |
| Caching | `curl -I https://yourdomain/static/logo.png` twice | `X-Cache: Hit from cloudfront` |
| No SSH | Try `ssh` to an instance | No route, connection refused; use Session Manager |
| Alerts | Trigger an alarm | SNS email received |

---

## 10. Cost Considerations

Main cost drivers: **2 NAT Gateways**, **ALB**, **RDS Multi-AZ**, EC2 instances and **WAF**. To reduce cost in a learning environment use a single NAT Gateway, `db.t3.micro`, and **delete all resources after the demo**. Use AWS Budgets to set an alert.

---

## 11. Cleanup

Delete in this order: Route 53 records -> CloudFront -> WAF -> ASG -> ALB and target groups -> RDS (disable deletion protection) -> NAT Gateways and release Elastic IPs -> VPC -> S3 bucket -> CloudWatch alarms and SNS topic.

---

## 12. Learning Outcomes Achieved

- Designed a VPC with correct subnet, route table and NAT Gateway configuration.
- Built a highly available architecture across two Availability Zones.
- Configured ALB listener rules and target group health checks.
- Implemented Auto Scaling with target tracking and step scaling.
- Secured the application with WAF, Security Groups, NACLs and private subnets.
- Used Systems Manager Session Manager as a bastion-free access method.

---

## 13. Repository Structure

```
.
├── README.md
└── architecture/
    ├── architecture-diagram.png
    └── generate_diagram.py
```

## 14. Demo (optional)

- Live URL: _add your URL here_
- Video: _add your video link here_

## 15. Author

_Your name · AWS Cloud Graduation Project_
