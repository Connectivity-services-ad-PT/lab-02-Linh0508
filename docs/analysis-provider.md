# Phân tích yêu cầu — vai Provider

- **Cặp đàm phán**: Pair 10: Access Gate → Core Business
- **Loại**: REST sync
- **Provider service**: Core Business (cung cấp policy decision engine)
- **Consumer service**: Access Gate (gọi để kiểm tra quyền truy cập)
- **Người viết**: Nguyễn Thị Thuỳ Linh
- **Ngày**: 2026-05-26

---

## 1. Service mình cung cấp là gì

**Core Business** cung cấp một **Real-time Policy Decision Engine** cho hệ thống kiểm soát truy cập vật lý. Service này:

- Đánh giá policy truy cập dựa trên thẻ nhân viên (cardId), vị trí (gateId), thời gian, và các quy tắc kinh doanh lưu trữ trong hệ thống
- Trả về quyết định **ALLOW** hoặc **DENY** trong thời gian < 100ms
- Cung cấp audit trail đầy đủ cho mục đích kiểm tra, tuân thủ pháp lý
- Đảm bảo độ tin cậy 99.9% (ngoại trừ maintenance windows dự kiến)

Service này **cơ bản** cho hoạt động của toàn bộ hệ thống access control và không thể bị ngắt service một cách đột ngột.

---

## 2. Endpoints dự kiến phải cung cấp (tối thiểu 4)

### Endpoint 1: POST /access/check — Check real-time access policy
**Mục đích**: Xác định ngay liệu cardholder có quyền vào cổng này tại thời điểm này không  
**Khi Consumer gọi**: Ngay khi có quẹt thẻ tại cổng (latency-critical)  
**Input bắt buộc**:
- `cardId` (string): ID thẻ RFID
- `gateId` (string): ID cổng vật lý
- `correlationId` (string, UUID): Dùng để deduplication và audit

**Input tùy chọn**:
- `timestamp` (string, ISO 8601): Thời gian quẹt thẻ (nếu không gửi, lấy server time)
- `additionalContext` (object): Thông tin bổ sung như device type, location code

**Output** (HTTP 200):
- `decision` (string): "ALLOW" or "DENY"
- `policyId` (string, nullable): ID policy được áp dụng
- `reasonCode` (string enum): "VALID", "EXPIRED", "REVOKED", "TIME_RESTRICTED", "LOCATION_RESTRICTED"
- `expiresAt` (string, ISO 8601, nullable): Khi nào policy này hết hiệu lực
- `decisionId` (string, UUID): ID để audit sau này
- `processingTimeMs` (number): Thời gian xử lý

**Error responses**: 400, 401, 403, 409, 503 (với Problem Details)

---

### Endpoint 2: GET /policies/access/{policyId} — Retrieve policy details
**Mục đích**: Consumer muốn xem chi tiết policy nào đó  
**Khi Consumer gọi**: Khi debug hoặc kiểm chứng quyết định  
**Input**: `policyId` (path parameter)  
**Output**:
- `policyId`, `validFrom`, `validUntil`
- `cardholderGroups` (array): Nhóm có quyền
- `accessGates` (array): Cổng được phép (empty = tất cả)
- `timeWindows` (array): Khung giờ cho phép (ví dụ: chỉ 9-17h)
- `quotas` (object): Giới hạn truy cập hàng ngày/tháng
- `rules` (array): Chi tiết rule

**Error responses**: 404, 401

---

### Endpoint 3: GET /health — Health check
**Mục đích**: Monitoring, deployment checks  
**Output**:
- `status` (string): "UP", "DOWN", "DEGRADED"
- `timestamp` (string, ISO 8601)
- `components` (object):
  - `database`: Status kết nối database
  - `cache`: Status policy cache
  - `downstream`: Status service phía sau

**Error responses**: 503

---

### Endpoint 4: GET /decisions/{decisionId} — Audit access decision
**Mục đích**: Tra cứu quyết định trước đó để kiểm tra lỗi  
**Input**: `decisionId` (path parameter)  
**Output**:
- `decisionId`, `cardId`, `gateId`, `decision`
- `policyId`, `reasonCode`
- `requestedAt`, `decidedAt`, `processingTimeMs`
- `requestContext` (object, nullable): Context từ request gốc

**Error responses**: 404, 401

---

## 3. Dữ liệu đầu vào bắt buộc và tùy chọn

### POST /access/check — Bắt buộc
- `cardId`: string, format khác null, max 50 ký tự
- `gateId`: string, format khác null, max 50 ký tự
- `correlationId`: string UUID format, dùng để dedup

### POST /access/check — Tùy chọn
- `timestamp`: ISO 8601 string
- `additionalContext.deviceType`: string (card_reader, biometric, mobile)
- `additionalContext.locationCode`: string (building/floor)
- `additionalContext.temperature`: number (°C)

### GET /policies/access/{policyId} — Bắt buộc
- Header: `Authorization: Bearer <token>`

### GET /health — Không có bắt buộc
- Chỉ có query params tùy chọn (ví dụ: `?deep=true` để kiểm tra sâu)

---

## 4. Mã lỗi có thể trả về (4xx, 5xx)

| Status | Tình huống | Ví dụ |
|---:|---|---|
| **400 Bad Request** | Payload sai format, cardId không phải UUID, timestamp sai ISO 8601 | `{type: "https://...", status: 400, title: "Invalid card ID format"}` |
| **401 Unauthorized** | Thiếu Bearer token, token hết hạn, signature sai | `{type: "https://...", status: 401, title: "Missing authorization"}` |
| **403 Forbidden** | Token hợp lệ nhưng Consumer (gate) không được quyền access endpoint này hoặc gateId cụ thể | `{type: "https://...", status: 403, title: "Gate not authorized"}` |
| **404 Not Found** | PolicyId hoặc DecisionId không tồn tại hoặc đã expire (> 30 days) | `{type: "https://...", status: 404, title: "Policy not found"}` |
| **409 Conflict** | Duplicate request: Cùng correlationId gửi lại trong 60s, hoặc xung đột nghiệp vụ khác | `{type: "https://...", status: 409, title: "Duplicate request"}` |
| **422 Unprocessable Entity** | Policy rule không consistent trong database, payload hợp lệ nhưng vi phạm business rule | `{type: "https://...", status: 422, title: "Policy inconsistency"}` |
| **429 Too Many Requests** | Vượt rate limit (> 5000 req/min per key) | `{type: "https://...", status: 429, title: "Rate limit exceeded", headers: {"Retry-After": "60"}}` |
| **503 Service Unavailable** | Database down, cache rebuilding, downstream service down | `{type: "https://...", status: 503, title: "Service temporarily unavailable"}` |

Tất cả error responses dùng **Problem Details** format (RFC 7807).

---

## 5. Rủi ro và thách thức khi implement

### Technical Risk
1. **Latency tới mã lỗi 100ms SLA**: Nếu query policy từ database mất > 100ms, gate bị kẹt
   - **Mitigation**: In-memory cache policy (TTL 5 min), edge deployment
   
2. **Race condition**: Thẻ bị revoke giữa lúc check và lúc gate mở
   - **Mitigation**: Implement decision versioning; gate phải verify decisionId vẫn valid
   
3. **Duplicate request storm**: Card reader quẹt lại do network timeout, server nhận nhiều request giống hệt
   - **Mitigation**: Deduplicate bằng correlationId (60s window)
   
4. **Schema evolution**: Thêm trường policy mới → old gates không parse được
   - **Mitigation**: Tất cả field mới optional, API versioning strategy

### Operational Risk
1. **Policy cache inconsistency**: Cache stale (> 5 min) → decision sai
   - **Mitigation**: Clear cache on policy change, provide manual refresh endpoint
   
2. **Audit trail explosion**: 1M+ decisions/day → storage/query performance giảm
   - **Mitigation**: Archive decisions > 30 days to cold storage
   
3. **Token revocation latency**: Consumer token bị revoke nhưng consumer chưa biết
   - **Mitigation**: Short token TTL (max 24h), use blacklist for urgent revocation

### Business Risk
1. **Fail-open vs fail-closed**: Khi Core Business down, gate nên cho qua hay chặn?
   - **Provider perspective**: Recommend FAIL-CLOSED (deny) for security
   - **Need explicit agreement** từ customer

2. **Policy update latency**: Khi policy thay đổi, khi nào áp dụng tại gate?
   - **Provider perspective**: Assume near real-time (< 5s), older gates may need explicit refresh

3. **Audit compliance**: Cần lưu decision record bao lâu?
   - **Provider perspective**: 2-year retention; customer manages longer-term archive

---

## 6. Yêu cầu về bảo mật, xác thực

### Authentication
- **Scheme**: Bearer Token (OAuth 2.0 / JWT)
- **Header**: `Authorization: Bearer <token>`
- **Token claims**:
  - `iss`: Core Business identity service
  - `sub`: Consumer (gate) ID
  - `exp`: < 24 hours
  - `scope`: "access:read" for checks, "admin:policies" for policy view, "audit:read" for audit access
  - `gateIds`: List of gate IDs this consumer authorized for

### Authorization
- Each consumer có API key scoped to specific gate(s)
- Read policy: require "access:read" + "admin:policies"
- Query audit trail: require "audit:read" + có restrictions (only own decisions hoặc admin)

### Data Protection
- All endpoints HTTPS only (TLS 1.2+)
- CardId (PII): encrypted at rest, hashed in logs
- CorrelationId, DecisionId: can log plaintext (non-identifying)

### Rate Limiting & DDoS Protection
- Per-consumer: 5000 req/min
- Per-IP: 10000 req/min
- Burst allowance: 2x for 5 seconds
- Return `X-RateLimit-*` headers in responses

### Audit & Compliance
- All requests logged: timestamp, cardId (hashed), gateId, decision, consumer identity
- Decision trail: immutable (append-only), retention 2+ years
- Sensitive fields: logged with reduced precision/hashing

---

## 7. Giới hạn tốc độ (rate limiting)

### Per-Gate Tiers

**Tier 1: Low-Traffic Gate** (< 100 accesses/day)
- Rate: 100 req/min, Burst: 10 req/sec
- Example: Emergency exit

**Tier 2: Standard Gate** (100-500 accesses/day)  
- Rate: 500 req/min, Burst: 50 req/sec
- Example: Main lobby

**Tier 3: High-Traffic Gate** (> 500 accesses/day)
- Rate: 2000 req/min, Burst: 200 req/sec
- Example: Cafeteria entrance

### Admin/Policy Operations
- Rate: 10 req/min, Burst: 2 req/sec (strict limit)

### Escalation Policy
- Day 1-7 of overuse: Warning email
- Day 8-30: Throttling (responses delayed to 1s)
- Day 31+: Temporary suspension (1 hour auto-recovery)

---

## 8. Giả định bổ sung

- **Giả định 1**: Tất cả timestamp ở UTC, ISO 8601 format
- **Giả định 2**: CardId đã validate format từ gate (không accept rác)
- **Giả định 3**: Policy cache TTL = 5 minutes, manually refresh nếu cần real-time
- **Giả định 4**: Fail-open/fail-closed strategy phải được explicit agreement (sẽ đàm phán)
- **Giả định 5**: Audit log retention = 2 years minimum (customer can archive longer)
- **Giả định 6**: Response < 100ms target P99 (acceptable P99 < 150ms)
- **Giả định 7**: DB connection pool size >= 20 per instance
- **Giả định 8**: Gate assumed authenticated via API Gateway or internal network before reaching this service

---

## 9. Open Questions for Negotiation

1. Should /access/check return partial decision if policy cache stale (> 5 min)?
2. Is `policyId` required in response, or nullable if access granted by default?
3. Should GET /decisions/{decisionId} require admin scope?
4. How long retain decision audit records (SLA)?
5. Can cardholder quota (daily/monthly) reset at midnight or be configurable?
6. Max response size limit for /policies/access/{policyId} with 1000+ timeWindows?
7. Should duplicate requests (same correlationId) return cached response or fresh decision?
8. What's fail-open vs fail-closed SLA when Core Business is down?
- Giả định 3:

---

## 5. Câu hỏi cho Consumer

1. 
2. 
3. 

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên field không thống nhất | Consumer parse lỗi | Chốt naming trong `openapi.yaml` |
| Payload lớn | Timeout/mock lỗi | Thống nhất content-type và size limit |
