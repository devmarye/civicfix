# ADR-001: Use a serverless backend

**Status:** Accepted for MVP design  
**Date:** 2026-11-16

## Context

CivicFix is a portfolio demo with unpredictable, initially low traffic. We want reproducible deployments, limited idle cost and a realistic architecture suitable for civic reporting.

## Decision

Use API Gateway HTTP API, Python Lambda, DynamoDB and private S3 storage.

## Alternatives

- EC2 + relational database: flexible but always-on instances require management and may incur idle cost.
- ECS/Fargate + database: good for long-lived services, but more orchestration than MVP requires.

## Consequences

Advantages: managed scaling, no server management, clear IAM service boundaries. Costs: cold starts, event/timeout considerations, vendor-specific infrastructure, careful DynamoDB access modelling required.

## Review trigger

Revisit if workloads require persistent connections, specialized complex queries, or sustained compute that changes the cost trade-off.
