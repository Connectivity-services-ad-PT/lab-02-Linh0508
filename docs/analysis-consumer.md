# Phân tích yêu cầu — vai Consumer

- **Cặp đàm phán**: Pair 10: Access Gate → Core Business
- **Loại**: REST sync
- **Consumer service**: Access Gate (gọi API để check quyền)
- **Provider service**: Core Business (cung cấp policy decision)
- **Người viết**: Nguyễn Thị Thuỳ linh
- **Ngày**: 2026-05-26

---

## 1. Business Problem cần giải quyết

**Access Gate** cần giải quyết vấn đề:
- Người dùng quẹt thẻ RFID tại cổng → Gate cần **quyết định ngay** (< 100ms) cho phép hay chặn
- Gate không thể lưu toàn bộ policy (hàng ngàn quy tắc) → phải gọi Core Business real-time
- Nếu network lag → người dùng bị kẹt tại cổng → user experience tệ
- Nếu API không trả về quyết định deterministic → dẫn đến xung đột, lỗi bảo mật

**Business outcome mong muốn**:
- Cho phép nhân viên hợp pháp vào nhanh (< 1 giây from card swipe to door open)
- Chặn nhân viên bị revoke hoặc hết hạn
- Đảm bảo audit trail cho mục đích điều tra, tuân thủ

---

## 2. Use cases cần gọi API (tối thiểu 4)

### Use Case 1: Real-time Access Check (CRITICAL)
**Scenario**: Nhân viên quẹt thẻ tại main entrance lúc 9:00 AM  
**Flow**:
1. Gate card reader quẹt thẻ → read cardId
2. Gate gọi `POST /access/check` với { cardId, gateId, correlationId }
3. Core Business trả về { decision: "ALLOW", policyId, expiresAt, decisionId }
4. Gate mở cửa, log decisionId cho audit
5. **SLA**: Response < 100ms P99

**Data sent**:
```json
{
  "cardId": "EMP001234",
  "gateId": "LOBBY_01",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Data expected**:
```json
{
  "decision": "ALLOW",
  "policyId": "POL_EMP_STANDARD",
  "reasonCode": "VALID",
  "expiresAt": "2027-05-26T00:00:00Z",
  "decisionId": "DESC_550e8400_20260526_090001",
  "processingTimeMs": 45
}
```

---

### Use Case 2: Verify Rejected Access (DEBUG)
**Scenario**: Nhân viên bị từ chối vào → manager muốn biết tại sao  
**Flow**:
1. Gate trả về { decision: "DENY", reasonCode: "EXPIRED", decisionId }
2. Manager gọi `GET /decisions/{decisionId}` để xem chi tiết
3. Core Business trả về `requestContext` gốc, policy áp dụng
4. Manager thấy thẻ của nhân viên hết hạn 2026-01-01
5. **SLA**: Response < 1 second

**Data expected**:
```json
{
  "decisionId": "DESC_550e8400_20260526_090001",
  "cardId": "EMP001234",
  "gateId": "LOBBY_01",
  "decision": "DENY",
  "reasonCode": "EXPIRED",
  "policyId": "POL_EMP_STANDARD",
  "requestContext": {
    "cardholderName": "Nguyen Van A",
    "department": "Engineering",
    "expiresAt": "2026-01-01T00:00:00Z"
  }
}
```

---

### Use Case 3: Query Policy Details (OPERATIONAL)
**Scenario**: Manager cấu hình access cho nhóm → muốn xác minh policy đã apply  
**Flow**:
1. Manager gọi `GET /policies/access/{policyId}` (e.g., POL_EMP_STANDARD)
2. Core Business trả về full policy: timeWindows, quotas, cardholderGroups, accessGates
3. Manager confirm policy đúng (VD: chỉ access 9-18h, max 500 access/month)
4. **SLA**: Response < 2 seconds (not latency-critical)

**Data expected**:
```json
{
  "policyId": "POL_EMP_STANDARD",
  "validFrom": "2025-01-01T00:00:00Z",
  "validUntil": "2027-12-31T23:59:59Z",
  "cardholderGroups": ["EMPLOYEE", "CONTRACTOR"],
  "accessGates": [],
  "timeWindows": [
    {"dayOfWeek": "MON", "startTime": "09:00", "endTime": "18:00"},
    {"dayOfWeek": "TUE", "startTime": "09:00", "endTime": "18:00"}
  ],
  "quotas": {
    "dailyLimit": null,
    "monthlyLimit": 500
  }
}
```

---

### Use Case 4: System Health Check (MONITORING)
**Scenario**: Gate deployment monitoring → check Core Business health periodically  
**Flow**:
1. Gate periodic health check calls `GET /health` every 30 seconds
2. Core Business returns { status: "UP", components: {...} }
3. If status = "DOWN" → Gate switches to cached policy mode or fail-closed
4. If status = "DEGRADED" → Gate logs warning, continues but monitors closely
5. **SLA**: Response < 500ms, can tolerate < 5% failures

**Data expected**:
```json
{
  "status": "UP",
  "timestamp": "2026-05-26T14:30:00Z",
  "components": {
    "database": {"status": "UP", "latencyMs": 15},
    "cache": {"status": "UP", "hitRate": 0.95},
    "downstream": {"status": "UP"}
  }
}
```

---

## 3. Dữ liệu cần gửi lên và mong đợi nhận về

### POST /access/check (MUST HAVE)
**Consumer sends**:
- `cardId` (required): Unique ID from card, max 50 chars
- `gateId` (required): Gate identifier, max 50 chars
- `correlationId` (required): UUID for deduplication
- `timestamp` (optional): ISO 8601, server uses current if omitted
- `additionalContext` (optional): Device type, location, temperature

**Consumer expects**:
- `decision` (required): "ALLOW" or "DENY" — controls gate relay
- `policyId` (required nullable): Which policy was matched
- `reasonCode` (required): "VALID", "EXPIRED", "REVOKED", "TIME_RESTRICTED", etc.
- `expiresAt` (required nullable): When policy expires (for cache TTL)
- `decisionId` (required): UUID for audit trail
- `processingTimeMs` (required): Latency metric for monitoring

### GET /policies/access/{policyId} (OPTIONAL but important)
**Consumer expects**:
- `policyId`, `validFrom`, `validUntil`
- `cardholderGroups`: User belongs to which group
- `accessGates`: Allowed gates (empty = all)
- `timeWindows`: Allowed hours (for UI display)
- `quotas`: Daily/monthly limits
- `rules`: Ordered list of business rules

### GET /health (PERIODIC)
**Consumer expects**:
- `status`: "UP", "DOWN", or "DEGRADED"
- `timestamp`: When check was performed
- `components.database.status`, `components.cache.status`, etc.

### GET /decisions/{decisionId} (DEBUG)
**Consumer expects**:
- Complete audit record: cardId, gateId, decision, reasonCode
- `requestContext`: Original request details for investigation

---

## 4. Tình huống lỗi cần được xử lý

| Status | Consumer hiểu là | Gate sẽ xử lý thế nào |
|---:|---|---|
| **200 OK** | Decision thành công (ALLOW hoặc DENY) | Thực thi decision, log decisionId |
| **400 Bad Request** | CardId sai format hoặc correlationId không phải UUID | Log error, hiển thị "Card read error", yêu cầu user quẹt lại |
| **401 Unauthorized** | Gate API key hết hạn hoặc sai | Alert admin, try refresh key, if fail → fail-closed (deny all) |
| **403 Forbidden** | Gate (consumer) không được phép call API này | Alert security, lock gate, call admin |
| **404 Not Found** | CorrelationId/decisionId không tồn tại (old audit) | Not expected in normal flow; only for debug queries |
| **409 Conflict** | Duplicate request (same correlationId) | Return cached decision from 60s window |
| **422 Unprocessable Entity** | Policy inconsistency in database | Treat as DENY (fail-closed), log incident |
| **429 Too Many Requests** | Rate limit exceeded | Gate queued too many requests, wait & retry |
| **503 Service Unavailable** | Core Business down | Depend on fail-open/fail-closed SLA (to negotiate) |

---

## 5. Yêu cầu về hiệu năng

### Response Time SLAs
- **POST /access/check**: < 100ms P99 (user can't wait at door)
- **GET /health**: < 500ms acceptable
- **GET /policies/access/{policyId}**: < 2s acceptable
- **GET /decisions/{decisionId}**: < 1s acceptable

### Throughput
- Gate can handle ~10-20 card swipes/minute at peak
- At scale: 5 gates × 20 swipes/min = 100 req/min → acceptable within rate limit (5000 req/min)

### Reliability
- 99.9% uptime SLA = max 43 minutes/month downtime
- If Core Business down > 5 minutes → gate must have fallback strategy

### Caching & Offline Capability
- Gate can cache last policy decision for 30 seconds
- If network down → use cached policies to make best-effort decisions
- Log all offline decisions for later reconciliation

---

## 6. Yêu cầu về bảo mật từ phía Consumer

### Authentication to Provider
- **Must use**: Bearer Token in Authorization header
- **Token format**: JWT with exp < 24 hours
- **Required claims**: `sub` (gate ID), `scope` (access:read), `gateIds` (authorized gates)

### Data Confidentiality
- CardId is PII → encrypt TLS in transit, never store plaintext on gate
- CorrelationId is non-identifying → can log plaintext
- DecisionId can be logged (audit trail)

### Rate Limiting Awareness
- Gate must NOT spam API if Core Business slow
- Implement exponential backoff on 503 errors
- Don't exceed 5000 req/min per API key

### Retry & Idempotency
- Use same correlationId on retry (within 60 seconds)
- Core Business deduplicates; gate will receive cached response
- Don't create multiple audit records for same card swipe

### Monitoring & Alerting
- Gate monitors response time; alert if > 150ms P99
- Gate tracks error rates; alert if > 1% errors
- Gate validates decision determinism (same cardId+gateId → same policy)

---

## 7. Câu hỏi cần được đàm phán với Provider

1. **Fail-open or fail-closed SLA?** If Core Business down > 5 min, gate should DENY all or use cached policies?

2. **Timeout definition**: "< 100ms" — is this end-to-end (client sends to receives) or server processing only?

3. **Duplicate handling**: If gate sends same correlationId within 60s, will Provider return cached decision or fresh evaluation?

4. **Policy cache TTL**: Can gate cache policy for 5 minutes or must every decision be fresh?

5. **Audit retention**: How long does Core Business keep decision records (SLA for GET /decisions)?

6. **CardId format**: Must Gate validate cardId format before sending, or Provider will reject?

7. **Quota handling**: If employee hits monthly quota, does decision change immediately or next day reconcile?

8. **Multi-gate scenarios**: If one gate is offline, can consumer query from another gate's perspective?

9. **Real-time policy sync**: If policy updated, does gate need to explicitly call endpoint to refresh cache?

10. **Additional metadata**: Can gate send optional fields (device type, temperature) without breaking Provider?
- **Giả định 2:** Consumer cần khả năng phân trang cho danh sách log. Phương thức phân trang được ưu tiên là **cursor-based pagination** vì danh sách log có thể rất lớn và thay đổi liên tục. Cursor sẽ được trả về trong response.
- **Giả định 3:** Consumer kỳ vọng tất cả timestamp đều theo chuẩn ISO 8601 (ví dụ: `2026-05-20T10:30:00Z`) để dễ dàng xử lý và hiển thị trên toàn hệ thống.

---

## 5. Câu hỏi cho Provider

1.  **Về phân trang cho `/access/logs/recent`:** Provider hỗ trợ cursor-based pagination qua tham số `cursor` và `limit` phải không? Nếu có, `cursor` được trả về ở đâu trong response (ví dụ: trong header `Link` hoặc trong response body)?
2.  **Về chi tiết trạng thái `GateStatus`:** Field `isOnline` đại diện cho kết nối mạng của cổng, còn `isDoorOpen` đại diện cho trạng thái vật lý của cánh cổng? Có những trạng thái nào khác như `isMaintenance` hay `isLocked` không?
3.  **Về dữ liệu `CardInfo`:** Nếu một thẻ (`cardId`) bị vô hiệu hóa hoặc hết hạn, Provider có trả về `404 Not Found` hay vẫn trả về `200 OK` với `isActive: false` hoặc `status: expired` trong `CardInfo` object? Consumer mong muốn `200 OK` để phân biệt rõ "thẻ không tồn tại" và "thẻ tồn tại nhưng vô hiệu lực".

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider thay đổi tên field (VD: `gateId` thành `doorId`) | Consumer không thể parse và map dữ liệu, toàn bộ chức năng audit bị lỗi. | Chốt naming convention chặt chẽ trong `openapi.yaml` (ưu tiên `camelCase`). Sử dụng Spectral để kiểm tra breaking changes sau này. |
| Provider thiếu mã lỗi chi tiết | Consumer không thể phân loại lỗi để xử lý phù hợp, gây khó khăn cho việc debug và vận hành. | Chuẩn hóa response lỗi theo RFC 7807 (Problem Details) với các field `type`, `title`, `status`, `detail`, `instance`. |
| API `/access/check` (dành cho realtime) bị timeout hoặc chậm | Ảnh hưởng đến trải nghiệm người dùng tại cổng, gây ùn tắc. | Thống nhất timeout ở mức thấp (ví dụ: 500ms) và cơ chế fail-closed (từ chối truy cập) khi xảy ra lỗi. |
