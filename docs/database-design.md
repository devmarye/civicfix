# CivicFix — DynamoDB Data Model

**Version:** 1.0  
**Status:** Proposed / Week 1 Day 3  
**Scope:** Niagara Region demonstration; synthetic reports; no official municipal integration

## 1. Design principles

- Design keys for known access patterns; do not rely on table scans in application routes.
- Separate report state from immutable status history.
- Store photo objects in private S3; DynamoDB stores only validated S3 object keys.
- A submission is private until an administrator explicitly approves it for public display.
- Protect owner data and exact/sensitive location details in public responses.
- Use pagination and maximum result limits; indexes are eventually consistent.
- Use conditional/transactional writes to prevent conflicting administrator updates.

## 2. Access patterns

| ID | Use case | DynamoDB operation |
|---|---|---|
| AP-01 | Create report | Conditional `Put` plus initial history `Put` in `TransactWriteItems` |
| AP-02 | Fetch report by ID | `GetItem` on Reports |
| AP-03 | List a resident's reports newest first | Query `UserReportsIndex` |
| AP-04 | List admin reports by status, month and region | Query `StatusReportsIndex` |
| AP-05 | List publicly visible reports for bounded map view | Query `GeoReportsIndex` for cells intersecting bbox, then exact bbox filter |
| AP-06 | Update report status atomically | Transaction: conditional report `Update` + history `Put` |
| AP-07 | Read full status history | Query History by `reportId` |
| AP-08 | Filter by category | Bounded post-query filter (MVP); add purpose-built index if measured demand justifies it |

## 3. Table: CivicFixReports

**Primary key:** `reportId` (String; server-generated UUID/ULID). No sort key.  
**Billing:** On-demand initially.  
**Safety:** Point-in-time recovery for production environment; deletion protection and alarms as appropriate.

| Attribute | Type | Notes |
|---|---|---|
| `reportId` | S | Unique opaque ID |
| `userId` | S | Derived from Cognito `sub` claim; never trusted from request body |
| `category` | S | `POTHOLE`, `SIDEWALK`, `STREETLIGHT`, `DUMPING`, `TREE`, `OTHER` |
| `description` | S | 10–1000 characters after trimming |
| `status` | S | `SUBMITTED`, `UNDER_REVIEW`, `IN_PROGRESS`, `RESOLVED`, `REJECTED` |
| `visibility` | S | `PRIVATE` or `PUBLIC`, admin controlled |
| `location` | M | `{latitude: number, longitude: number}` stored as DynamoDB numbers |
| `photoKeys` | L<S> | 0–1 validated private S3 keys in MVP |
| `regionId` | S | `NIAGARA` for demo |
| `createdAt`, `updatedAt` | S | UTC ISO-8601, e.g. `2026-11-16T15:00:00.000Z` |
| `createdKey` | S | `createdAt#reportId`, stable unique chronological sort key |
| `statusBucket` | S | `REGION#NIAGARA#STATUS#SUBMITTED#MONTH#2026-11`; changes with status |
| `geoCell` | S | Set **only for PUBLIC** reports, geohash length 5; remove when private |
| `version` | N | Optimistic locking counter; starts at 1 |
| `reviewedAt`, `reviewedBy` | S | Optional administrative metadata |

**Sample (conceptual JSON; numbers are DynamoDB N in storage):**

```json
{
  "reportId": "rpt_01JTEST000000000000000001",
  "userId": "cognito-sub-example",
  "category": "POTHOLE",
  "description": "Large pothole near the intersection, obstructing vehicles.",
  "status": "SUBMITTED",
  "visibility": "PRIVATE",
  "location": {"latitude": 43.12, "longitude": -79.20},
  "photoKeys": ["uploads/cognito-sub-example/upload_123.jpg"],
  "regionId": "NIAGARA",
  "createdAt": "2026-11-16T15:00:00.000Z",
  "updatedAt": "2026-11-16T15:00:00.000Z",
  "createdKey": "2026-11-16T15:00:00.000Z#rpt_01JTEST000000000000000001",
  "statusBucket": "REGION#NIAGARA#STATUS#SUBMITTED#MONTH#2026-11",
  "version": 1
}
```

`geoCell` is intentionally absent from unapproved/private reports (sparse public geo index).

### Global secondary indexes

| Index | Partition key | Sort key | Purpose |
|---|---|---|---|
| `UserReportsIndex` | `userId` | `createdKey` | Resident's reports, reverse chronological |
| `StatusReportsIndex` | `statusBucket` | `createdKey` | Admin queue by region/status/month |
| `GeoReportsIndex` | `geoCell` | `createdKey` | Approved reports by map cell |

Use `INCLUDE` projections for safe listing fields only where useful; remember GSI projections are database-level and **not** a substitute for response authorization or redaction. The `GeoReportsIndex` query must never expose `userId`, original photo keys or exact private location through a public API. Prefer a separate public response mapper.

**Status bucket trade-off:** Status-based queues need one Query per requested (status, month) bucket. The admin API can merge bounded pages across recent buckets and sort them by `createdKey`; an initial demo can require a status filter. For a global all-status feed at scale, add a separate feed projection/index or query-oriented read model instead of scanning.

**Category filters:** A DynamoDB `FilterExpression` is applied after key-based reads; page until enough matches or a read budget is reached. Return a continuation cursor rather than implying the first page includes every possible match.

### Map access pattern

1. Validate bounding box and enforce a maximum supported area.
2. Derive the distinct length-5 geohash cells overlapping the box; cap cell count.
3. `Query` `GeoReportsIndex` separately for each cell; use a bounded page size.
4. Filter candidate coordinates against exact bounds; optionally filter category/status.
5. Deduplicate by `reportId`; sort and limit results; encode per-cell pagination state in an opaque signed continuation cursor.
6. For each report returned, recheck public visibility when necessary, and return an approximate location only. Never expose owner identity or private S3 keys.

Geohash indexing is an MVP compromise; it is not a native spatial query. Excessive cells or unbounded queries must be rejected or narrowed. The API can initially return a maximum of 100 visible reports per map request.

## 4. Table: CivicFixReportHistory

**Partition key:** `reportId` (S)  
**Sort key:** `eventKey` (S) = `<UTC timestamp>#<unique event id>`.

| Attribute | Type | Purpose |
|---|---|---|
| `reportId` | S | Report identifier |
| `eventKey` | S | Chronological, unique event key |
| `fromStatus` | S or NULL | Previous state, NULL for creation |
| `toStatus` | S | New state |
| `actorId` | S | Cognito sub of actor |
| `actorRole` | S | `RESIDENT` or `ADMIN` |
| `reason` | S | Optional admin note; treat as potentially sensitive |
| `changedAt` | S | UTC timestamp |
| `eventType` | S | `CREATED`, `STATUS_CHANGED`, `VISIBILITY_CHANGED` |

An initial creation event and report item should be written in the same transaction. When changing status, use `TransactWriteItems` to update the report **with a `version = expectedVersion` condition** and insert a unique history event atomically. Never silently overwrite a concurrent administrator update. The service returns `409 CONFLICT` when the version is outdated.

Suggested transitions:

- `SUBMITTED -> UNDER_REVIEW` or `REJECTED`
- `UNDER_REVIEW -> IN_PROGRESS` or `REJECTED`
- `IN_PROGRESS -> RESOLVED` or `UNDER_REVIEW`
- `RESOLVED`, `REJECTED`: terminal for MVP (later reopening can be a separate feature)

Public visibility is a separate admin action/field, not inferred from report status alone. Only reviewed content should be approved. Making a report private removes `geoCell`; changing from private to public computes and sets `geoCell` atomically with `visibility` and audit history. A briefly stale GSI entry is not permission to disclose a private report; the public API checks the base record where needed.

## 5. Consistency, idempotency and security

- Primary-key `GetItem` can be strongly consistent; GSI queries are eventually consistent.
- Use `TransactWriteItems` for report+history writes. Transactions consume more capacity than simple writes.
- Supply an idempotency token for report creation (client-generated unique request ID, stored/checked server-side); use DynamoDB transaction client tokens only within their documented replay window.
- Derive owner from validated JWT claims; enforce resident ownership for private report reads.
- `GET /reports/{id}` returns only publicly visible data unless the caller is owner or admin.
- S3 bucket has Block Public Access. Use short-lived presigned PUTs with allowed size/content type checks, protected key prefixes, and lifecycle cleanup for unused staging uploads.
- Validate that uploaded objects belong to the submitting principal before persisting their keys. Never trust arbitrary `photoKeys` or an externally supplied `userId`.
- For public photos, serve through a controlled endpoint/short-lived signed read URL after visibility checks; consider moderation and metadata stripping.
- Avoid publishing an exact private residential address or unreviewed photos in map results.
- Log correlation IDs and errors, not raw images, auth tokens or sensitive descriptions.

## 6. Infrastructure to provision in Terraform (Week 2)

- `aws_dynamodb_table` for Reports and History with `PAY_PER_REQUEST` billing.
- Three GSIs on Reports (key schemas above), with appropriate projections.
- Environment-specific names/tags and appropriate PITR settings.
- IAM policies limiting Lambda access to needed tables and indexes.
- CloudWatch metric alarms for errors and throttling.
- S3 private upload bucket (provisioned separately) and lifecycle rules.

## 7. Design trade-offs and validation

- Two tables are easier to reason about for the MVP than a generalized single-table design.
- GSIs improve key-based list retrieval but add storage/write costs and eventual consistency.
- Status-month buckets avoid one unlimited status partition, but complicate all-status queries.
- Geohash cells are simple for a regional demo; reevaluate for large-scale city/region coverage.
- Document and test access patterns with seeded synthetic data before hardening the schema.

## 8. Acceptance checks

- [ ] Report create writes both report and history atomically.
- [ ] Resident can list own reports without table Scan.
- [ ] Admin can list requested status-month buckets.
- [ ] Public map excludes unapproved/private reports.
- [ ] Unauthorized user cannot read private report or photo.
- [ ] Status update cannot bypass allowed transition or optimistic version check.
- [ ] All list endpoints have caps and opaque continuation cursors.
- [ ] Every approved/public issue has an associated geohash cell.

## References

- AWS DynamoDB Query key conditions: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.KeyConditionExpressions.html
- DynamoDB GSI behavior: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html
- DynamoDB transactions: https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html
