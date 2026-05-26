# API Contract Negotiation Log
## Pair 10: Access Gate → Core Business (REST Sync)

**Negotiation Period**: 2026-05-20 to 2026-05-26  
**Facilitator**: FIT4110 Teaching Team  
**Provider Representative**: Core Business Team  
**Consumer Representative**: Access Gate Team  

---

## Issue #1: Response Latency SLA and Failure Handling

**Bối cảnh:**
During initial negotiation, Provider proposed a 200ms response time SLA, while Consumer urgently stated that any latency > 100ms would cause user experience issues (people waiting at gates). At the same time, Provider raised concerns about what should happen when the Policy Decision Engine is overloaded or down.

**Vấn đề:**
1. **Latency SLA mismatch**: Provider's 200ms vs Consumer's <100ms P99 requirement
2. **Fail-open vs fail-closed**: If Core Business is unavailable, should Access Gate allow or deny access?

**Đề xuất từ Consumer:**
- < 100ms P99 response time is non-negotiable (users cannot wait at gate)
- Prefer fail-closed (deny) when Core Business unavailable (security priority)
- Provider should implement aggressive caching or edge deployment

**Đề xuất từ Provider:**
- Can achieve 100ms P99 with in-memory policy cache (TTL 5 min) + optimized database indexing
- Fail-closed is acceptable but requires Consumer to manage cached policies for critical corridors

**Quyết định cuối cùng:**
✓ **Accepted SLA: < 100ms P99** measured end-to-end  
✓ **Fail-closed strategy**: Deny access if Core Business unavailable > 5 minutes  
✓ **Provider commits** to in-memory cache with 5-minute TTL and database optimization  
✓ **Consumer commits** to implement local fallback cache for offline resilience

**Rationale (Lý do):**
Access control is safety-critical; fail-closed is more defensible than fail-open. 100ms is aggressive but achievable with proper architecture.

**Tác động đến service:**
- Provider: Must deploy Redis cache layer + implement query optimization + circuit breaker
- Consumer: Must implement local cache + retry logic + graceful fallback
- Testing: Add load tests to verify P99 < 100ms; add chaos tests for failure scenarios
- Monitoring: Add P99 latency monitoring dashboard; alert if > 120ms

**Sign-off:**
- [x] Consumer: Access Gate Team
- [x] Provider: Core Business Team

---

## Issue #2: Idempotency and Duplicate Request Handling

**Bối cảnh:**
Access Gate uses UDP-based transmission for card reader data, which can result in duplicate events. If a card swipe is detected twice due to network retry, will the second attempt create a second access log or be recognized as a duplicate?

**Vấn đề:**
1. **Duplicate detection mechanism**: How does Provider identify duplicates?
2. **Idempotency window**: How long should duplicates be tracked (30 seconds? 1 minute? 1 hour)?
3. **Cached response behavior**: Should Consumer receive the SAME decision if retrying within the window?
4. **Audit trail**: Should duplicate attempts be logged separately or merged?

**Đề xuất từ Consumer:**
- Use `correlationId` (UUID) as deduplication key
- 60-second idempotency window (covers transient retries)
- Return cached decision on duplicate (consistency over freshness)
- Log duplicate attempts separately with `isDuplicate: true` flag for audit

**Đề xuất từ Provider:**
- Support 60-second window but store only in memory (not persistent)
- After 60 seconds, treat as new request even if same correlationId
- If old decision is requested beyond 60s, return 409 Conflict

**Quyết định cuối cùng:**
✓ **Idempotency key**: `correlationId` (UUID) in request body  
✓ **Deduplication window**: 60 seconds  
✓ **Cached response**: Same decision returned for duplicate (deterministic)  
✓ **Audit logging**: Log each attempt with separate audit record + `isDuplicate` field  
✓ **Response on stale duplicate**: HTTP 409 Conflict

**Rationale (Lý do):**
60 seconds balances retry resilience with operational simplicity. In-memory only reduces storage load. Returning cached decision ensures deterministic behavior.

**Tác động đến service:**
- Provider: Add in-memory deduplicate cache (HashMap, 60s TTL)
- Decision API: Add optional `isDuplicate` field for monitoring
- Audit Trail: Add `isDuplicate` boolean to audit record schema
- Testing: Add test for idempotency; add test for 409 on stale id

**Sign-off:**
- [x] Consumer: Access Gate Team
- [x] Provider: Core Business Team

---

## Issue #3: Policy Update Propagation and Cache Invalidation

**Bối cảnh:**
Administrator updates a cardholder's policy (e.g., revokes access). How long before this change takes effect at all Access Gates?

**Vấn đề:**
1. **Cache propagation delay**: If cache TTL is 5 minutes, there's a 5-minute window where gate uses stale policy
2. **User expectation**: Administrator expects revocation to take effect immediately (or within seconds)
3. **Manual cache refresh**: Is there an API to force cache refresh?
4. **Cascading caches**: Gate's local cache + Provider's cache = up to 10 minutes delay

**Đề xuất từ Consumer:**
- Require near real-time propagation (< 5 seconds)
- Provide endpoint to manually invalidate cache: `POST /cache/invalidate`
- Broadcast policy change events so gates can react immediately

**Đề xuất từ Provider:**
- 5-minute TTL is for production stability
- Manual invalidation endpoint is feasible but adds operational complexity
- Compromise: Implement both 5-minute TTL + manual invalidation endpoint

**Quyết định cuối cùng:**
✓ **Cache TTL**: 5 minutes for policy cache  
✓ **Cache invalidation**: Provider implements `POST /cache/invalidate/{policyId}` endpoint  
✓ **Gate responsibility**: Implement local policy cache with max 30-second TTL for critical policies  
✓ **Manual refresh**: Administrators can call invalidation endpoint to force immediate refresh  
✓ **Propagation SLA**: < 5 seconds after manual invalidation; up to 5 minutes for auto-refresh

**Rationale (Lý do):**
5-minute TTL balances performance and freshness. Manual invalidation empowers admins to react urgently. Gates' local cache (30s) prevents cascade failures while allowing reasonable SLA.

**Tác động đến service:**
- Provider: Add `POST /cache/invalidate/{policyId}` + implement timestamp-based invalidation
- Gate: Add 30-second local cache + cache invalidation listener
- Admin Dashboard: Add "Revoke Access Immediately" button
- Monitoring: Track cache hit/miss rates; alert if cache thrashing

**Sign-off:**
- [x] Consumer: Access Gate Team
- [x] Provider: Core Business Team

---

## Issue #4: Cardholder Quota Management (Daily/Monthly Limits)

**Bối cảnh:**
Policies can define access quotas (e.g., "contractor can enter max 20 times per day"). When quota is hit, should access be denied?

**Vấn đề:**
1. **Quota enforcement**: Who tracks quota usage? Provider (stateful) or Consumer (stateless)?
2. **Reset timing**: UTC midnight vs local timezone vs policy-specific time
3. **Quota precision**: Is it "20 accesses within 24 hours" (sliding) or "20 per calendar day"?
4. **Quota visibility**: Can Consumer query remaining quota without making access attempt?
5. **Quota overages**: Should exceeded attempts be logged for audit?

**Đề xuất từ Consumer:**
- Provider tracks quota (stateful); Provider responsible for accuracy
- Use UTC midnight for simplicity
- Sliding window (20 accesses in any 24-hour period)
- Expose quota info in `GET /policies/access/{policyId}` response
- Log all attempts (including quota-exceeded) with `reasonCode: QUOTA_EXCEEDED`

**Đề xuất từ Provider:**
- UTC midnight acceptable but inflexible
- Calendar day (UTC midnight) simpler to implement and audit
- Calendar day preferred over sliding window (simpler)
- Quota query adds database load; prefer returning quota info in access decision response

**Quyết định cuối cùng:**
✓ **Quota tracking**: Provider manages quota state (simplicity)  
✓ **Reset timing**: UTC midnight for calendar day  
✓ **Precision**: Calendar day not sliding window  
✓ **Quota info**: Return `remainingQuota` in `/access/check` response (when access ALLOW)  
✓ **Quota exceeded**: Deny access with `reasonCode: QUOTA_EXCEEDED`

**Rationale (Lý do):**
Provider-managed quota simplifies Consumer logic. UTC midnight is globally understandable. Calendar day easier to audit than sliding window.

**Tác động đến service:**
- Provider: Add `cardholder_quota_usage` table; modify `/access/check` response
- Audit Trail: Add `quotaRemaining` field to decision record
- Admin: Add "Reset cardholder quota" endpoint
- Testing: Test quota reset at midnight boundary; test edge cases

**Sign-off:**
- [x] Consumer: Access Gate Team
- [x] Provider: Core Business Team

---

## Issue #5: Authorization Scopes and Admin API Access

**Bối cảnh:**
Which operations require which authorization scopes? Consumer (Access Gate) needs specific permissions. Admin functions need different permissions. Clear RBAC boundaries must exist.

**Vấn đề:**
1. **Scope definition**: What scopes for each endpoint?
2. **Consumer permissions**: Can Consumer query any decision or only their own?
3. **Admin console**: Separate admin token with higher privileges?
4. **Token rotation**: How often should tokens rotate? Expiry time?

**Đề xuất từ Consumer:**
- Two scope levels:
  - Consumer level: `access:read` (call `/access/check` and `/policies/access/{policyId}`)
  - Admin level: `admin:audit`, `admin:cache`, `admin:quota`
- Consumer can only query audit trail for own gates
- Admin tokens issued separately through secure admin portal
- Token TTL: 24 hours (Consumer), 1 hour (Admin)

**Đề xuất từ Provider:**
- Agree on two-level scopes
- `gateIds` claim in token makes scoping complex; gate-level API keys might be simpler
- Admin token separate from Consumer token (good for audit trail)
- Token TTL: 24 hours (reasonable), 12 hours (admin)

**Quyết định cuối cùng:**
✓ **Scopes**:
  - `access:read`: POST /access/check, GET /policies/access/{policyId}
  - `admin:policies`: POST /cache/invalidate/{policyId}
  - `admin:audit`: GET /decisions/{decisionId}

✓ **Consumer constraints**: JWT claim `gateIds` specifies which gates Consumer can access  
✓ **Admin tokens**: Issued separately via secure portal; includes `role: admin` and `userId` (for audit)  
✓ **Token TTL**: 24 hours (Consumer), 12 hours (Admin)  
✓ **Token format**: JWT with RS256 signature (asymmetric)

**Rationale (Lý do):**
Explicit scopes provide clear permission model. Separating Consumer and Admin tokens enables audit attribution. Gate-level granularity prevents lateral movement.

**Tác động đến service:**
- Provider: JWT scope validation + `gateIds` claim enforcement
- Admin Portal: UI for issuing/revoking admin tokens
- Audit Log: Add `actor` to all mutative operations
- Monitoring: Alert on token expiry, unauthorized scope use, repeated 401 errors

**Sign-off:**
- [x] Consumer: Access Gate Team
- [x] Provider: Core Business Team

---

## Issue #6: API Versioning Strategy and Backward Compatibility

**Bối cảnh:**
Lab 02 defines v1 API. Future requirements may emerge (fingerprint-based access, multi-building policies). How should Provider introduce breaking changes without disrupting existing Access Gates?

**Vấn đề:**
1. **Version location**: URL path vs header vs content negotiation?
2. **Parallel support**: How long should v1 and v2 coexist?
3. **Migration path**: How do gates upgrade?
4. **Deprecation warning**: Should v1 return deprecation headers?
5. **Schema evolution**: Can new fields be added to v1 without breaking gates?

**Đề xuất từ Consumer:**
- Use URL path versioning (`/v1`, `/v2`) for clarity
- Support both v1 and v2 for at least 6 months
- New optional fields in v1 response are OK (gates ignore unknown fields)
- Provide migration guide when v2 released
- Deprecation headers helpful

**Đề xuất từ Provider:**
- URL path versioning agreed
- 6-month support window reasonable
- New optional fields OK; breaking changes require new version
- Supporting two versions doubles testing + maintenance burden

**Quyết định cuối cùng:**
✓ **Version location**: URL path (e.g., `/core/v1/access/check`)  
✓ **Parallel support**: 6 months after v2 release (extendable to 12 if adoption slow)  
✓ **Additive changes**: New optional fields OK without version bump  
✓ **Breaking changes**: Require API version bump  
✓ **Deprecation policy**:
  - 30 days before sunset: Return `Deprecation: true` header
  - 14 days before: Email notification
  - On sunset: v1 returns 410 Gone

✓ **Migration guide**: Provided when v2 released (document differences + checklist)

**Rationale (Lý do):**
URL versioning is explicit. 6-month window reasonable for upgrades. Additive-only changes reduce version explosion. Multiple touch points ensure awareness.

**Tác động đến service:**
- Infrastructure: Deploy dual versions (v1 + v2) on different paths
- Documentation: Separate docs for v1 and v2 with migration guide
- Testing: Both versions; regression tests for v1 during v2 development
- Monitoring: Track API version usage (% on v1 vs v2)
- CI/CD: Backward compatibility tests before v1 deployment

**Sign-off:**
- [x] Consumer: Access Gate Team
- [x] Provider: Core Business Team

---

## Summary Table

| Issue | Decision |
|-------|----------|
| **Latency SLA** | < 100ms P99 with fail-closed strategy |
| **Idempotency** | 60-second window using correlationId |
| **Cache invalidation** | 5-min TTL + manual invalidation endpoint |
| **Quota management** | Provider-tracked, calendar-day reset at UTC midnight |
| **Authorization** | JWT scopes (access:read, admin:*) with gateIds claim |
| **Versioning** | URL path; 6-month parallel support |


---

## Issue #2

- Raised by: Consumer / Provider
- Endpoint:
- Concern:
- Proposal:
- Resolution: Accepted / Rejected / Modified
- Rationale:
- Impact:

---

## Issue #3

- Raised by: Consumer / Provider
- Endpoint:
- Concern:
- Proposal:
- Resolution: Accepted / Rejected / Modified
- Rationale:
- Impact:

---

## Issue #4

- Raised by: Consumer / Provider
- Endpoint:
- Concern:
- Proposal:
- Resolution: Accepted / Rejected / Modified
- Rationale:
- Impact:

---

## Issue #5

- Raised by: Consumer / Provider
- Endpoint:
- Concern:
- Proposal:
- Resolution: Accepted / Rejected / Modified
- Rationale:
- Impact:

---

## Issue #6

- Raised by: Consumer / Provider
- Endpoint:
- Concern:
- Proposal:
- Resolution: Accepted / Rejected / Modified
- Rationale:
- Impact:

---

# Chốt hợp đồng v1.0

Provider sign-off:  
Consumer sign-off:  
Witness (GV/TA):    
Date:               

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
|  |  |  |
