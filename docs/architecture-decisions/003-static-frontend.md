# ADR-003: Export Next.js statically to S3 and CloudFront

**Status:** Accepted with routing constraint  
**Date:** 2026-11-16

## Context

The CivicFix MVP is primarily a client-driven application. API Gateway + Lambda provides all authenticated runtime data and mutation operations.

## Decision

Use Next.js static export (`output: 'export'`) and host generated assets in private S3 via CloudFront OAC. Client-side code calls the API.

## Alternatives

Deploy Next.js on a runtime that supports SSR and dynamic App Router pages. This could support dynamic server routes and more flexible runtime rendering, but increases hosting and operational complexity.

## Consequences

Lower baseline hosting complexity. No server actions, SSR, runtime route handlers or unknown dynamic `[id]` pages. Use static routes such as `/reports/view?id=...` initially; revisit when richer routing is necessary. CloudFront must be configured correctly for static route filenames.

## Review trigger

Move to full Next.js server hosting when server-side rendering, richer SEO for report pages, or authenticated server sessions are mandatory.
