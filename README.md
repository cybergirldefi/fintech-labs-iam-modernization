# FinTech Labs IAM Modernization Project

> Designing, auditing, and securing an identity-centric, Zero Trust IAM framework for FinTech Labs Inc.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Cloud](https://img.shields.io/badge/cloud-AWS-orange)
![Model](https://img.shields.io/badge/security%20model-Zero%20Trust-blue)

---

## Table of Contents

1. [Overview](#overview)
2. [Scenario](#scenario)
3. [Objectives](#objectives)
4. [Part 1: Identity Inventory & Taxonomy](#part-1-identity-inventory--taxonomy)
5. [Part 2: Least-Privilege Access Matrix](#part-2-least-privilege-access-matrix)
6. [Part 3: Incident Investigation & Audit Analysis](#part-3-incident-investigation--audit-analysis)
7. [Part 4: Zero Trust Executive Summary](#part-4-zero-trust-executive-summary)
8. [Hands-On AWS Implementation](#hands-on-aws-implementation)
9. [Testing & Verification](#testing--verification)
10. [Key Takeaways](#key-takeaways)
11. [Submission Checklist](#submission-checklist)
12. [Cleanup](#cleanup)

---

## Overview

This project is a take-home assignment that applies Week 1 IAM concepts to a realistic scenario. It covers the full **AAA framework** (Authentication, Authorization, Accounting) and demonstrates **Least Privilege (PoLP)** and **Separation of Duties (SoD)** with a working implementation in AWS.

## Scenario

**FinTech Labs Inc.** is a fast-growing financial technology startup that has relied on a traditional on-premise "Castle-and-Moat" network. After a near-miss incident, where a developer's leaked password allowed an unauthorized script to touch customer records, leadership mandated a shift to an **Identity-Centric, Zero Trust Security Model**.

As Lead IAM Security Engineer, the goal is to deliver a four-part modernization proposal and audit.

## Objectives

- Classify human and non-human identities and assess the risk of each.
- Replace broad admin access with a least-privilege, SoD-compliant access model.
- Investigate a suspicious event using audit logs and the AAA framework.
- Communicate the case for Zero Trust to executive leadership.
- Implement and test the access model in AWS IAM.

---

## Part 1: Identity Inventory & Taxonomy

| # | Entity | Identity Category | Primary Risk if Compromised |
|---|--------|-------------------|-----------------------------|
| 1 | **Sarah** (backend software engineer) | Workforce Identity | Attacker gains access to source code, secrets, and the deployment pipeline, enabling code tampering or a supply-chain attack. |
| 2 | **Payment-Gateway-API-Key** (token used to talk to Stripe) | Non-Human / Workload Identity | Direct financial fraud: unauthorized charges, refunds, or payment data exposure. Keys are often long-lived and rarely rotated. |
| 3 | **Alex** (customer service representative) | Workforce Identity | Exposure of customer personal data in support tickets, plus social-engineering or account-takeover abuse. |
| 4 | **Lambda-Log-Processor** (serverless function scraping audit logs hourly) | Non-Human / Workload Identity | Attacker can read or tamper with audit logs, hiding evidence and blinding detection. |

> **Note:** No entity in this inventory is a *Customer Identity*. Customer Identities (CIAM) are the external end users of FinTech Labs' products, not employees or workloads.

---

## Part 2: Least-Privilege Access Matrix

| Role | `Res-Dev-Code` | `Res-Prod-Database` | `Res-IAM-Console` |
|------|:--------------:|:-------------------:|:-----------------:|
| **Software Engineer** | Read/Write | None | None |
| **Database Administrator (DBA)** | None | Read/Write | None |
| **IAM Security Admin** | None | None | Read/Write |
| **Security Auditor** | Read | Read (metadata/logs only) | Read |
| **Customer Support Rep** | None | None | None |

### Separation of Duties (SoD) rationale

- **Developers cannot modify production databases.** Engineers have no access to `Res-Prod-Database`, so a compromised developer account cannot alter customer financial records.
- **DBAs cannot push code.** DBAs have no access to `Res-Dev-Code`, so no single person can both change application code and the production data it touches.
- **Nobody administers their own access.** Only the IAM Security Admin can change permissions, and that role has no access to code or data.
- **Support staff use application-level tooling**, never direct infrastructure access.

> The role list was not specified in the assignment brief, so the roles above are a reasonable set based on the scenario. Adjust them to match your submission if needed.

---

## Part 3: Incident Investigation & Audit Analysis

### Log excerpt

```json
{
  "timestamp": "2026-09-24T02:14:05Z",
  "identity_type": "IAM Role",
  "principal": "arn:aws:iam::123456789:role/DevOps-Deployment-Role",
  "source_ip": "198.51.100.42",
  "action": "rds:DownloadDBClusterSnapshot",
  "status": "SUCCESS"
}
```

### Forensic analysis (AAA framework)

| Question | Finding | AAA Pillar |
|----------|---------|------------|
| **Who** performed the action? | A non-human **IAM Role**, `DevOps-Deployment-Role`, in account `123456789`. The role is a workload identity, but a person or script *assumed* it, so the real actor is still unknown. | Authentication |
| **What** was done? | A snapshot of the production database cluster was downloaded (`rds:DownloadDBClusterSnapshot`). This is a bulk data-exfiltration action. | Accounting |
| **When** did it happen? | `2026-09-24` at `02:14:05 UTC`, outside normal business hours and consistent with the reported 2:00 AM event window. | Accounting |
| **Where** did it come from? | Source IP `198.51.100.42`. This must be checked against the company's known IP ranges, VPN, and CI/CD runners. An unrecognized IP suggests external access. | Accounting |
| **Was it allowed?** | Yes, `status: SUCCESS`. The role was permitted to perform the action. | Authorization |
| **Why is this a problem?** | A *deployment* role should not need to download production database snapshots. This points to **over-privileged permissions**, a failure of least privilege. | Authorization |

### Conclusions and recommendations

1. **Authentication failure:** Investigate how the role was assumed (stolen credentials, leaked access keys, or a compromised pipeline). Check CloudTrail for `AssumeRole` events preceding this one.
2. **Authorization failure:** Remove `rds:DownloadDBClusterSnapshot` and any prod-data access from `DevOps-Deployment-Role`. Scope it to deployment actions only.
3. **Accounting gap:** Add alerts for off-hours snapshot activity and unknown source IPs (CloudWatch / GuardDuty).
4. **Containment:** Revoke active sessions for the role, rotate credentials, and review all actions taken from `198.51.100.42`.
5. **Hardening:** Enforce MFA for human users, use short-lived credentials for roles, and restrict role assumption by source IP or VPC condition keys.

---

## Part 4: Zero Trust Executive Summary

**To the CEO: Why is our firewall no longer enough, and how does Identity as the Perimeter help?**

Our traditional firewall assumes everything inside the network is trustworthy and everything outside is hostile. That model no longer fits how we operate. Our infrastructure now lives in the cloud, our engineers work from anywhere, and our applications talk to each other through APIs and automated tokens that never pass through a single network gate. As our recent incident showed, one leaked developer password let an unauthorized script reach customer records, and the firewall never noticed because the request looked like a trusted insider.

Shifting to Identity as the Perimeter closes this gap. Instead of trusting a location, we verify every person, service, and script each time it requests access: authenticate who it is, authorize only the minimum permissions its job requires, and record every action for audit. Multi-factor authentication makes stolen passwords far less useful, least-privilege roles limit the damage of any single compromise, and detailed logs let us detect and investigate suspicious activity quickly. The result is smaller exposure, faster detection, and stronger protection of customer financial data.

---

## Hands-On AWS Implementation

This section builds the access model from Part 2 in AWS. Two S3 buckets simulate the development code repository and the production data store.

### Prerequisites

- An AWS account with permission to create S3 buckets and IAM resources
- Access to the AWS Management Console

### Architecture

```
SoftwareEngineers (group) ── FinTech-SoftwareEngineer-Policy ──▶ fintech-dev-code-<initials>
        └── sarah-dev

DatabaseAdmins (group)    ── FinTech-DBA-Policy ───────────────▶ fintech-prod-data-<initials>
        └── bob-dba
```

### Step 1: Create the cloud resources (S3 buckets)

1. Log in to the AWS Management Console and open **S3**.
2. Click **Create bucket** and create:
   - `fintech-dev-code-<your-initials>` (simulates `Res-Dev-Code`)
   - `fintech-prod-data-<your-initials>` (simulates `Res-Prod-Database`)
3. Leave the defaults (Block Public Access stays on) and click **Create bucket** for each.

<details>
<summary>📸 Click to view: both buckets created</summary>

<br>

<a href="screenshots/01-s3-buckets.png">
  <img src="screenshots/01-s3-buckets.png" alt="S3 console showing the dev code and prod data buckets" width="600">
</a>

</details>

### Step 2: Create custom least-privilege IAM policies

Go to **IAM → Policies → Create policy → JSON**.

#### Policy A: `FinTech-SoftwareEngineer-Policy`

Grants access to the development code bucket only.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowConsoleListing",
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowDevCodeAccessOnly",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::fintech-dev-code-<your-initials>",
        "arn:aws:s3:::fintech-dev-code-<your-initials>/*"
      ]
    }
  ]
}
```

<details>
<summary>📸 Click to view: Software Engineer policy in AWS</summary>

<br>

<a href="screenshots/02-policy-software-engineer.png">
  <img src="screenshots/02-policy-software-engineer.png" alt="FinTech-SoftwareEngineer-Policy JSON in the IAM policy editor" width="600">
</a>

</details>

#### Policy B: `FinTech-DBA-Policy`

Grants access to the production data bucket only.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowConsoleListing",
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowProdDataAccessOnly",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::fintech-prod-data-<your-initials>",
        "arn:aws:s3:::fintech-prod-data-<your-initials>/*"
      ]
    }
  ]
}
```

<details>
<summary>📸 Click to view: DBA policy in AWS</summary>

<br>

<a href="screenshots/03-policy-dba.png">
  <img src="screenshots/03-policy-dba.png" alt="FinTech-DBA-Policy JSON in the IAM policy editor" width="600">
</a>

</details>

> Replace `<your-initials>` with the same initials used for your buckets. AWS denies everything not explicitly allowed (implicit deny), which is what enforces SoD here.

### Step 3: Create user groups and test users

Policies are attached to **groups**, not individual users, following IAM best practice.

| Group | Attached Policy | Member |
|-------|-----------------|--------|
| `SoftwareEngineers` | `FinTech-SoftwareEngineer-Policy` | `sarah-dev` |
| `DatabaseAdmins` | `FinTech-DBA-Policy` | `bob-dba` |

1. **IAM → User groups → Create group** for each group above and attach its policy.
2. **IAM → Users → Create user** for `sarah-dev` and `bob-dba`, enable console access, and add each to their group.

<details>
<summary>📸 Click to view: groups and users</summary>

<br>

**User groups**

| `SoftwareEngineers` | `DatabaseAdmins` |
|:---:|:---:|
| <a href="screenshots/04-group-software-engineers.png"><img src="screenshots/04-group-software-engineers.png" alt="SoftwareEngineers user group created" width="380"></a> | <a href="screenshots/05-group-database-admins.png"><img src="screenshots/05-group-database-admins.png" alt="DatabaseAdmins user group created" width="380"></a> |

**Users, with policies inherited through their groups**

| `sarah-dev` | `bob-dba` |
|:---:|:---:|
| <a href="screenshots/06-user-sarah-dev.png"><img src="screenshots/06-user-sarah-dev.png" alt="sarah-dev inherits FinTech-SoftwareEngineer-Policy via SoftwareEngineers" width="380"></a> | <a href="screenshots/07-user-bob-dba.png"><img src="screenshots/07-user-bob-dba.png" alt="bob-dba inherits FinTech-DBA-Policy via DatabaseAdmins" width="380"></a> |

</details>

---

## Testing & Verification

| Test | Steps | Expected Result |
|------|-------|-----------------|
| **Sarah → dev bucket** | Sign in as `sarah-dev`, open S3, open `fintech-dev-code-<initials>` | ✅ Can read and write objects |
| **Sarah → prod bucket** | Open `fintech-prod-data-<initials>` | ❌ **Access Denied** |
| **Bob → prod bucket** | Sign in as `bob-dba`, open `fintech-prod-data-<initials>` | ✅ Can read and write objects |
| **Bob → dev bucket** | Open `fintech-dev-code-<initials>` | ❌ **Access Denied** |

Both users can *see* all bucket names (via `s3:ListAllMyBuckets`) so the console works, but neither can open the other team's bucket. **Separation of Duties is enforced.**

### Evidence (click any image to enlarge)

**Sarah (`sarah-dev`), Software Engineer**

| Dev bucket: allowed ✅ | Prod bucket: denied ❌ |
|:---:|:---:|
| <a href="screenshots/08-sarah-dev-bucket-success.png"><img src="screenshots/08-sarah-dev-bucket-success.png" alt="Sarah uploads a file to the dev code bucket" width="380"></a> | <a href="screenshots/09-sarah-prod-access-denied.png"><img src="screenshots/09-sarah-prod-access-denied.png" alt="Sarah receives insufficient permissions on the prod data bucket" width="380"></a> |

**Bob (`bob-dba`), Database Administrator**

| Prod bucket: allowed ✅ | Dev bucket: denied ❌ |
|:---:|:---:|
| <a href="screenshots/10-bob-prod-bucket-success.png"><img src="screenshots/10-bob-prod-bucket-success.png" alt="Bob uploads a file to the prod data bucket" width="380"></a> | <a href="screenshots/11-bob-dev-access-denied.png"><img src="screenshots/11-bob-dev-access-denied.png" alt="Bob receives insufficient permissions on the dev code bucket" width="380"></a> |

### Troubleshooting note

My first test as `sarah-dev` on the dev bucket failed with **"Insufficient permissions to list objects"**, even though she should have access. The cause was a mismatch between the bucket name and the resource ARN in the policy. The dev bucket was created as `fin-tech-dev-code-titi`, while the policy referenced `fintech-dev-code-<initials>`. IAM matches resource names exactly, so the policy never applied and the implicit deny took over. After correcting the resource ARNs so they match the real bucket names (the DBA policy was also updated to version 2), the tests passed.

**Lesson:** least-privilege policies fail closed. A single typo in a resource ARN blocks access rather than granting too much, which is the safer failure mode, but ARNs should always be copied from the real resource.

<details>
<summary>📸 Click to view the troubleshooting screenshots</summary>

<br>

| Initial failure on the dev bucket | DBA policy after the fix |
|:---:|:---:|
| <a href="screenshots/12-troubleshoot-initial-denied.png"><img src="screenshots/12-troubleshoot-initial-denied.png" alt="Sarah initially denied on the dev bucket" width="380"></a> | <a href="screenshots/13-troubleshoot-dba-policy-updated.png"><img src="screenshots/13-troubleshoot-dba-policy-updated.png" alt="FinTech-DBA-Policy updated to version 2" width="380"></a> |

</details>

---

## Repository Structure

```
fintech-labs-iam-modernization/
├── README.md
└── screenshots/
    ├── 01-s3-buckets.png
    ├── 02-policy-software-engineer.png
    ├── 03-policy-dba.png
    ├── 04-group-software-engineers.png
    ├── 05-group-database-admins.png
    ├── 06-user-sarah-dev.png
    ├── 07-user-bob-dba.png
    ├── 08-sarah-dev-bucket-success.png
    ├── 09-sarah-prod-access-denied.png
    ├── 10-bob-prod-bucket-success.png
    ├── 11-bob-dev-access-denied.png
    ├── 12-troubleshoot-initial-denied.png
    └── 13-troubleshoot-dba-policy-updated.png
```

> Filenames are case-sensitive on GitHub. Make sure your screenshots match these names exactly.

---

## Key Takeaways

- **Identity is the new perimeter.** Every request is authenticated and authorized, regardless of network location.
- **Least privilege limits blast radius.** A compromised account can only reach what its role requires.
- **Separation of Duties prevents single points of failure.** No one role can both change code and alter production data.
- **Non-human identities need the same rigor as people.** API keys, roles, and functions should be scoped, rotated, and monitored.
- **Logging closes the loop.** Without accounting, you cannot detect or investigate misuse.

---

## Submission Checklist

- [x] Part 1: Completed identity taxonomy table with risk analysis
- [x] Part 2: Built a least-privilege access matrix respecting Separation of Duties
- [x] Part 3: Answered forensic audit questions using the log snippet
- [x] Part 4: Drafted the Zero Trust executive summary

## Cleanup

To avoid unnecessary charges and leftover access, delete these resources when finished:

1. Empty and delete both S3 buckets.
2. Delete the users `sarah-dev` and `bob-dba`.
3. Delete the groups `SoftwareEngineers` and `DatabaseAdmins`.
4. Delete the policies `FinTech-SoftwareEngineer-Policy` and `FinTech-DBA-Policy`.

---

## Author

**Your Name**: Lead IAM Security Engineer (Take-Home Project)
[LinkedIn](https://linkedin.com/in/your-profile) · [GitHub](https://github.com/your-username)
