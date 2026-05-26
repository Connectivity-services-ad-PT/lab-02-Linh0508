# API Versioning Strategy
## Core Business — Policy Decision Engine API

---

## 1. Versioning Strategy Selected: URL Path Versioning

**Approach**: Each API version is deployed on a distinct URL path.

```
v1:   https://api.campus.local/core/v1
v2:   https://api.campus.local/core/v2
(future: https://api.campus.local/core/v3, etc.)
```

**Rationale**:
- **Explicit and discoverable**: Consumers know exactly which version they're using by reading the URL
- **Independent deployments**: v1 and v2 can be deployed/scaled independently
- **Clear in logs and monitoring**: Request logs show version without parsing headers
- **Backward compatibility**: Old clients continue hitting v1 without code changes
- **No content negotiation complexity**: Unlike header-based or Accept-based versioning, no negotiation logic needed

---

## 2. Concrete Examples of Version Usage

### Example 1: Access Check Endpoint

**v1 URL**:
```
POST https://api.campus.local/core/v1/access/check
```

**v2 URL** (hypothetical future version):
```
POST https://api.campus.local/core/v2/access/check
```

### Example 2: Health Check Endpoint

Both versions expose the same operational endpoints:
```
GET https://api.campus.local/core/v1/health
GET https://api.campus.local/core/v2/health
```

### Example 3: Complete Request with Version

```bash
curl -X POST https://api.campus.local/core/v1/access/check \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "cardId": "EMP001234",
    "gateId": "LOBBY_01",
    "correlationId": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

---

## 3. Version Compatibility Matrix

| Feature | v1 | v2 | v3 |
|---------|----|----|-----|
| **Core endpoints** | ✓ | ✓ | ✓ |
| Basic authentication | ✓ | ✓ | ✓ |
| Biometric auth | ✗ | ✓ | ✓ |
| Multi-building support | ✗ | ✗ | ✓ |
| Real-time policy cache | ✓ | ✓ | ✓ |
| Audit retention (days) | 30 | 60 | 90 |

---

## 4. Backward Compatibility Policy

### 4.1 Additive Changes (No Version Bump Required)

These changes can be added to a version without breaking consumers:

1. **New optional fields in request**
   - Example: Adding optional `temperature` field to `/access/check` request
   - Consumer can omit it; Provider has default behavior
   - Version: **Same** (no bump)

2. **New optional fields in response**
   - Example: Adding `processingTimeMs` field to access decision
   - Consumer can ignore unknown fields (JSON parser should allow extra keys)
   - Version: **Same** (no bump)

3. **New optional query parameters**
   - Example: Adding `?includeAuditTrail=true` to GET endpoints
   - Default behavior unchanged if parameter omitted
   - Version: **Same** (no bump)

4. **New endpoints in same version**
   - Example: Adding `/cache/invalidate` to v1
   - Doesn't break existing endpoints
   - Version: **Same** (no bump)

### 4.2 Breaking Changes (Version Bump Required)

These require a new API version:

1. **Removing or renaming required fields**
   - Example: Changing `cardId` to `employeeId` (breaking change)
   - Consumers expecting `cardId` will break
   - Version: **New** (e.g., v1 → v2)

2. **Changing field type**
   - Example: Changing `processingTimeMs` from integer to string
   - Consumer code expecting integer will break
   - Version: **New**

3. **Removing or changing enum values**
   - Example: Changing decision enum from `[ALLOW, DENY]` to `[PERMIT, REJECT]`
   - Consumer code checking for `ALLOW` will break
   - Version: **New**

4. **Changing HTTP status codes**
   - Example: Changing `/access/check` from always returning 200 to returning 202 on pending decisions
   - Consumers expecting 200 will fail
   - Version: **New**

5. **Changing security scheme**
   - Example: Switching from Bearer tokens to API keys
   - Consumers with Bearer setup will break
   - Version: **New**

6. **Removing or significantly changing endpoint behavior**
   - Example: `/decisions/{decisionId}` changing from returning cached decision to real-time evaluation
   - Consumers relying on deterministic responses will break
   - Version: **New**

---

## 5. Semantic Versioning Scheme

Versions follow pattern: `vMAJOR.MINOR.PATCH` (e.g., v1.2.3)

### Major Version (v1 → v2 → v3)
- **Triggers**: Breaking changes
- **Consumer action**: Code refactoring required; must update endpoints, schemas
- **Release cycle**: Every 18–24 months (or less if major requirements emerge)

### Minor Version (v1.0 → v1.1 → v1.2)
- **Triggers**: Backward-compatible new features (additive changes)
- **Consumer action**: Optional upgrade; can continue using v1.0 if newer features not needed
- **Release cycle**: Every 3–6 months

### Patch Version (v1.2.0 → v1.2.1 → v1.2.2)
- **Triggers**: Bug fixes, security patches, performance improvements (no API changes)
- **Consumer action**: Highly recommended (security patches); no code changes needed
- **Release cycle**: As needed (typically within 2 weeks of discovery)

**Note for Lab 02**: Lab 02 delivers **v1.0.0**. Future labs (Lab 03, etc.) may introduce v1.1.0, v1.2.0, or v2.0.0 depending on requirements.

---

## 6. Release Process for New Versions

### Step 1: Negotiation & Planning (4–8 weeks before release)
- Consumer and Provider teams meet to discuss breaking changes
- Document all breaking changes in release notes
- Agree on timeline and migration deadline

### Step 2: Development & Testing (4–12 weeks)
- Provider develops new version on separate branch: `feature/v2-biometric`
- Comprehensive test coverage (unit tests, integration tests, acceptance tests)
- Consumer teams prepare upgrade code (but don't deploy yet)

### Step 3: Pre-Release (2 weeks before go-live)
- Deploy new version (e.g., v2) to staging environment
- Consumers run end-to-end testing against staging
- Fix any integration issues discovered
- Finalize documentation and migration guides

### Step 4: Parallel Deployment (Go-Live Day)
- Deploy v2 to production alongside v1
- Both versions serve traffic simultaneously
- Monitor v1 and v2 metrics for stability
- Activate deprecation warnings on v1 responses

### Step 5: Migration Phase (Starts day 0, ends day 180)
```
Days 1–30:      Stable parallel operation; no rush to migrate
Days 31–90:     Gradual migration; gates begin upgrading
Days 91–150:    Most gates upgraded; legacy gates still supported
Days 151–180:   Final stragglers migrate
Day 181:        Sunset v1 (if 100% migrated; otherwise extend)
```

### Step 6: Sunset (After migration complete)
- Start returning deprecation headers 30 days before shutdown
- Send email notifications 14 days before shutdown
- On shutdown date: v1 endpoints return HTTP **410 Gone**
- Detailed sunset log kept in `/docs/v1-sunset-report.md`

---

## 7. Deprecation Policy

### Deprecation Timeline

```
Day 0:   v2.0.0 released; v1.0.0 enters "supported with deprecation notice"
Day 1–29:  v1 continues normal operation; consumers given 30 days to upgrade
Day 30:    Deprecation header added to v1 responses
           Deprecation: true
           Sunset: Sun, 26 Nov 2026 00:00:00 GMT
           
Day 31–44: Email reminder sent to all registered consumers
Day 45:    Final reminder email
Day 60–150: Migration window; gates upgrade to v2
Day 151:   Final deadline approached; last 5% of gates upgrade
Day 180:   Sunset date
Day 181+:  v1 returns 410 Gone; consumers must use v2
```

### Deprecation Header Format

RFC 8594 compliant deprecation headers added to v1 responses:

```
HTTP/1.1 200 OK
Deprecation: true
Sunset: Mon, 27 May 2027 00:00:00 GMT
Link: </core/v2/access/check>; rel="successor-version"
```

### Sunset Header Explanation

| Header | Meaning | Example |
|--------|---------|---------|
| **Deprecation** | This version is deprecated | `true` |
| **Sunset** | Date when v1 stops accepting requests | `Mon, 27 May 2027 00:00:00 GMT` |
| **Link** | URL of successor version | `</core/v2/access/check>; rel="successor-version"` |

---

## 8. Example: Upgrade from v1 to v2

### What Changed in v2?

**Scenario**: v2 adds biometric authentication support and renames `cardId` to `credentialId` (breaking change).

### v1 Request
```bash
curl -X POST https://api.campus.local/core/v1/access/check \
  -H "Authorization: Bearer ..." \
  -H "Content-Type: application/json" \
  -d '{
    "cardId": "EMP001234",
    "gateId": "LOBBY_01",
    "correlationId": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

### v2 Request (Backward Incompatible)
```bash
curl -X POST https://api.campus.local/core/v2/access/check \
  -H "Authorization: Bearer ..." \
  -H "Content-Type: application/json" \
  -d '{
    "credentialId": "EMP001234",            # Changed from cardId
    "credentialType": "RFID",               # New required field
    "gateId": "LOBBY_01",
    "correlationId": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

### Migration Checklist for Consumer (Gate)

- [ ] Update card reader driver to populate `credentialType: "RFID"`
- [ ] Update request body to use `credentialId` instead of `cardId`
- [ ] Update response parsing (no schema changes in v2, but good to verify)
- [ ] Test against v2 staging endpoint: `https://api.campus.local/core/v2/access/check`
- [ ] Verify failover logic (gate can retry v2, fallback to v1 if needed)
- [ ] Update monitoring dashboards to track both `/v1` and `/v2` endpoints
- [ ] Deploy updated gate code to production
- [ ] Verify traffic flows to v2 endpoint
- [ ] Monitor error rates for 1 week

---

## 9. Client Upgrade Patterns

### Pattern A: Aggressive Upgrade (Recommended for new deployments)
- Gate deployed after v2 release date uses v2 directly
- No v1 support needed
- Simplifies maintenance

### Pattern B: Graceful Fallback (Recommended for existing gates)
- Gate defaults to v2
- On 503 / 404 errors from v2, automatically retry v1
- Provides resilience during transition
- Eventually all gates upgrade; v1 retries stop

```java
// Pseudocode for fallback logic
try {
    response = callAPI("v2/access/check", request);
} catch (HTTP_404 | HTTP_503) {
    // v2 not available; fall back to v1
    log.warn("v2 failed; retrying on v1");
    response = callAPI("v1/access/check", request);
}
```

### Pattern C: Dual-Write (For zero-downtime switch)
- Gate sends request to both v1 and v2 in parallel
- Uses whichever responds faster
- High complexity but provides hedge
- Not recommended unless v1/v2 responses differ

---

## 10. Monitoring Version Adoption

### Metrics to Track

```
// Prometheus / Grafana dashboard queries

access_gate_v1_requests_total        # v1 request count
access_gate_v2_requests_total        # v2 request count
access_gate_v1_request_duration_p99  # v1 P99 latency
access_gate_v2_request_duration_p99  # v2 P99 latency
access_gate_v1_error_rate            # v1 errors per minute
access_gate_v2_error_rate            # v2 errors per minute
```

### Dashboard Alerts

- Alert if v2 error rate > 5% (indicates migration issue)
- Alert if v1 traffic > 30% (slow migration; extend support or investigate)
- Alert if v1 P99 latency > v2 P99 + 100ms (v1 degradation; might warrant early shutdown)

---

## 11. Version Lifecycle Summary

```
┌─────────────┬────────────────┬──────────────┬──────────┐
│ Phase       │ Duration       │ Support      │ Actions  │
├─────────────┼────────────────┼──────────────┼──────────┤
│ Active      │ 12–24 months   │ Full         │ Receive patches, minor updates │
│ Maintenance │ 6 months       │ Critical     │ Security patches only |
│ Deprecated  │ 6 months       │ Deprecation  │ Emit warnings; no new features │
│ Sunset      │ ~1 day         │ None (410)   │ Stop accepting requests │
│ Archived    │ Indefinite     │ None         │ Documentation preserved |
└─────────────┴────────────────┴──────────────┴──────────┘
```

**Total Lifetime**: ~24–42 months per major version

---

## 12. FAQ

**Q: How do I know which version to use?**  
A: New consumers should use the latest version. Existing consumers can upgrade at their own pace.

**Q: Can I use v1 and v2 simultaneously?**  
A: Yes, they run in parallel for 6 months. Upgrade when ready.

**Q: What happens if I don't upgrade before v1 sunset?**  
A: v1 endpoints return HTTP 410 Gone. Consumer code will break; you must upgrade.

**Q: Is there a v0 (experimental)?**  
A: No. Lab 02 delivers v1.0.0 production-ready.

**Q: Can I use v1.5.0 (minor version without major)?**  
A: Only if v1.5.0 exists and v1.0.0 support has ended. Otherwise, use v1.0.0.

---

## 13. References

- [RFC 8594: Deprecation HTTP Header](https://tools.ietf.org/html/rfc8594)
- [Semantic Versioning 2.0.0](https://semver.org)
- [OpenAPI 3.1.0 Specification](https://spec.openapis.org/oas/v3.1.0)
