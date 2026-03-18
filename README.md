# GitHub Actions → AWS OIDC Setup (Without Access Keys)

This document explains how to configure **GitHub Actions authentication with AWS using OIDC (OpenID Connect)** to securely deploy applications and push Docker images to Amazon ECR **without storing AWS access keys**.

---

## 📌 Architecture Overview

```
GitHub Actions
      ↓ (OIDC Token)
AWS IAM OIDC Provider
      ↓
IAM Role (GitHubActionsRole)
      ↓
Amazon ECR / AWS Services
```

✅ No long-term credentials
✅ Secure authentication
✅ Recommended by AWS & GitHub

---

## ✅ Prerequisites

* AWS Account access
* GitHub repository or organization
* IAM permissions to create roles and policies
* GitHub Actions enabled

---

## STEP 1 — Create GitHub OIDC Provider in AWS

1. Open **AWS Console**
2. Navigate to:

```
IAM → Identity Providers → Add Provider
```

3. Configure:

| Field         | Value                                       |
| ------------- | ------------------------------------------- |
| Provider type | OpenID Connect                              |
| Provider URL  | https://token.actions.githubusercontent.com |
| Audience      | sts.amazonaws.com                           |

4. Click **Get Thumbprint**
5. Click **Add Provider**

---

## STEP 2 — Create IAM Policy (ECR Access)

Go to:

```
IAM → Policies → Create Policy → JSON
```

Paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:BatchCheckLayerAvailability",
        "ecr:DescribeRepositories"
      ],
      "Resource": "*"
    }
  ]
}
```

Policy Name:

```
GitHubECRPolicy
```

Create policy.

---

## STEP 3 — Create IAM Role for GitHub Actions

Navigate:

```
IAM → Roles → Create Role
```

### Select Trusted Entity

* Trusted entity type: **Web Identity**
* Identity Provider: `token.actions.githubusercontent.com`
* Audience: `sts.amazonaws.com`

Click **Next**.

---

### Attach Permissions

Attach policy:

```
GitHubECRPolicy
```

---

### Role Name

```
GitHubActionsRole
```

Create role.

---

## STEP 4 — Configure Trust Relationship (IMPORTANT)

Open:

```
IAM → Roles → GitHubActionsRole → Trust Relationships → Edit
```

Replace trust policy with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub":
            "repo:NXT-Asseto/*:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Replace:

```
<ACCOUNT_ID>
```

with your AWS Account ID.

---

### 🔐 What This Restricts

* Only repositories under **NXT-Asseto**
* Only workflows running from **main branch**
* Can assume AWS role

---

## STEP 5 — Copy Role ARN

From role summary page copy:

```
arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsRole
```

---

## STEP 6 — Add GitHub Secret

In your repository:

```
Settings → Secrets and variables → Actions → New repository secret
```

Create:

```
AWS_ROLE_ARN
```

Paste Role ARN.

---

## STEP 7 — GitHub Actions Workflow Example

Create:

```
.github/workflows/deploy.yml
```

```yaml
name: Deploy to AWS

on:
  push:
    branches: [ main ]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-south-1
```

---

## STEP 8 — Verify Authentication

Push code to `main` branch.

In GitHub Actions logs you should see:

```
Assuming role with OIDC
Configured AWS credentials
```

---

## ✅ Benefits of OIDC Authentication

* No AWS Access Keys stored in GitHub
* Short-lived credentials
* Secure federation
* Industry best practice CI/CD authentication

---

## 🚨 Common Issues

| Issue                 | Cause                                |
| --------------------- | ------------------------------------ |
| AccessDenied          | Trust policy mismatch                |
| No OIDC token         | Missing `id-token: write` permission |
| Authentication failed | Wrong repo/org name                  |
| Role not assumed      | Branch restriction mismatch          |

---

## 📚 References

* AWS IAM OIDC Documentation
* GitHub Actions OIDC Authentication Guide

---

## ✅ Result

GitHub Actions can now securely authenticate with AWS and deploy infrastructure or push Docker images without static credentials.

---

# *Prepared by*:
*Shaik Moulali*
# *DevOps Engineer*
