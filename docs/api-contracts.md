# CivicFix — REST API Contracts

**Version:** v1 (design)  
**Status:** Proposed / Week 1 Day 3  
**Transport:** HTTPS, JSON, AWS API Gateway HTTP API to Python Lambda

## 1. Conventions

- API base path: `/v1` (stage/domain mapping configured in Terraform).
- Content type: `application/json`, except direct browser-to-S3 uploads.
- Protected routes use `Authorization: Bearer <Cognito access token>`.
- API Gateway validates JWT issuer/audience and suitable access-token scopes; Lambda validates ownership and administrator group/permissions.
- Caller identity is derived from the JWT `sub`; client cannot supply `userId`/`actorId`.
- All timestamps are UTC ISO-8601 strings; `reportId` is a server-generated opaque ID.
- All list endpoints accept bounded `limit` (default 20, maximum 100) and opaque `nextCursor` tokens.
- Use `X-Request-Id` or server-generated request ID in logs and error responses.
- JSON examples use synthetic data and placeholder values.

## 2. Report lifecycle and visibility

Statuses: `SUBMITTED`, `UNDER_REVIEW`, `IN_PROGRESS`, `RESOLVED`, `REJECTED`.

Allowed transitions:

- `SUBMITTED -> UNDER_REVIEW | REJECTED`
- `UNDER_REVIEW -> IN_PROGRESS | REJECTED`
- `IN_PROGRESS -> RESOLVED | UNDER_REVIEW`

`RESOLVED` and `REJECTED` are terminal for MVP.

A newly submitted report starts `visibility: PRIVATE`. An admin may separately publish reviewed content, but the public response never contains reporter identity, private S3 keys, or unreviewed photos. Privacy is enforced server-side.

## 3. Endpoints

| Method | Route | Auth | Purpose |
|---|---|---|---|
| `GET` | `/v1/health` | Public | Simple health endpoint |
| `POST` | `/v1/uploads/presign` | Resident | Request short-lived image upload URL |
| `POST` | `/v1/reports` | Resident | Create report |
| `GET` | `/v1/me/reports` | Resident | List own reports |
| `GET` | `/v1/reports` | Public | Public map/list reports by bbox |
| `GET` | `/v1/reports/{reportId}` | Public/Owner/Admin | Public summary or authorized full details |
| `GET` | `/v1/reports/{reportId}/history` | Public/Owner/Admin | Redacted public history or complete authorized history |
| `GET` | `/v1/admin/reports` | Admin | Work queue by status and month |
| `PATCH` | `/v1/admin/reports/{reportId}/status` | Admin | Transition status atomically |
| `PATCH` | `/v1/admin/reports/{reportId}/visibility` | Admin | Publish/hide reviewed report |

The frontend login and registration workflow is handled by Cognito and is **not** a custom password API.

## 4. Upload image

### POST `/v1/uploads/presign`

Authorization: Resident access token.

**Request:**

```json
{
  "contentType": "image/jpeg",
  "fileSize": 1245832,
  "fileName": "road-damage.jpg"
}
```

**Response `200 OK`:**

```json
{
  "uploadId": "upload_01JTEST",
  "objectKey": "uploads/cognito-sub-example/upload_01JTEST.jpg",
  "uploadUrl": "https://example.invalid/signed-upload-url",
  "requiredHeaders": {"Content-Type": "image/jpeg"},
  "expiresInSeconds": 300
}
```

Constraints: JPEG/PNG/WebP only; maximum 5 MiB; short-lived URL; keys generated server-side under authenticated user's upload prefix. Direct PUT to S3 uses the returned headers. Backend verifies upload ownership/object metadata before attaching it to a report; don't trust the client's `objectKey` alone. Unused uploads expire through an S3 lifecycle policy. In a production system, add content inspection, moderation, and stricter upload controls.

## 5. Create report

### POST `/v1/reports`

Authorization: Resident access token.  
Header: `Idempotency-Key: <client-generated-unique-id>` recommended/required for MVP retries.

**Request:**

```json
{
  "category": "POTHOLE",
  "description": "Large pothole near the intersection, obstructing vehicles.",
  "location": {"latitude": 43.12, "longitude": -79.20},
  "uploadId": "upload_01JTEST"
}
```

`uploadId` is optional. The backend looks up/verifies a corresponding staged upload owned by the caller; no arbitrary photo key is accepted.

**Response `201 Created`:**

```json
{
  "reportId": "rpt_01JTEST000000000000000001",
  "category": "POTHOLE",
  "status": "SUBMITTED",
  "visibility": "PRIVATE",
  "version": 1,
  "createdAt": "2026-11-16T15:00:00.000Z"
}
```

**Validation:** category must be one of six enums; description trimmed, 10–1000 characters; coordinates finite within latitude [-90, 90] and longitude [-180, 180]; demo can additionally enforce Niagara-area supported bounds. Extra/unknown fields rejected. Owner identity comes from JWT.

**Semantics:** Transactionally insert report and creation history. Same idempotency key + same body returns the original outcome; reuse with a different body returns `409 CONFLICT`. Implement a durable idempotency record/check appropriate to the chosen retry horizon.

## 6. List resident reports

### GET `/v1/me/reports?limit=20&nextCursor=...`

Authorization: Resident access token.

**Response `200 OK`:**

```json
{
  "items": [
    {
      "reportId": "rpt_01JTEST000000000000000001",
      "category": "POTHOLE",
      "description": "Large pothole near the intersection, obstructing vehicles.",
      "status": "SUBMITTED",
      "visibility": "PRIVATE",
      "createdAt": "2026-11-16T15:00:00.000Z"
    }
  ],
  "nextCursor": null
}
```

Implementation: Query `UserReportsIndex` using JWT `sub`; descending `createdKey`; never accept `userId` query parameter.

## 7. Public map reports

### GET `/v1/reports?bbox=-79.30,43.08,-79.10,43.20&category=POTHOLE&status=UNDER_REVIEW&limit=20`

Authorization: Public.  
`bbox` format: `minLongitude,minLatitude,maxLongitude,maxLatitude`.  
`category`, `status`, and `nextCursor` optional.

**Response `200 OK`:**

```json
{
  "items": [
    {
      "reportId": "rpt_01JPUBLIC",
      "category": "POTHOLE",
      "descriptionPreview": "Road surface is damaged near a local intersection.",
      "status": "UNDER_REVIEW",
      "approximateLocation": {"latitude": 43.12, "longitude": -79.20},
      "createdAt": "2026-11-16T15:00:00.000Z"
    }
  ],
  "nextCursor": null
}
```

Only admin-approved `PUBLIC` reports. Clamp bounding-box size and cell count. Query geohash cells, apply exact bounding-box check, cap results and paginate across cells. Do not reveal `userId`, exact original coordinates (when sensitive), photo object keys, or raw unmoderated descriptions.

## 8. Report detail and history

### GET `/v1/reports/{reportId}`

- Anonymous requester: public summary only if `visibility = PUBLIC`; otherwise `404 NOT_FOUND`.
- Owner: full own report, even if private.
- Admin: full report with moderation-relevant data.
- Public media must use controlled, short-lived access after server-side visibility checks.

**Response `200 OK` (owner example):**

```json
{
  "reportId": "rpt_01JTEST000000000000000001",
  "category": "POTHOLE",
  "description": "Large pothole near the intersection, obstructing vehicles.",
  "status": "SUBMITTED",
  "visibility": "PRIVATE",
  "location": {"latitude": 43.12, "longitude": -79.20},
  "photoUrl": null,
  "createdAt": "2026-11-16T15:00:00.000Z",
  "updatedAt": "2026-11-16T15:00:00.000Z",
  "version": 1
}
```

`photoUrl` is null when no photo is accessible, otherwise a short-lived authorized URL.

### GET `/v1/reports/{reportId}/history?limit=20&nextCursor=...`

Owner/admin see appropriately authorized status history. Public users see a redacted status timeline for `PUBLIC` reports; actor IDs and private comments are not returned.

**Response `200 OK`:**

```json
{
  "items": [
    {
      "eventType": "CREATED",
      "fromStatus": null,
      "toStatus": "SUBMITTED",
      "changedAt": "2026-11-16T15:00:00.000Z"
    }
  ],
  "nextCursor": null
}
```

## 9. Admin work queue

### GET `/v1/admin/reports?status=SUBMITTED&month=2026-11&limit=20&nextCursor=...`

Authorization: Administrator. `status` required for MVP; `month` defaults to current UTC month; `region` defaults to `NIAGARA`. Optional category filter uses bounded post-query filtering.

Implementation: Query `StatusReportsIndex` using `REGION#NIAGARA#STATUS#SUBMITTED#MONTH#2026-11`, descending `createdKey`. For all-status/month-spanning administration in future, implement multi-bucket merge or separate read model.

**Response `200 OK`:**

```json
{
  "items": [
    {
      "reportId": "rpt_01JTEST000000000000000001",
      "category": "POTHOLE",
      "status": "SUBMITTED",
      "visibility": "PRIVATE",
      "version": 1,
      "createdAt": "2026-11-16T15:00:00.000Z"
    }
  ],
  "nextCursor": null
}
```

## 10. Admin status change

### PATCH `/v1/admin/reports/{reportId}/status`

Authorization: Administrator.

**Request:**

```json
{
  "toStatus": "UNDER_REVIEW",
  "expectedVersion": 1,
  "reason": "Reviewed submission details."
}
```

**Response `200 OK`:**

```json
{
  "reportId": "rpt_01JTEST000000000000000001",
  "status": "UNDER_REVIEW",
  "version": 2,
  "updatedAt": "2026-11-16T15:05:00.000Z"
}
```

Implementation: Check role, validate transition, transactional update conditioned on `version == expectedVersion` plus immutable history insert. Update `statusBucket` to new status. An outdated version or forbidden transition returns `409 CONFLICT` with a suitable application error code.

## 11. Admin visibility change

### PATCH `/v1/admin/reports/{reportId}/visibility`

Authorization: Administrator.

**Request:**

```json
{
  "visibility": "PUBLIC",
  "expectedVersion": 2,
  "reason": "Reviewed and approved for public map."
}
```

A status of `SUBMITTED` cannot be published before review. Validate content/photo moderation readiness. Update visibility, geoCell and history atomically. If visibility is changed to PRIVATE, remove `geoCell`. Public read endpoints must recheck current visibility when necessary because GSI propagation is asynchronous.

## 12. Error contract

All API errors use:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Please provide a valid issue description.",
    "details": [{"field": "description", "issue": "Minimum 10 characters"}],
    "requestId": "req_abc123"
  }
}
```

| HTTP status | Error code | Example |
|---|---|---|
| `400` | `VALIDATION_ERROR` | Invalid category, bbox or coordinates |
| `401` | `UNAUTHORIZED` | Missing/invalid token on protected route |
| `403` | `FORBIDDEN` | Resident attempts admin operation |
| `404` | `NOT_FOUND` | Missing report or inaccessible private report |
| `409` | `CONFLICT` | Version conflict, invalid transition, idempotency-key misuse |
| `413` | `FILE_TOO_LARGE` | Requested upload above 5 MiB |
| `415` | `UNSUPPORTED_MEDIA_TYPE` | Invalid image type |
| `429` | `RATE_LIMITED` | Excessive requests |
| `500` | `INTERNAL_ERROR` | Unexpected server error (no internal details returned) |

For unsupported map area size, use `400 VALIDATION_ERROR` with a helpful message asking the client to zoom in.

## 13. Authorization matrix

| Operation | Anonymous | Resident | Admin |
|---|---|---|---|
| View homepage and public map | Yes | Yes | Yes |
| View approved public report | Yes | Yes | Yes |
| Create report | No | Yes | Yes (if permitted) |
| List own reports | No | Yes | Yes (own only) |
| View private report | No | Owner only | Yes |
| Upload report image | No | Yes | Yes (if permitted) |
| List admin work queue | No | No | Yes |
| Update status or visibility | No | No | Yes |

AWS API Gateway JWT authorizer handles token validation; backend Lambda also verifies required permissions/roles and ownership. The client UI is never an authorization boundary.

## 14. Contract and integration tests

- Valid/invalid payload tests for every write route.
- JWT missing, expired, wrong issuer or insufficient scope.
- Private report must not be retrievable by other residents or anonymous clients.
- Concurrent administrator status updates: one succeeds, stale one returns `409`.
- Status transition rules, history creation and version increment.
- Uploaded object ownership/content constraints.
- Map result redaction and visibility changes, including stale index cases.
- Pagination, limits, malformed cursors and malformed bounding boxes.
- Idempotent repeated report submission.

## References

- API Gateway JWT authorizers: https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html
- DynamoDB Query: https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html
- DynamoDB transactional writes: https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html
