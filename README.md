# IAM-Advanced-Policy-Lab
Designed and tested three custom IAM policies — resource-level restriction, MFA enforcement, and time-based access control — then verified them with IAM Policy Simulator and audited the result with IAM Access Analyzer.

# 🔐 AWS IAM Advanced Policy Lab

> Designing and testing custom IAM policies with resource-level restrictions, MFA enforcement, and time-based access controls — entirely through the AWS Console.

![AWS](https://img.shields.io/badge/AWS-IAM-FF9900?style=flat&logo=amazon-aws&logoColor=white)
![Status](https://img.shields.io/badge/Status-Complete-1A6B3A)
![Cost](https://img.shields.io/badge/Cost-%240.00-1A6B3A)

---

## 📋 Overview

Most beginners rely entirely on AWS managed policies like `ReadOnlyAccess` or `AdministratorAccess`. This project goes further — I designed **three custom IAM policies from scratch**, each demonstrating a different advanced access-control technique used by real cloud security teams:

1. **Resource-level restriction** — limit access to a single named S3 bucket, not all buckets
2. **MFA enforcement via condition keys** — deny every action unless multi-factor authentication was used
3. **Time-based access control** — allow S3 access only during defined business hours

Every policy was validated in the **IAM Policy Simulator** before being attached to a live IAM user, following the same test-before-deploy workflow used in production environments.

---

## 🧱 Architecture

```
                     ┌──────────────────────────┐
                     │      policy-test-user      │
                     └──────────────┬─────────────┘
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                   │
   ┌────────────▼───────┐ ┌─────────▼──────────┐ ┌──────▼───────────────┐
   │ S3-Specific-Bucket- │ │ Deny-All-Without-   │ │ S3-Business-Hours-   │
   │ ReadOnly            │ │ MFA                 │ │ Only                 │
   │                      │ │                      │ │                       │
   │ Allow: GetObject,    │ │ Deny: *  unless      │ │ Allow: S3 read 09–18  │
   │ ListBucket           │ │ MFA present          │ │ Deny: S3 * after 18   │
   │ Resource: 1 bucket   │ │ Resource: *          │ │ Resource: *           │
   └──────────────────────┘ └──────────────────────┘ └───────────────────────┘
```

| Resource | Purpose |
|---|---|
| `nikunj-allowed-bucket` | Target bucket — access **allowed** by Policy 1 |
| `nikunj-restricted-bucket` | Control bucket — access **denied** by Policy 1 (proves resource-level restriction) |
| `policy-test-group` | IAM group holding Policy 1 |
| `policy-test-user` | IAM user used for all simulation and attachment testing |

---

## 🛠️ AWS Services Used

| Service | How it was used |
|---|---|
| **IAM Policies** | Authored 3 customer-managed JSON policies (Visual editor + raw JSON) |
| **IAM Policy Simulator** | Validated Allow/Deny behaviour for every policy before deployment |
| **IAM Users & Groups** | Created an isolated test identity with zero default permissions |
| **Amazon S3** | Two buckets used as real targets for resource-level ARN testing |
| **IAM Access Analyzer** | Verified zero unintended public exposure after policies were applied |

---

## 🔑 Policy 1 — `S3-Specific-Bucket-ReadOnly`

Restricts S3 read access to **one named bucket only**, instead of the AWS-managed `AmazonS3ReadOnlyAccess` which grants access to every bucket in the account.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::nikunj-allowed-bucket",
        "arn:aws:s3:::nikunj-allowed-bucket/*"
      ]
    }
  ]
}
```

**Simulator result:**

| Action | Resource | Result |
|---|---|---|
| `s3:GetObject` | `nikunj-allowed-bucket/*` | ✅ Allowed |
| `s3:ListBucket` | `nikunj-allowed-bucket` | ✅ Allowed |
| `s3:GetObject` | `nikunj-restricted-bucket/*` | ❌ Implicitly denied |

---

## 🔑 Policy 2 — `Deny-All-Without-MFA`

Explicit `Deny` on every action and resource unless the session used multi-factor authentication. Demonstrates condition-key based zero-trust enforcement.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllWithoutMFA",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

**Simulator result:**

| Condition | Result |
|---|---|
| `aws:MultiFactorAuthPresent = false` | ❌ Denied — *1 matching statement* |
| `aws:MultiFactorAuthPresent = true` | ✅ Deny lifted — other policy Allows apply |

---

## 🔑 Policy 3 — `S3-Business-Hours-Only`

Restricts S3 read access to a 09:00–18:00 UTC window using `DateGreaterThan` / `DateLessThan` condition operators, paired with an explicit Deny for all other hours.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3DuringBusinessHours",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*",
      "Condition": {
        "DateGreaterThan": { "aws:CurrentTime": "2000-01-01T09:00:00Z" },
        "DateLessThan":    { "aws:CurrentTime": "2000-01-01T18:00:00Z" }
      }
    },
    {
      "Sid": "DenyOutsideBusinessHours",
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "DateGreaterThan": { "aws:CurrentTime": "2000-01-01T18:00:00Z" }
      }
    }
  ]
}
```

**Simulator result:**

| Simulated time | Result |
|---|---|
| 10:00 UTC (business hours) | ✅ Allowed |
| 20:00 UTC (after hours) | ❌ Denied |

---

## ✅ Verification

- **IAM Policy Simulator** — every policy tested individually *and* combined at the user level (group policy + directly attached policies evaluated together)
- **IAM Access Analyzer** — `account-access-analyzer` created with zone of trust = current account → **0 active findings**, confirming no S3 bucket became unintentionally public while testing

---

## 📸 Screenshots

| # | Screenshot | Shows |
|---|---|---|
| 1 | `policy1-json.png` | Policy 1 JSON with resource-level ARN |
| 2 | `policy2-json.png` | Policy 2 JSON with MFA condition block |
| 3 | `policy3-json.png` | Policy 3 JSON with time-based conditions |
| 4 | `simulator-allow-deny.png` | Policy 1 — allowed bucket vs. restricted bucket |
| 5 | `simulator-mfa.png` | Policy 2 — denied without MFA vs. lifted with MFA |
| 6 | `simulator-time.png` | Policy 3 — business hours vs. after hours |
| 7 | `user-permissions.png` | `policy-test-user` with all customer-managed policies attached |
| 8 | `access-analyzer.png` | Access Analyzer — 0 findings |

*(Screenshots stored in `/screenshots`)*

---

## 💡 Key Concepts Demonstrated

- **Principle of Least Privilege** — action-level, resource-level, and condition-level restriction stacked together
- **IAM Condition Keys** — `aws:MultiFactorAuthPresent`, `aws:CurrentTime`
- **Explicit Deny vs. Implicit Deny** — explicit Deny always overrides any Allow, by design
- **Customer-managed vs. AWS-managed policies** — precision over convenience
- **Policy Simulator-first workflow** — never attach an untested policy to a production identity

---

## 🧰 Tools

AWS Console only — no SDK, no code, no third-party tools. All policies were authored and tested using native AWS IAM features available on the Free Tier.

5. Policy design:-
   Policy 1 — S3-Specific-Bucket-ReadOnly
   Grants s3:GetObject and s3:ListBucket against one explicit bucket ARN instead of Resource: "*".
   ```json
   {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
     "arn:aws:s3:::nikunj-allowed-bucket",
        "arn:aws:s3:::nikunj-allowed-bucket/*"
      ]
     }
   ]
 }```

Policy 2 — Deny-All-Without-MFA
A blanket explicit deny on every action and resource, conditioned on the absence of MFA. BoolIfExists ensures the deny only fires when MFA truly wasn't used.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllWithoutMFA",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
       }
     }
  ]
}```



