# How this gitops repo is deployed to AWS (CloudFormation) and why.

## Layout

- `cfn/taxdocs-bootstrap-dev.yaml` — artefact S3 bucket + OIDC deploy role (W6 D3 Task 1).
- `cfn/taxdocs-network-dev.yaml` — 3-AZ VPC + NAT Conditions + app SG (Task 2).
- `cfn/taxdocs-app-dev.yaml` — RDS + `!ImportValue` + Secrets Manager password (Task 3).
- `cfn/taxdocs-artifacts-dev.yaml` — hardened `uptimecrew-taxdocs-artifacts-dev` (Task 3).
- `.github/workflows/cfn-validate.yml` — cfn-lint + cfn-nag (Task 4, when present).

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

## Network stack (`taxdocs-network-dev`)

Parameters: `EnvName` (dev/staging/prod), `VpcCidr` (AllowedPattern, default `10.40.0.0/16`).

`IsProdLike` is `staging` or `prod`; `IsDev` is the inverse. **Why Conditions for NAT:** one NAT in public-A is enough for capstone cost in `dev` (private B/C share that NAT). Staging/prod create `NatGatewayBPerAz` / `NatGatewayCPerAz` so each AZ has its own NAT and a private route table (HA; no cross-AZ NAT after an AZ loss).

Subnets: 3 public + 3 private, `!Select [n, !GetAZs ""]`, CIDRs `!Cidr [!Ref VpcCidr, 8, 8]`. App SG: ingress tcp/8080 from the VPC CIDR only (never `0.0.0.0/0`); egress tcp/443 to the internet. tcp/5432 to the RDS SG is Task 3 (`AWS::EC2::SecurityGroupEgress` + `!ImportValue taxdocs-network-dev-AppSgId`) so this stack can exist first.

Exports: `taxdocs-network-dev-VpcId`, `…-PublicSubnets`, `…-PrivateSubnets`, `…-AppSgId`.

This cohort already has a live shared stack `taxdocs-network-dev` (`UPDATE_COMPLETE`, VPC `vpc-05555553da6d76dfd` / `10.40.0.0/16`). **Do not CREATE a second stack** — export names would collide. Do not UPDATE the shared VPC unless you intend to replace classmates’ attachments. Paste `describe-stacks` / `list-exports` as Done-when evidence.

```bash
aws cloudformation describe-stacks --profile 668668940354 --region us-east-1 \
  --stack-name taxdocs-network-dev --query 'Stacks[0].{Status:StackStatus,Outputs:Outputs}'

aws ec2 describe-vpcs --profile 668668940354 --region us-east-1 \
  --filters Name=cidr,Values=10.40.0.0/16 --query 'Vpcs[].{Id:VpcId,Cidr:CidrBlock}'

aws cloudformation list-exports --profile 668668940354 --region us-east-1 \
  --query "Exports[?starts_with(Name, 'taxdocs-network-dev-')].Name"
```

## App + artefacts stacks (`taxdocs-app-dev`, `taxdocs-artifacts-dev`)

App stack **never hardcodes subnet IDs**. `DbSubnetGroup` uses `!Split [",", !ImportValue taxdocs-network-dev-PrivateSubnets]`; RDS ingress and `AppToRdsEgress` use `!ImportValue taxdocs-network-dev-AppSgId` / `VpcId`. After a network rebuild, CFN re-resolves those exports.

**Why dynamic reference vs NoEcho:** `NoEcho: true` hides the password in the Console but it still lands in the template parameter, change-set JSON, and often stack events. `{{resolve:secretsmanager:taxdocs/${EnvName}/db-master:SecretString:password}}` keeps the secret in Secrets Manager; the YAML has no password. The secret is created out of band (`taxdocs/dev/db-master` already exists in this account).

Artefact bucket `uptimecrew-taxdocs-artifacts-dev`: PAB all four, SSE-KMS `alias/aws/s3`, versioning, 90d → STANDARD_IA then 365d → GLACIER_IR, deny `aws:SecureTransport: false`, both Retain policies.

Live (do not CREATE a second copy): `taxdocs-app-dev` has RDS outputs (`UPDATE_ROLLBACK_COMPLETE` after a later failed update — instance still exported). `taxdocs-artifacts-dev` is `UPDATE_COMPLETE`. Delete of `taxdocs-network-dev` was refused: `Cannot delete export taxdocs-network-dev-PrivateSubnets as it is in use by taxdocs-app-dev` (stack stayed `UPDATE_COMPLETE`).

```bash
aws cloudformation describe-stacks --profile 668668940354 --region us-east-1 \
  --stack-name taxdocs-app-dev --query 'Stacks[0].{Status:StackStatus,Outputs:Outputs}'

aws s3api get-public-access-block --profile 668668940354 --bucket uptimecrew-taxdocs-artifacts-dev

aws s3api get-bucket-policy --profile 668668940354 --bucket uptimecrew-taxdocs-artifacts-dev
```

## Out of scope (later today / later weeks)

- ESO / IRSA — still W6 D3 app-side.
- Argo Rollouts — W6 D5.
