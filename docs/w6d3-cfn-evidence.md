# W6 D3 CloudFormation evidence (2026-09-16, us-east-1, profile 668668940354)

Shared-sandbox stacks already existed. Statuses are `UPDATE_*`, not a second greenfield `CREATE`.

## Live stacks

| Stack | Status |
|---|---|
| taxdocs-bootstrap-dev | UPDATE_COMPLETE |
| taxdocs-network-dev | UPDATE_COMPLETE |
| taxdocs-app-dev | UPDATE_ROLLBACK_COMPLETE (RDS outputs still exported) |
| taxdocs-artifacts-dev | UPDATE_COMPLETE |

## T4 — detect-stack-drift (bootstrap lifecycle 30 → 7 → 30)

DRIFTED:

```json
{
  "StackDriftDetectionId": "94595670-b1f6-11f1-b60d-12af382b91fd",
  "StackDriftStatus": "DRIFTED",
  "DetectionStatus": "DETECTION_COMPLETE",
  "DriftedStackResourceCount": 1
}
```

IN_SYNC after revert:

```json
{
  "StackDriftDetectionId": "972c73a0-b1f6-11f1-9b87-1283a6aa582f",
  "StackDriftStatus": "IN_SYNC",
  "DetectionStatus": "DETECTION_COMPLETE",
  "DriftedStackResourceCount": 0
}
```

## T4 — network delete refused

```
Delete canceled. Cannot delete export taxdocs-network-dev-VpcId as it is in use by taxdocs-app-dev.
```

## T4 — UPDATE ChangeSets (deleted without execute)

Artifacts — no replacement:

```
ArtifactsBucketPolicy  Modify  Replacement=False
ArtifactsBucket        Modify  Replacement=False
```

Bootstrap — no replacement:

```
BootstrapBucketPolicy  Modify  Replacement=False
BootstrapBucket        Modify  Replacement=False
CfnDeployRole          Modify  Replacement=False
```

App — no replacement:

```
DbInstance       Modify  Replacement=False
DbSubnetGroup    Modify  Replacement=False
RdsSecurityGroup Modify  Replacement=False
```

Network (do not execute): AppSecurityGroup `Replacement=True` vs the shared live stack.
