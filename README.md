# GitHub Actions → AWS OIDC Demo

A minimal demo of how GitHub Actions authenticates to AWS using OpenID Connect (OIDC).

This demo does **not** store an AWS Access Key ID or Secret Access Key in GitHub Secrets. Instead, the workflow exchanges a short-lived GitHub-issued token for temporary AWS credentials.

```text
GitHub Actions
      ↓
GitHub OIDC token (JWT, sub + aud claims)
      ↓
AWS STS (AssumeRoleWithWebIdentity)
      ↓
IAM role trust policy evaluated (aud, sub)
      ↓
Temporary AWS credentials
      ↓
AWS CLI commands
```

The workflow runs:

```bash
aws sts get-caller-identity
aws s3 ls
```

---

## Why OIDC?

The traditional approach is to create an IAM user and store its keys in GitHub Secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

These are long-lived credentials. If they leak, they remain valid until someone rotates or disables them, which is a manual, ongoing burden.

With OIDC, no AWS key is stored anywhere. Each workflow run receives a short-lived token that AWS validates against a trust policy, and AWS issues temporary credentials that expire on their own.

---

## How It Works

```mermaid
sequenceDiagram
    autonumber

    participant GH as GitHub Actions
    participant OIDC as GitHub OIDC Provider
    participant STS as AWS STS
    participant ROLE as IAM Role
    participant S3 as Amazon S3

    GH->>OIDC: Request OIDC token (id-token: write)
    OIDC-->>GH: Signed JWT with sub and aud claims

    GH->>STS: AssumeRoleWithWebIdentity (JWT)
    STS->>STS: Verify JWT signature and issuer
    STS->>ROLE: Evaluate trust policy conditions (aud, sub)

    alt Conditions match
        ROLE-->>STS: Trust allowed
        STS-->>GH: Temporary credentials
        GH->>S3: aws s3 ls
        S3-->>GH: Bucket list
    else Conditions do not match
        STS-->>GH: Not authorized
    end
```

### Flow summary

1. The job starts, and GitHub issues it an OIDC JWT. The workflow needs `id-token: write` to request one.
2. `aws-actions/configure-aws-credentials` sends the JWT to STS and asks to assume the IAM role.
3. STS verifies the token's signature and issuer, then evaluates the role's trust policy. The `aud` and `sub` conditions decide whether this exact repository, branch, or environment may assume the role.
4. If the conditions match, STS returns temporary credentials for the AWS CLI commands.

---

## The `sub` Claim

The trust policy must match the `sub` claim GitHub puts in the token. Its format depends on the repository and the job:

| Job type | Format |
|---|---|
| Branch, repo with IDs | `repo:<ORG>@<ORG_ID>/<REPO>@<REPO_ID>:ref:refs/heads/<BRANCH>` |
| GitHub environment, repo with IDs | `repo:<ORG>@<ORG_ID>/<REPO>@<REPO_ID>:environment:<ENV_NAME>` |
| Branch, older repo without IDs | `repo:<ORG>/<REPO>:ref:refs/heads/<BRANCH>` |

Repositories created on or after **15 July 2026** include the owner and repository IDs by default. Older repositories omit them unless opted in. See the [aws-actions/configure-aws-credentials OIDC documentation](https://github.com/aws-actions/configure-aws-credentials#claims-and-scoping-permissions) and the [GitHub changelog on immutable subject claims](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/).

**Don't guess the format.** Print the `sub` from a real run (see Troubleshooting) and copy it into the trust policy.

To get the IDs:

```bash
gh api users/<ORG> --jq .id              # or orgs/<ORG> for an organization
gh api repos/<ORG>/<REPO> --jq .id
```

---

## Repository Setup

### Step 1: Create the OIDC identity provider in AWS

In the AWS console, go to **IAM → Identity providers → Add provider** and choose **OpenID Connect**:

| Field | Value |
|---|---|
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

AWS manages certificate validation for GitHub's OIDC endpoint, so you don't need to rotate a thumbprint as routine maintenance.

### Step 2: Create the IAM role

Create a role named `S3-ROLE` (or your choice) with the OIDC provider as the trusted entity. Use this trust policy, replacing the placeholders:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<ORG>@<ORG_ID>/<REPO>@<REPO_ID>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

- **`aud`** ensures the token was issued for AWS STS.
- **`sub`** ensures only the specified repository and branch (or environment) can assume the role. Avoid wildcards such as `repo:*`, which let any repository assume the role.

For a job that uses `environment: production`, change the `sub` to `repo:<ORG>@<ORG_ID>/<REPO>@<REPO_ID>:environment:production`.

### Step 3: Attach permissions to the role

For this demo, you can list all buckets with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    }
  ]
}
```

This is the broadest option and is acceptable only for a demo. The better pattern is to scope access to a single bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME"
    }
  ]
}
```

Follow least privilege and grant only what your workflow needs.

### Step 4: (Optional) Create a GitHub environment

In your repository, go to **Settings → Environments → New environment** and create `production`. Add protection rules such as required reviewers and a deployment branch restricted to `main`. If you use this, the job must declare `environment: production` and the trust policy `sub` must use the `environment:` format above.

### Step 5: Add the workflow

Create `.github/workflows/deploy.yml` (see the file in this repository).

---

## Expected Output

```text
$ aws sts get-caller-identity
{
    "UserId": "AROA...:GitHubActions-<run_id>",
    "Account": "393036065076",
    "Arn": "arn:aws:sts::393036065076:assumed-role/S3-ROLE/GitHubActions-<run_id>"
}

$ aws s3 ls
2026-10-01 10:20:30 demo-bucket
```

If the account has no buckets, `aws s3 ls` returns nothing and exits successfully. That is expected, not an error.

---

## Security Notes

- **No stored AWS keys.** Don't add `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` to GitHub Secrets for this workflow.
- **Minimal permissions.** The workflow requests only `id-token: write` and `contents: read`.
- **Restrictive `sub`.** Scope it to a specific repository and branch or environment, never a wildcard.
- **Stable IDs help.** The ID-based `sub` means a repo deleted and recreated under the same name gets different IDs and cannot assume the role.
- **Environment protection.** Use required reviewers and branch restrictions for production roles.
- **Pin actions.** This demo uses version tags. For production, pin each action to a full commit SHA.

---

## Troubleshooting

**`Not authorized to perform sts:AssumeRoleWithWebIdentity`**
The trust policy `sub` doesn't match the token. Print the real `sub` from the job and compare it character by character with the trust policy. Check the owner and repo IDs, the branch or environment, and that the role is in the account you expect.

To print the claims during debugging (remove the step afterward, since it exposes claim details in logs):

```yaml
- name: Print OIDC claims (temporary)
  run: |
    TOKEN=$(curl -sSL -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=sts.amazonaws.com" | jq -r '.value')
    echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' \
      | awk '{ while (length($0) % 4) $0 = $0 "="; print }' \
      | base64 -d 2>/dev/null | jq '{sub, aud, iss}'
```

**Wrong account**
The role ARN, the trust policy `Federated` principal, and the OIDC provider must all reference the same account ID. An OIDC provider exists only in the account where it was created.

**`Credentials could not be loaded` or no OIDC token**
The job is missing `id-token: write` in its `permissions` block.

**`get-caller-identity` works but `s3 ls` fails with AccessDenied**
OIDC authentication succeeded. Check the permissions policy attached to the role.

**Job stuck or skipped**
If the environment has required reviewers, the job waits for approval.

---

## Official Documentation

- [OpenID Connect concepts](https://docs.github.com/en/actions/concepts/security/openid-connect)
- [Configuring OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
- [GitHub changelog: immutable subject claims for Actions OIDC tokens](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/)
- [Security hardening for deployments](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments)
