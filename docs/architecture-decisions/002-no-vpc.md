# ADR-002: Do not attach Lambda functions to a customer VPC for the MVP

**Status:** Accepted for MVP design  
**Date:** 2026-11-16

## Context

Backend Lambdas access managed AWS services (DynamoDB and S3). No private RDS instance or internal network resource needs to be reached.

## Decision

Keep MVP Lambdas outside a customer-managed VPC. Enforce permissions with IAM and resource policies.

## Alternatives

Private subnet Lambda with NAT gateway and endpoints. This can be justified for resources requiring private network access, but adds cost and configuration without improving the present user flow.

## Consequences

Simpler setup and lower baseline networking cost. We cannot directly access resources that are only available in a private VPC until networking is added.

## Review trigger

Add a VPC architecture when private database connectivity, internal service access, or explicit egress requirements demand it.
