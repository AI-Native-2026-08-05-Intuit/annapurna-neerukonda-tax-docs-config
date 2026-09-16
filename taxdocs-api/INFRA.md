# How this gitops repo is deployed to AWS (CloudFormation) and why.

## Layout

- `cfn/taxdocs-bootstrap-dev.yaml` — artefact S3 bucket + OIDC deploy role (W6 D3 Task 1).
- `.github/workflows/cfn-validate.yml` — cfn-lint + cfn-nag (Task 4, when present).
- Tasks 2–3 templates are not in this PR.

Curriculum names `uptimecrew/taxdocs-config`. This cohort's gitops repo is `AI-Native-2026-08-05-Intuit/annapurna-neerukonda-tax-docs-config`. Bootstrap OIDC `sub` is pinned to that repo.

## Bootstrap stack (`taxdocs-bootstrap-dev`)

Parameters: `EnvName` (dev/staging/prod, default `dev`), `RetentionDays` (7–3650, default 30), `GitHubOrg` + `GitHubRepo` (defaults `uptimecrew` / `taxdocs-config`; pass this cohort’s gitops repo at deploy).

The bucket has Public Access Block on all four toggles, SSE-KMS `alias/aws/s3`, versioning, noncurrent expiry, and a deny of `aws:SecureTransport=false`. **`DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain` are both set.** DeletionPolicy only runs on *stack delete*; UpdateReplacePolicy is what keeps the physical bucket if CloudFormation *replaces* the resource (for example a BucketName change). One without the other still drops artefacts.

The IAM role trusts `sts:AssumeRoleWithWebIdentity` on this account's GitHub OIDC provider, `aud=sts.amazonaws.com`, `sub` `repo:<org>/<repo>:*`. Inline policy: twelve CloudFormation actions on `stack/taxdocs-*` only, plus Get/Put/List on the artefact bucket and `iam:PassRole` on this role. No `Action: '*'`.

Outputs (exported): `BootstrapBucketName`, `BootstrapBucketArn`, `CfnDeployRoleArn`.

## Deploy (ChangeSet CREATE) — CLI

Region: **us-east-1**. Template: `cfn/taxdocs-bootstrap-dev.yaml`. Use `--template-body` (not upload) so CloudFormation does not call `CreateUploadBucket`. Override `GitHubOrg` / `GitHubRepo` to this cohort’s gitops repo. Capability: `CAPABILITY_NAMED_IAM`.

```bash
aws cloudformation create-change-set \
  --region us-east-1 \
  --stack-name taxdocs-bootstrap-dev \
  --change-set-name taxdocs-bootstrap-dev-create \
  --change-set-type CREATE \
  --template-body file://cfn/taxdocs-bootstrap-dev.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=GitHubOrg,ParameterValue=AI-Native-2026-08-05-Intuit \
    ParameterKey=GitHubRepo,ParameterValue=annapurna-neerukonda-tax-docs-config

aws cloudformation describe-change-set \
  --region us-east-1 \
  --stack-name taxdocs-bootstrap-dev \
  --change-set-name taxdocs-bootstrap-dev-create

aws cloudformation execute-change-set \
  --region us-east-1 \
  --stack-name taxdocs-bootstrap-dev \
  --change-set-name taxdocs-bootstrap-dev-create

aws cloudformation wait stack-create-complete \
  --region us-east-1 \
  --stack-name taxdocs-bootstrap-dev

aws cloudformation describe-stacks \
  --region us-east-1 \
  --stack-name taxdocs-bootstrap-dev \
  --query 'Stacks[0].{Status:StackStatus,Outputs:Outputs}'
```

IAM role on this cohort account is `taxdocs-api-cfn-deploy-annapurna-neerukonda` (`CfnDeployRoleName`). Trust: `sts:AssumeRoleWithWebIdentity` and `sub` `repo:AI-Native-2026-08-05-Intuit/annapurna-neerukonda-tax-docs-config:*`.

## Task 1 evidence (2026-09-16, us-east-1, `--profile 668668940354`)

`aws cloudformation describe-stacks --stack-name taxdocs-bootstrap-dev` (SCP blocked a greenfield `CREATE`; live path was import + UPDATE):

```json
{
  "Status": "UPDATE_COMPLETE",
  "Outputs": [
    {
      "OutputKey": "BootstrapBucketArn",
      "OutputValue": "arn:aws:s3:::taxdocs-bootstrap-dev-668668940354",
      "ExportName": "taxdocs-dev-BootstrapBucketArn"
    },
    {
      "OutputKey": "BootstrapBucketName",
      "OutputValue": "taxdocs-bootstrap-dev-668668940354",
      "ExportName": "taxdocs-dev-BootstrapBucketName"
    },
    {
      "OutputKey": "CfnDeployRoleArn",
      "OutputValue": "arn:aws:iam::668668940354:role/taxdocs-api-cfn-deploy-annapurna-neerukonda",
      "ExportName": "taxdocs-dev-CfnDeployRoleArn"
    }
  ]
}
```

Role trust (`taxdocs-api-cfn-deploy-annapurna-neerukonda`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::668668940354:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:AI-Native-2026-08-05-Intuit/annapurna-neerukonda-tax-docs-config:*"
        }
      }
    }
  ]
}
```

## Out of scope (later today / later weeks)

- VPC / subnets / SG — Task 2.
- App stack + Secrets Manager dynamic reference — Task 3 (`taxdocs/dev/db-master` is created out of band; the password never enters YAML).
- ESO / IRSA — still W6 D3 app-side.
- Argo Rollouts — W6 D5.
