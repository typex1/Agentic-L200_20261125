---
inclusion: always
---

# Lab IAM permissions — what works and what returns AccessDenied

You are running inside the AWS lab Code Editor IDE for the AgentCore customer-support
lab. The lab role has a deliberately narrow permission scope. Before you run any AWS
CLI, boto3, or SDK call, check it against the rules below. Prefer actions on the
Allowed list; expect `AccessDenied` (or an explicit `Deny`) for anything on the Denied
list. Do not retry a call that this document says is denied — the policy will not change.

The preinstalled Python environment (3.12) already has: `strands-agents`,
`strands-agents-tools==0.8.12`, `boto3`, `botocore`, `bedrock-agentcore`,
`bedrock-agentcore-starter-toolkit`, `aws-opentelemetry-distro`, `ddgs`, `pyyaml`,
`ipykernel`. A network firewall is enabled. `AWS_DEFAULT_REGION` is preset to the lab
region — do not hardcode a different region.

## Bedrock foundation models

ALLOWED
- `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream`
- ONLY for Amazon Nova Pro:
  - inference profile `us.amazon.nova-pro-v1:0` (use this ID for invocation)
  - foundation model `amazon.nova-pro-v1:0`

DENIED (AccessDenied)
- Any other model: Claude / Anthropic, Titan, Llama, Mistral, other Nova variants
  (nova-lite, nova-micro, nova-canvas, etc.).
- Any Bedrock management or classic-agent action (ListFoundationModels write ops,
  CreateAgent, fine-tuning, provisioned throughput, guardrails, knowledge bases).

When configuring a Strands agent or any Bedrock client, set the model to
`us.amazon.nova-pro-v1:0`. Any other model ID will fail.

## Bedrock AgentCore

ALLOWED — read/list (any resource)
- Gateways: `GetGateway`, `ListGateways`, `GetGatewayTarget`, `ListGatewayTargets`
- Memory: `GetMemory`, `ListMemories`, `ListEvents`, `RetrieveMemoryRecords`
- Workload identity: `GetWorkloadIdentity`, `ListWorkloadIdentities`
- Identity/console reads: `ListConsentPortals`, `ListOauth2CredentialProviders`,
  `ListApiKeyCredentialProviders`, `ListPaymentCredentialProviders`, `GetTokenVault`

ALLOWED — write (constrained; the constraints matter)
- `CreateGateway` — ONLY with `authorizerType = CUSTOM_JWT`. Creating a gateway with
  `authorizerType = NONE` (or no authorizer) is denied. Always wire the Cognito JWT
  authorizer.
- `DeleteGateway` — allowed on gateways in this account/region.
- Gateway targets — `CreateGatewayTarget`, `UpdateGatewayTarget`,
  `DeleteGatewayTarget`, `SynchronizeGatewayTargets` — ONLY on gateways whose name
  starts with `customersupport-gw-`. Name the gateway accordingly or target calls fail.
- `CreateWorkloadIdentity`, `DeleteWorkloadIdentity`
- `CreateMemory`
- `CreateEvent` — ONLY on memory resources named `CustomerSupportMemory-*`.

DENIED (AccessDenied)
- `UpdateGateway` (not granted — build it right the first time; delete and recreate if
  needed).
- `CreateGateway` with a non-JWT authorizer.
- Gateway target actions on any gateway not named `customersupport-gw-*`.
- `CreateEvent` against any memory not named `CustomerSupportMemory-*`.

## Lambda

ALLOWED
- `lambda:GetFunction`, `lambda:InvokeFunction` on the prebuilt `CustomerSupportLambda`.
- `lambda:GetEventSourceMapping`.

DENIED — explicit Deny on `CustomerSupportLambda`
- `UpdateFunctionCode`, `UpdateFunctionConfiguration`, `PublishVersion`,
  `PutFunctionConcurrency`, `PutProvisionedConcurrencyConfig`,
  `PutFunctionEventInvokeConfig`, `UpdateFunctionEventInvokeConfig`,
  `PutFunctionCodeSigningConfig`, `AddPermission`, `CreateFunctionUrlConfig`,
  `UpdateFunctionUrlConfig`.
- Creating new functions is not granted either.

Treat the support Lambda as read-only: view it and invoke it, never modify it. It is
reached from DynamoDB only indirectly, through this Lambda.

## IAM

ALLOWED
- `iam:GetRole`, `iam:GetRolePolicy` on the `GatewayAgentCoreRole` only.
- `iam:PassRole` on `GatewayAgentCoreRole` only, and only to service
  `bedrock-agentcore.amazonaws.com` (so the gateway can assume it).

DENIED (AccessDenied)
- Creating/editing/deleting any role or policy, attaching policies, reading any other
  role, passing any other role or to any other service.

## SSM Parameter Store

ALLOWED
- Read (`GetParameter`, `GetParameters`) under `/app/*` — all prewired config lives
  here (table names, Cognito IDs, OAuth URLs, gateway role ARN, Lambda ARN). Read
  config from SSM instead of hardcoding.
- Write (`PutParameter`, `DeleteParameter`) ONLY under
  `/app/customersupport/*` and `/app/agentcore/*`.

DENIED (AccessDenied)
- Writing parameters outside those two prefixes.

## Cognito

ALLOWED (read-only, used to set up the gateway JWT authorizer)
- `cognito-idp:DescribeResourceServer`, `ListUserPoolClients`, `DescribeUserPoolClient`.

DENIED (AccessDenied)
- Creating user pools/clients, creating or managing users/groups, any write action.

The user pool, app clients, resource server, and OAuth URLs are precreated; read their
IDs/URLs from the `/app/customersupport/agentcore/*` SSM parameters.

## S3

ALLOWED
- Read-only (`Get*`, `List*`, `Describe*`) ONLY on course buckets matching `*-tcprod`.

DENIED (AccessDenied)
- Any S3 write anywhere; any read on buckets not ending in `-tcprod`.

## DynamoDB

- No direct DynamoDB actions are granted. Do NOT call `GetItem`, `Query`, `PutItem`,
  `Scan`, etc. directly — they return AccessDenied. The warranty and customer-profile
  tables are accessed only through `CustomerSupportLambda`.
- The only DynamoDB-adjacent grant is `kms:Decrypt`/`kms:DescribeKey` scoped (via
  `kms:ViaService dynamodb.*.amazonaws.com`) so the service can decrypt on your behalf.

## KMS

ALLOWED
- Encrypt / Decrypt / ReEncryptFrom / ReEncryptTo / GenerateDataKey /
  GenerateDataKeyWithoutPlaintext / DescribeKey / GetKeyPolicy / GetKeyRotationStatus
  on the lab's `GeneralKMSKey`.
- `Decrypt` / `DescribeKey` via the DynamoDB service (see above).

DENIED
- Creating/deleting keys, editing key policies, operations on any other key.

## CloudWatch Logs

ALLOWED
- `CreateLogGroup`, `CreateLogStream`, `DescribeLogStreams`, `PutLogEvents` ONLY under
  log groups `/aws/bedrock-agentcore/*` and `agents/*`.

DENIED
- Log actions on any other log group.

## Hard guardrail — never touch "Enforcer" resources

There is an explicit account-wide Deny on `iam:*`, `lambda:*`, `events:*`,
`dynamodb:*`, and `cloudformation:*` for ANY resource whose name contains `Enforcer`
(roles, policies, functions, EventBridge rules/targets, DynamoDB tables, CloudFormation
stacks). These belong to the lab's grading/enforcement system. Never read, modify, or
delete anything named `*Enforcer*` — it will be denied and may break lab grading.

## Quick decision rule

1. Bedrock invoke → Nova Pro only (`us.amazon.nova-pro-v1:0`).
2. AgentCore gateway → always `CUSTOM_JWT`; name it `customersupport-gw-*`.
3. Lambda → invoke/read `CustomerSupportLambda` only; never modify it.
4. DynamoDB → never call directly; go through the Lambda.
5. SSM → read `/app/*`; write only `/app/customersupport/*` or `/app/agentcore/*`.
6. S3 → read-only on `*-tcprod`.
7. Anything named `*Enforcer*` → never touch.

If a call falls outside these allowances, expect AccessDenied — report it and adjust
rather than retrying.
