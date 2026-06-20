# IAM-Advanced-Policy-Lab
Designed and tested three custom IAM policies — resource-level restriction, MFA enforcement, and time-based access control — then verified them with IAM Policy Simulator and audited the result with IAM Access Analyzer.

# 🔐 AWS IAM Advanced Policy Lab

Designing and testing custom IAM policies with resource-level restrictions, MFA enforcement, and time-based access controls — entirely through the AWS Console.
---

## 📋 Overview

Most beginners rely entirely on AWS managed policies like `ReadOnlyAccess` or `AdministratorAccess`. This project goes further — I designed **three custom IAM policies from scratch**, each demonstrating a different advanced access-control technique used by real cloud security teams:

1. **Resource-level restriction** — limit access to a single named S3 bucket, not all buckets
2. **MFA enforcement via condition keys** — deny every action unless multi-factor authentication was used
3. **Time-based access control** — allow S3 access only during defined business hours

Every policy was validated in the **IAM Policy Simulator** before being attached to a live IAM user, following the same test-before-deploy workflow used in production environments.

---

## 🧱 Architecture
| Architecture Diagram |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/iam_policy_lab_architecture_accurate.png)


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
| 1 | | Create S3 Bucket |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20142815.png)
| 2 | | IAM User Group |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20172911.png)
| 3 | | Policy 1 JSON with resource-level ARN |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20145522.png)
| 4 | | Policy 2 JSON with MFA condition block |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20150714.png
)
| 5 | | Policy 3 JSON with time-based conditions |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20152050.png)
| 6 | | Policy 1 — allowed bucket vs. restricted bucket |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20160248.png)![imale link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20160432.png)
| 7 | | Policy 2 — denied without MFA vs. allowed with MFA |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20164604.png)![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20164505.png)
| 8 | | Policy 3 — business hours vs. after hours |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20165816.png)![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20165849.png)
| 9 | | `policy-test-user` with all customer-managed policies attached |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20172515.png)
| 10 | | Access Analyzer — 0 findings |![image link](https://github.com/nikunjdaa-sit/IAM-Advanced-Policy-Lab/blob/762e8cefc5629cf2c30444e8f2324bf7acc3932b/Screenshot%202026-06-19%20172311.png)


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

