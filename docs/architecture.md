# Architecture Notes

## Overview

The stack provisions a KMS CMK with automatic key rotation enabled as the encryption anchor for the entire environment, applied to S3 bucket server-side encryption, RDS storage encryption, and EC2 EBS volumes [from-code]. IAM roles and policies are synthesized by CDK constructs rather than managed through the console, meaning every permission boundary is expressed as TypeScript and tracked in git [from-code]. The environment suffix pattern via CDK context allows the same stack to deploy dev, staging, and prod with differentiated resource naming and removal policies — DESTROY in dev, RETAIN in prod [inferred]. The non-obvious design choice is using a single CMK per environment rather than per-service keys, which simplifies key policy management but creates a shared blast radius if the key policy is misconfigured [editorial].

## Key Decisions

- Single CMK per environment reduces key management overhead but means a misconfigured key policy can lock out S3, RDS, and EBS simultaneously — per-service keys would isolate blast radius at the cost of maintaining a separate key policy per resource type [editorial]
- KMS automatic key rotation rotates the backing key material annually but does not re-encrypt existing data — existing ciphertexts remain decryptable with old material indefinitely, which satisfies most compliance frameworks but surprises engineers expecting full re-encryption [inferred]
- CDK RemovalPolicy.DESTROY on dev KMS keys means a cdk destroy in the wrong environment could make encrypted snapshots permanently unreadable if RDS deletion protection is not separately enforced via a DeletionProtection: true property on the DatabaseInstance construct [inferred]
- Environment suffix via CDK context rather than environment variables means a missing --context environmentSuffix flag at synth time silently falls back to 'dev' defaults, which could deploy dev-grade removal policies against a prod AWS account if the pipeline is misconfigured [from-code]
- Encoding IAM policies in CDK TypeScript gives you type safety and reviewability but means every permission change requires a full CDK deploy cycle of roughly 3-8 minutes — there is no break-glass path for emergency permission grants without drifting from IaC state; SSM Parameter Store or AWS IAM Access Analyzer would need to compensate for that gap [editorial]