# Delivery Plan — VA Hub Polyglot Integration Platform (6 tháng đầu)

Nguồn yêu cầu: [BRIEF-v2.md](BRIEF-v2.md). Nguyên tắc điều hành: walking skeleton trước feature, vertical slice mỗi sprint, DoD không cắt, nợ kỹ thuật có deadline. Sprint 2 tuần, ký hiệu S1–S13.

> **Rev 2 (2026-07-04): hai quyết định.** (1) **AI Copilot tạm hoãn**; (2) **ưu tiên số 1 là tích hợp & ETL từ nhiều nguồn khác nhau**. Slot Phase 4 cũ (Copilot) trở thành **Connector framework & ETL đa nguồn**; write actions + Designer + Alerting UI dồn về Phase 5; các hạng mục LLM chuyển vào mục 8 "Chốt trước khi kích hoạt lại AI Copilot". Anomaly detection **giữ lại** — ML thống kê chạy nội bộ, không phụ thuộc LLM/API ngoài.
>
> **Rev 3 (2026-07-04): stakeholder đã trả lời đủ 12 câu hỏi mục 7.** Quyết định chính: nguồn #1 **và** sink #1 đều là **MySQL** (incremental polling; UPSERT theo business key + UpdatedAt) — MySQL là DB chính toàn hệ; MVP chạy trên **ServBay**, chuyển Docker/K8s sau khi MVP ổn định (chưa cần Helm/HPA/Vault, bắt buộc stateless); Keycloak nội bộ; alert **Email/Zalo** (không Slack); PII theo **Nghị định 13/2023**; Designer co thành **wizard/template** (20–50 pipeline năm đầu, chưa cần canvas); pilot tháng 7 = **shadow mode**. Chi tiết từng câu + tác động: mục 7.
>
> **Rev 4 (2026-07-06): Phase 0 đóng** — tag `v0.1.0`; benchmark/onboarding trong [README.md](../README.md); CI p95 qua `scripts/gitlab-ci-p95.ps1` hoặc GitLab Analytics. **Phase 1 mở** — ưu tiên walking skeleton, không tinh chỉnh thêm hạ tầng P0 trừ blocker P1.

## 0. Giả định quan trọng

1. **Team**: 6 kỹ sư full-time từ tuần 1 (2 Go, 2 Python, 2 Frontend); EM kiêm architect dành ~50% thời gian cho review/ADR/unblock, không nhận critical-path task. Không tuyển thêm giữa chừng.
2. **Deploy target** (chốt — mục 7 câu 4): MVP phát triển & kiểm chứng trên **ServBay**; hạ tầng không có trong ServBay (Kafka, OTel Collector, Jaeger, Prometheus, Loki, Grafana) chạy docker-compose — compose là môi trường chuẩn tham chiếu cho CI và staging. Chuyển Docker/K8s cho production chỉ sau khi MVP ổn định (ngoài 6 tháng); trong kỳ này **không** làm Helm/HPA/Vault. Đổi lại mọi service bắt buộc stateless + config qua env (12-factor) để migrate rẻ. Không multi-region. Mục tiêu tháng 6 là **pilot production nội bộ chạy shadow mode**, không phải GA.
3. **SLA & quy mô** (chốt — mục 7 câu 3, 8, 11): user là đội CNTT nội bộ ~5 người; topology ≤ 500 node; data plane MVP 50–200 msg/s peak, message 5–20 KB; 20–50 pipeline trong năm đầu. Load test bắt buộc hạ về 100 WS client + 500 msg/s (vẫn ≈ 20×/2,5× nhu cầu thật — headroom có chủ đích); mốc 1.000 WS client thành stretch đo headroom kiến trúc.
4. **AI Copilot: tạm hoãn** (quyết định 2026-07-04, chưa định ngày kích hoạt lại). Capacity giải phóng (~4 tuần) dồn toàn bộ cho ưu tiên số 1: tích hợp & ETL đa nguồn (Phase 4 mới). Các seam kiến trúc copilot cần (approval token, audit trail, RBAC, SSE-ready BFF) vẫn được xây ở P5 vì write actions từ UI cũng cần chúng — kích hoạt lại copilot read-only ước tính 4 tuần, không phải retrofit.
5. **Ưu tiên tính năng**: tích hợp & ETL đa nguồn (số connector + tốc độ thêm nguồn mới) > Visualizer real-time + observability > write actions + Designer + Alerting UI > Anomaly > Replay/chaos toggle/cost widget (stretch, được phép cắt). Danh mục nguồn đã chốt (mục 7 câu 7): **MySQL → REST API → SQL Server → Excel/SFTP → SOAP (nếu phát sinh)** — phủ 4 họ: database, API, file, legacy. Anomaly giữ lại vì không phụ thuộc LLM. Designer chốt ở mức **wizard/template, linear + fan-out** (mục 7 câu 11 — chưa làm canvas kéo-thả).

---

## 1. Bản đồ Phase (6 tháng)

| STT | Tên phase | Thời lượng | Mục tiêu 1 câu | Điều kiện Exit | Rủi ro chính |
| --- | --- | --- | --- | --- | --- |
| P0 | Contracts & Dev Platform | 2 tuần (S1) | Mọi dev chạy toàn stack local bằng 1 lệnh; contract codegen 3 ngôn ngữ hoạt động | **Đóng 2026-07-06** (`v0.1.0`); xem README audit | Toolchain đa ngôn ngữ ngốn thời gian hơn dự kiến |
| P1 | Walking Skeleton | 4 tuần (S2–S3) | 1 message xuyên adapter→Kafka→ETL→sink với 1 trace liền mạch nhìn thấy từ Web UI | **Đang mở** — demo live + DoD checklist P1 | OTel context propagation qua Kafka đứt; glue code bị đánh giá thấp |
| P2 | Real-time Visualizer & Observability sâu | 4 tuần (S4–S5) | Topology sống: status/throughput cập nhật 1s qua WebSocket; dashboard + log tail qua BFF; OIDC | Visualizer ≥ 30fps với 200 node mock; reconnect < 5s có test; drawer đủ metric/log/trace | Hiệu năng React Flow; 2 FE quá tải |
| P3 | Breadth & Resilience data plane | 4 tuần (S6–S7) | 2–3 nguồn thật, router + DLQ + retry + idempotency, batch Airflow, alert cơ bản | 500 msg/s sustained 30 phút không lag; message hỏng vào DLQ xem/requeue được từ UI | Hệ thống nguồn thật "bẩn" hơn giả định |
| P4 | Connector framework & ETL đa nguồn | 4 tuần (S8–S9) | Công nghiệp hóa việc thêm nguồn: connector SDK + ETL mapping khai báo; tổng ≥ 5 nguồn thuộc ≥ 3 họ | Nguồn mới nhất được thêm trong ≤ 3 người-ngày bằng SDK + template (đo thật); data quality check chạy trên mọi pipeline | Nguồn legacy "bẩn" ngốn effort; framework over-engineered |
| P5 | Designer, Write actions, Alerting UI & Anomaly | 5 tuần (S10–S12) | Tạo/sửa pipeline từ UI có revision/rollback; restart/scale 2-step confirm + RBAC; alert rule qua form; anomaly alert sau shadow | Deploy 1 pipeline mới từ Designer không sửa file tay; RBAC matrix test pass; anomaly precision shadow ≥ 60% | Designer là hố scope; 2 FE nghẽn |
| P6 | Hardening & Production Readiness | 3 tuần (S12–S13) | Chịu tải, an ninh, người ngoài team vận hành được | Load test đạt (100 WS client + 500 msg/s bắt buộc; 1.000 client stretch); OWASP ASVS L1 checklist đóng; 5 runbook được người ngoài verify; go/no-go shadow-mode pass | Load test phát hiện vấn đề kiến trúc muộn |

Ghi chú: P5 và P6 chồng nửa sprint (S12): 2 FE + 1 Python đóng P5, trong khi 2 Go + 1 Python bắt đầu hardening. **Không mở phase mới khi phase trước chưa đóng DoD** — nếu trượt, cắt stretch (Replay, chaos toggle, cost widget) trước, tuyệt đối không cắt mục DoD.

---

## 2. Chi tiết từng Phase

### Phase 0 — Contracts & Dev Platform (2 tuần, S1)

> **Trạng thái:** **Đóng 2026-07-06** — tag `v0.1.0`; milestone GitLab **Phase 0 — Contracts & Dev Platform** (đóng trên UI). Onboarding: [PHASE-0.md](PHASE-0.md); bằng chứng số: [README.md](../README.md) (benchmark, onboarding, CI). Nợ còn lại: [TECH-DEBT.md](TECH-DEBT.md).

**Vì sao ở đây:** với 3 ngôn ngữ và 4 tầng, chi phí lớn nhất nằm ở ranh giới. Contract codegen và CI đa ngôn ngữ là 2 thứ đắt nhất để retrofit — làm sau 1 tháng nghĩa là mỗi team đã tự chế convention riêng và ta trả giá bằng rework ở P1. Giới hạn cứng 2 tuần để không rơi vào "nền tảng hóa vô hạn": mọi thứ P1 chưa cần đều OUT. (Monorepo skeleton + pnpm/turbo/CI SAST đã có sẵn trong repo — P0 xây tiếp trên đó.)

**Vertical slice giao được:** dev clone repo, chạy `make dev-up` (hoặc `scripts/docker-dev-up.ps1` trên Windows), mở web thấy trang `/status` hiển thị health của 4 skeleton service (adapter, etl, bff, web) + Kafka/Jaeger/Prometheus/Loki; đổi 1 field trong proto → `make generate` → type Go/Python/TS đổi theo; push → CI xanh.

**Scope IN:**
- Skeleton 4 service: main + `/healthz` + `/readyz` + `/metrics` + structured log JSON
- docker-compose dev stack trong `infra/compose/` (Kafka, MySQL, OTel Collector, Jaeger, Prometheus, Loki, Grafana); dev trên ServBay có thể trỏ MySQL của ServBay qua env — compose vẫn là môi trường chuẩn cho CI/testcontainers (mục 7 câu 4)
- GitLab CI: build/lint/test + **Secret Detection** + SAST cho Go + Python + TS, có cache (gitleaks riêng không dùng — xem TD-P0-06)
- Contract toolchain: buf (proto) + JSON Schema + OpenAPI, `make generate` cho 3 ngôn ngữ
- ADR template + ADR-001..004; protected `main`, PR template có checklist DoD

**Scope OUT (hoãn có chủ đích):**
- K8s manifests → sau 6 tháng, khi MVP ổn định và chuyển production (mục 7 câu 4); ServBay + docker-compose đủ cho toàn kỳ MVP — đầu tư k8s sớm là chi phí chết khi topology service còn đổi
- Schema registry service → P3 (1 producer/1 consumer thì buf breaking check trong CI là đủ)
- GraphQL → không làm (ADR-004 chọn REST + WS; xem chi phí đổi ý bên dưới)
- Preview environment per MR → không làm trong 6 tháng (chi phí vận hành > lợi ích với team 6 người)

**Definition of Done:**
1. `make dev-up` từ clone sạch đến toàn bộ container healthy < 5 phút trên máy 16GB (đo trên 2 máy khác nhau, số liệu ghi vào README) — **Dev A: 100 s (2026-07-06); Dev B: chưa đo**
2. CI full pipeline p95 < 12 phút; merge vào `main` yêu cầu CI xanh + 1 approve — **cấu hình ✓; p95 ghi bằng `scripts/gitlab-ci-p95.ps1` hoặc GitLab Analytics**
3. `make generate` sinh code Go/TS/Python từ `shared/proto` + `packages/schemas`; CI có job diff-check fail nếu generated code lệch schema
4. 4 skeleton service có `/healthz`, `/readyz`, `/metrics`; 100% log line dạng JSON có field `trace_id`, `service`, `severity` (rỗng cũng phải có field)
5. Trang `/status` lấy health qua `GET /api/status` của BFF — browser không gọi trực tiếp service nào khác
6. GitLab **Secret Detection** + SAST xanh; `configs/env.example` đủ biến cho dev stack; 0 secret thật trong repo
7. 1 dev không tham gia setup chạy được toàn bộ theo README trong ≤ 30 phút — **drill ~15 phút (2026-07-06), ghi README**

**Nợ kỹ thuật cho phép:**

| Mục | Shortcut là gì | Deadline trả | Phase trả |
| --- | --- | --- | --- |
| SAST default ruleset | Chưa tune rule theo stack, có false negative | S6 | P3 |
| ServBay + docker-compose thay K8s | Không test được probe/resource limit trên orchestrator production | Ticket điều kiện: kích hoạt khi quyết định chuyển production (go/no-go P6) | Sau 6 tháng |
| Kafka 1 broker, không auth | Chỉ chấp nhận ở dev; staging cần SASL + 3 broker | S7 | P3 |

**ADR cần viết (kèm chi phí đổi ý sau 6 tháng):**
- ADR-001 Monorepo toolchain (pnpm + turbo; go module độc lập + go.work; Python per-app). Đổi turbo→nx ≈ 3 người-ngày, chỉ chạm config.
- ADR-002 Kafka làm message bus; **mọi producer/consumer qua interface `MessageBus`** (Go interface / Python Protocol), cấm import client Kafka ngoài package bus. Đổi sang Redpanda ≈ 0 (API-compatible, đổi endpoint); sang Pulsar ≈ 2–3 người-tuần (implement lại interface, business code không đổi).
- ADR-003 Contract pipeline: proto cho message, JSON Schema cho pipeline config, OpenAPI spec-first cho BFF → codegen client TS (`packages/sdk-ts`) + server stub. Đổi generator ≈ vài người-ngày vì spec là nguồn, không phải code.
- ADR-004 BFF protocol: REST + WebSocket, không GraphQL. Nếu sau này cần GraphQL: thêm gateway trước BFF ≈ 2 người-tuần, client không đổi vì đã bọc trong sdk-ts.

**Chỉ số theo dõi:** CI p95 (phút); số lần `main` đỏ/tuần (mục tiêu ≤ 1); thời gian onboarding dev thật; % service có healthz (phải 100%).

---

### Phase 1 — Walking Skeleton (4 tuần, S2–S3)

> **Trạng thái:** **Đang mở** — toàn bộ nguồn lực ưu tiên luồng adapter → Kafka → ETL → MySQL sink + trace UI. Không mở rộng scope P0.

**Vì sao ở đây:** đây là kiểm chứng rủi ro kiến trúc đắt nhất — trace propagation qua Kafka header, codegen 3 ngôn ngữ dùng thật, UI đọc topology từ BFF. Mọi quyết định sau phải dựa trên đường ống đã chạy, không dựa trên slide. Không sớm hơn được vì cần contract toolchain P0; không muộn hơn được vì mọi phase sau xếp chồng lên chính đường ống này.

**Vertical slice giao được:** dev ghi 1 bản ghi vào bảng nguồn MySQL → adapter Go poll incremental (checkpoint theo `updated_at`/PK — mục 7 câu 1) đóng message (proto `orders.v1`) vào topic → Python ETL validate + enrich → UPSERT vào MySQL đích theo business key + `updated_at` (mục 7 câu 2) → UI trang Topology hiển thị DAG 4 node với trạng thái, click node ETL mở drawer, click "View in Jaeger" ra đúng trace ≥ 4 span liền mạch.

**Scope IN:**
- 1 adapter MySQL incremental polling (Go), 1 ETL có logic thật (validate + enrich), 1 sink MySQL — UPSERT theo business key + `updated_at`
- OTel SDK cả 3 ngôn ngữ; W3C `traceparent` trong Kafka header
- Topology API từ config file tĩnh; UI topology page (đọc khi load + nút refresh); drawer v1 (status + link Jaeger)
- Integration test bằng testcontainers chạy trong CI

**Scope OUT (hoãn có chủ đích):**
- WebSocket live → P2 (realtime infra xứng đáng 1 slice riêng có reconnect/backoff tử tế, không làm vội trong skeleton)
- Router, DLQ → P3 (1 đường thẳng chưa có gì để route; DLQ không test được khi chưa có traffic lỗi thật)
- OIDC → P2 (P1 dùng basic auth qua reverse proxy, dev-only — có ticket nợ bên dưới)

**Definition of Done:**
1. E2E test trong CI: insert 1 bản ghi vào MySQL nguồn → assert row đúng ở MySQL đích **và** trace ở Jaeger có ≥ 4 span cùng `trace_id` (query Jaeger API trong test, không kiểm bằng mắt), chạy < 5 phút
2. buf breaking check bật; PR cố tình đổi schema không tương thích phải bị CI chặn (có test case minh chứng trong repo)
3. Coverage code mới: Go ≥ 60%, Python ≥ 70%, TS ≥ 60%; báo số trong MR, không giảm > 1%/PR
4. Mỗi service expose đúng 4 metric tên chuẩn hóa: `vahub_messages_in_total`, `vahub_messages_out_total`, `vahub_errors_total`, `vahub_process_duration_seconds` (histogram); dashboard Grafana "walking-skeleton" commit dạng JSON trong repo
5. ETL xử lý field PII (ví dụ email) nhưng log chỉ chứa id/hash — có test assert log output không chứa giá trị PII
6. Health check + readiness của cả 4 service được Prometheus scrape; alert "service down" thô hoạt động ở dev stack
7. README của 3 service mới cập nhật (run/test/env); ARCHITECTURE.md thêm sơ đồ walking skeleton + đường đi của trace
8. Demo cuối phase: người demo là dev không viết phần đó (kiểm tra tri thức không tập trung 1 người)

**Nợ kỹ thuật cho phép:**

| Mục | Shortcut là gì | Deadline trả | Phase trả |
| --- | --- | --- | --- |
| Topology từ config tĩnh | Chưa có registry/discovery, thêm node phải sửa file | S7 | P3 |
| Basic auth proxy | Không SSO, không role | S5 | P2 |
| Sink upsert đơn giản | Chưa có idempotency key chuẩn toàn tuyến | S7 | P3 |
| UI refresh tay | Không realtime | S4 | P2 |

**ADR cần viết:**
- ADR-005 Trace propagation qua Kafka header (W3C traceparent). Chuẩn mở — đổi backend trace (Jaeger→Tempo) ≈ 2 người-ngày (đổi exporter collector + deep-link UI).
- ADR-006 Topology model & naming (node id, edge id, pipeline id). Đây là ngôn ngữ chung của UI/BFF/data plane — đổi sau P3 chạm mọi tầng (≈ nhiều người-tuần), nên phải được review kỹ nhất trong architecture review đầu tiên.

**Chỉ số theo dõi:** e2e latency p95 của skeleton (làm baseline); integration test flake rate < 2%; lead time MR < 24h; số ngày `main` đỏ.

---

### Phase 2 — Real-time Visualizer & Observability sâu (4 tuần, S4–S5)

**Vì sao ở đây:** realtime topology là giá trị lõi với ops và là rủi ro FE lớn nhất (React Flow perf + WebSocket). Phải đo giới hạn React Flow sớm — nếu fail ở 200 node thì còn 3 tháng để đổi hướng render; sau Rev 3 rủi ro này chỉ còn ở Visualizer vì Designer đã co thành wizard/template (mục 7 câu 11). Không làm trước P1 vì cần topology + metric thật để stream.

**Vertical slice giao được:** ops mở Topology thấy status node + throughput edge cập nhật mỗi 1s không reload; kill BFF → UI hiện "reconnecting" và tự nối lại < 5s; drawer node có p50/p95/p99, error rate, sparkline 60s, log tail 100 dòng, link Jaeger; Dashboard 4 chart Prometheus qua BFF, auto-refresh 5s có nút pause; đăng nhập bằng SSO.

**Scope IN:**
- WS gateway trong BFF: snapshot-on-connect + delta mỗi 1s; schema event trong `packages/schemas` (zod, versioned)
- 7 custom node type theo brief; edge animation theo throughput, đổi màu amber khi backpressure
- Drawer đầy đủ; dashboard v1 (Recharts, layout cố định); log tail qua BFF→Loki
- OIDC Keycloak — user nội bộ tự quản, federate Google Workspace/Microsoft Entra ID để sau (mục 7 câu 5); đóng nợ P1. i18n scaffold + design tokens + dark mode (lint enforce)
- Mock topology generator 200 node/400 edge cho perf test

**Scope OUT (hoãn có chủ đích):**
- Dashboard drag/resize (react-grid-layout) → P6-stretch, và chỉ nếu ops thật yêu cầu (giá trị thấp hơn chi phí lúc này)
- Group/collapse theo domain → P3 (chỉ có ý nghĩa khi topology thật > 30 node)
- Alert UI → P3 (rule file trước, form sau)
- Replay/time-travel → P6 stretch

**Definition of Done:**
1. Visualizer mock 200 node/400 edge, 20% node đổi trạng thái mỗi giây, giữ ≥ 30fps — đo bằng Playwright + Chrome tracing chạy nightly, số liệu lưu lại từng lần chạy
2. WS delta payload p95 < 10KB/s/client với 200 node (delta, không gửi lại toàn bộ topology) — đo trong integration test
3. Reconnect: e2e test kill BFF → client tự nối lại < 5s với exponential backoff + jitter, nhận snapshot mới, không hiển thị data cũ như data sống
4. OIDC hoạt động: token TTL ≤ 15 phút + refresh; API không token trả 401 (test); basic auth P1 đã gỡ
5. 100% string UI qua next-intl key — lint rule chặn string literal tiếng Việt trong JSX, chạy trong CI; check tương tự cho hardcode hex color ngoài tokens
6. `prefers-reduced-motion` tắt edge animation + node pulse — E2E test có emulate
7. LCP trang topology < 2,5s trên máy chuẩn CI; Visualizer là dynamic import; main chunk < 300KB gzip (bundle analyzer chạy trong CI)
8. Log tail trả 100 dòng < 2s qua BFF; mọi panel metric ghi rõ timestamp nguồn dữ liệu

**Nợ kỹ thuật cho phép:**

| Mục | Shortcut là gì | Deadline trả | Phase trả |
| --- | --- | --- | --- |
| WS fan-out in-memory, 1 instance BFF | Chưa scale ngang (thiếu pub/sub backplane) | Quyết định tại arch review tháng 5; thi công S12 nếu load test cần | P6 (nợ điều kiện) |
| Sparkline giữ in-memory 60s ở BFF | Mất dữ liệu khi BFF restart | S9 — chuyển sang Prometheus range query | P4 |
| Dashboard layout cố định | Không cá nhân hóa | Review tháng 4: P6-stretch hoặc đóng won't-do | P6 |

**ADR cần viết:**
- ADR-007 WS event schema + versioning (field `v`, additive-only; breaking → event type mới). Đổi transport sang socket.io/SSE ≈ 1 người-tuần vì client đã bọc trong `packages/sdk-ts`.
- ADR-008 AuthN Keycloak OIDC. Đổi sang Auth0/IdP khác ≈ 3 người-ngày (config NextAuth + verifier BFF), vì không tự viết flow.
- ADR-009 Chart: Recharts, **mọi chart bọc trong `packages/ui/charts`** (call site không import Recharts trực tiếp). Đổi sang visx ≈ 1–2 người-tuần cho ~10 chart, call site không đổi.

**Chỉ số theo dõi:** fps benchmark nightly (trend); WS payload/client; FE error rate qua Sentry (< 1% session); số vi phạm lint i18n/token (trend về 0).

---

### Phase 3 — Breadth & Resilience data plane (4 tuần, S6–S7)

**Vì sao ở đây:** chỉ mở bề ngang sau khi skeleton + realtime đã đứng. Đây là lúc nối **hệ thống thật đầu tiên** — mọi giả định về data bẩn, throughput, schema drift được kiểm chứng bằng traffic thật. Resilience (DLQ/retry/idempotency) làm cùng lúc vì không có traffic lỗi thật thì không test được thiết kế resilience.

**Vertical slice giao được:** ops thấy 2–3 pipeline thật trên Visualizer; message hỏng chảy vào node DLQ đỏ; click DLQ xem message + lý do lỗi + trace; bấm requeue 1 message (thao tác ghi đầu tiên của hệ thống, có audit); alert "consumer lag vượt ngưỡng" bắn qua Email/Zalo (mục 7 câu 8).

**Scope IN:**
- Adapter #2 (REST API), #3 (SQL Server, DB-polling) theo thứ tự chốt ở mục 7 câu 7
- go-router: routing rule từ config; DLQ per pipeline + UI xem/requeue; retry policy chuẩn trong lib dùng chung Go/Python (exponential + max attempts + retry budget)
- Idempotency key toàn tuyến (sinh ở adapter, dedupe ở sink) — đóng nợ P1
- Schema registry (Karapace) + compatibility check CI; topology động từ registry — đóng nợ P1
- Airflow: 1 DAG batch thật, DAG mỏng chỉ orchestrate module python-etl
- 5 Prometheus alert rule dạng file trong repo + route Email/Zalo (mục 7 câu 8); group/collapse UI
- Audit log append-only cho hành động requeue

**Scope OUT (hoãn có chủ đích):**
- Alert management UI dạng form → P5 (đội nội bộ sửa rule qua MR là đủ đến lúc đó, còn được review miễn phí)
- Data lineage (OpenLineage/Marquez) → ngoài 6 tháng, cần ≥ 3 use case thật mới đầu tư
- Multi-tenancy → chưa có yêu cầu

**Definition of Done:**
1. 500 msg/s sustained 30 phút trên dev stack: consumer lag về 0 trong < 60s sau khi ngừng bơm — kịch bản load nằm trong repo, chạy nightly
2. Kill 1 consumer giữa tải → không mất message, không duplicate ở sink — integration test tự động cho cả at-least-once + dedupe
3. Message độc (schema sai/field thiếu) vào DLQ kèm lý do + `trace_id`; requeue từ UI chạy được — cả 2 đường có test
4. Schema registry chặn incompatible change: có PR mẫu cố tình phá bị CI + registry từ chối
5. Airflow DAG < 100 dòng, 0 business logic (chỉ gọi python-etl đã unit test); CI có job import + dry-run DAG
6. 5 alert rule được test bằng promtool trong CI; alert thật bắn qua Email/Zalo trong demo
7. Mọi write API (requeue) sinh đúng 1 audit row (user, thời điểm, đối tượng, kết quả) — contract test
8. Runbook #1 "Message vào DLQ thì làm gì" được 1 người ngoài team thực hiện thành công

**Nợ kỹ thuật cho phép:**

| Mục | Shortcut là gì | Deadline trả | Phase trả |
| --- | --- | --- | --- |
| Routing rule reload cần restart router | Chưa hot-reload | S11 (Designer cần deploy config động) | P5 |
| Alert chỉ route Email/Zalo, chưa phân cấp | Chưa escalation policy | S13 | P6 |
| Karapace single instance | Non-HA; mất registry là block deploy schema mới | S13 — HA hoặc backup + restore script được test | P6 |

**ADR cần viết:**
- ADR-010 DLQ & retry policy chuẩn trong lib chung Go/Python — copy logic retry giữa service bị cấm (xem anti-pattern #2).
- ADR-011 Schema registry Karapace (Confluent-compatible). Đổi sang Confluent/Redpanda SR ≈ config-only vì API tương thích.
- ADR-012 Idempotency contract. Đổi chiến lược sau khi có sink thứ 2 ≈ 2 người-tuần — chấp nhận, ghi rõ trong ADR điều kiện phải xem lại (thêm sink loại mới).

**Chỉ số theo dõi:** consumer lag p95; DLQ depth + tuổi message già nhất trong DLQ; duplicate rate ở sink (≈ 0); MTTR khi dev stack hỏng.

---

### Phase 4 — Connector framework & ETL đa nguồn (4 tuần, S8–S9)

*(Rev 2: phase này thay slot AI Copilot cũ — copilot hoãn, tích hợp đa nguồn là ưu tiên số 1.)*

**Vì sao ở đây:** đây là giá trị lõi của platform theo Rev 2. Không sớm hơn được vì framework phải được **chưng cất từ 3 adapter viết tay ở P1 + P3** (rule of three) — thiết kế connector SDK trước khi có 3 case thật gần như chắc chắn abstraction sai chỗ. Không muộn hơn vì mỗi nguồn thêm sau framework rẻ hơn hẳn; càng để muộn càng nhiều adapter viết tay phải migrate.

**Vertical slice giao được:** dev thêm 1 nguồn hoàn toàn mới (ví dụ SFTP Excel) bằng cách: chạy generator từ template → điền config + mapping khai báo → test → deploy; pipeline mới tự xuất hiện trên Visualizer với đủ trace/metric/DLQ; ops xem data quality dashboard (record pass/fail validation, freshness theo từng nguồn). Cuối phase: ≥ 5 nguồn sống thuộc ≥ 3 họ khác nhau.

**Scope IN:**
- Connector SDK (Go): interface chuẩn (poll/push, checkpoint/resume, rate limit, backoff) + template generator; refactor 3 adapter P1/P3 về SDK — phép thử framework tốt nhất
- ETL framework (Python): transform khai báo bằng mapping YAML (validate theo JSON Schema trong `packages/schemas`) cho case chuẩn + escape hatch code Python cho case phức tạp; unit test kiểu golden-file
- Thêm nguồn #4 (Excel/SFTP) và #5 (SOAP, nếu phát sinh) theo thứ tự chốt ở mục 7 câu 7
- Data quality checks: schema validation, null/duplicate rate, freshness per nguồn → metric + alert + dashboard
- Backfill: chạy lại ETL cho khoảng thời gian chọn qua Airflow (per nguồn), an toàn nhờ idempotency key P3

**Scope OUT (hoãn có chủ đích):**
- AI Copilot (mọi phần: chat, tool calling, RAG) → hoãn theo Rev 2; điều kiện kích hoạt lại ở mục 8
- Connector plugin động (load runtime .so/wasm) → ngoài 6 tháng; SDK compile-time đủ cho đội nội bộ
- CDC log-based (Debezium) cho mọi DB → chỉ áp dụng cho nguồn thật sự cần độ trễ giây; còn lại polling incremental rẻ hơn nhiều để vận hành — bảng quyết định trong ADR-015
- Data lineage chuẩn OpenLineage → ngoài 6 tháng (như P3 đã ghi)

**Definition of Done:**
1. Nguồn #5 (nguồn cuối của phase) được thêm trong ≤ 3 người-ngày từ kickoff đến chạy trên dev stack với đủ trace/metric/DLQ — đo thật, ghi breakdown thời gian vào report
2. 3 adapter cũ (P1/P3) đã migrate sang SDK — không tồn tại 2 cách viết adapter song song; integration test P3 không regression
3. ≥ 70% transform của các nguồn hiện có biểu diễn bằng mapping YAML (đếm theo số transform); phần code tay có unit test golden-file
4. Data quality: mỗi pipeline có pass/fail rate + freshness metric; alert khi fail rate vượt ngưỡng per nguồn; dashboard "data quality theo nguồn" commit trong repo
5. Backfill 1 ngày dữ liệu của 1 nguồn không tạo duplicate ở sink — idempotency test với dữ liệu backfill
6. Checkpoint/resume: kill adapter giữa chừng → resume đúng vị trí, không mất/không lặp — test cho cả họ file lẫn họ DB
7. Docs "How to add a source" ≤ 2 trang; 1 dev chưa từng viết adapter làm theo thành công (chính là phép đo của DoD 1)
8. Coverage giữ ngưỡng; mọi connector mới có contract test với schema registry

**Nợ kỹ thuật cho phép:**

| Mục | Shortcut là gì | Deadline trả | Phase trả |
| --- | --- | --- | --- |
| Mapping YAML chưa có UI editor | Sửa mapping = sửa file + MR | S11 — Designer property panel đọc/ghi mapping | P5 |
| CDC bằng polling incremental | Độ trễ phút, không phải giây | Nâng lên Debezium chỉ khi có yêu cầu latency thật — ticket điều kiện | Khi có yêu cầu |
| Ngưỡng data quality hardcode per nguồn | Chưa config từ UI | S12 — gộp vào Alert Management UI | P5 |

**ADR cần viết:**
- ADR-013 Connector SDK contract (checkpoint, backoff, rate limit). Chỉ chuẩn hóa những gì đã lặp ≥ 3 lần; mở rộng interface sau rẻ (additive), thu hẹp đắt.
- ADR-014 ETL mapping khai báo vs code (ranh giới ~70/30; escape hatch là first-class, không phải lỗ hổng). Đổi engine mapping ≈ 2 người-tuần vì mapping là data (YAML), không phải code.
- ADR-015 Chiến lược ingest per họ nguồn (polling vs CDC vs webhook — bảng quyết định kèm chi phí vận hành). *(Số ADR-013..015 cũ dành cho copilot chuyển thành ADR-022..024, dùng khi kích hoạt lại.)*

**Chỉ số theo dõi:** người-ngày để thêm 1 nguồn mới (trend phải giảm); % transform bằng mapping khai báo; data quality fail rate per nguồn; tổng số nguồn sống.

---

### Phase 5 — Designer, Write actions, Alerting UI & Anomaly (5 tuần, S10–S12)

**Vì sao ở đây:** mọi tính năng ghi (deploy pipeline, restart/scale) cần nền RBAC + audit + config động — chín ở P3–P4. Designer làm muộn **có chủ đích**: đến giờ team đã có 5+ pipeline thật và mapping khai báo từ P4, nên Designer sinh ra đúng thứ hệ thống thật dùng (config + mapping), không phải trên giả định — sai lầm phổ biến nhất của low-code tool. Anomaly cần ≥ 2 tháng metric lịch sử để train/validate — giờ mới có. Alerting UI vào đây vì tái dùng revision model của config service và cần data quality rule từ P4.

**Vertical slice giao được:** operator dùng wizard chọn source→transform (template)→sink (property panel đọc/ghi cả mapping YAML của P4), Save tạo revision, Deploy → pipeline chạy trên dev stack và hiện trên Visualizer; viewer bấm Deploy bị 403; operator bấm Restart trên node lỗi → confirm 2 bước → hành động chạy + audit; admin tạo alert rule qua form, dry-run trên 7 ngày dữ liệu rồi mới apply; anomaly detector bắt spike → node halo đỏ + alert (sau 2 tuần shadow).

*(Rev 3 — mục 7 câu 9: tổ chức xác nhận MVP chưa cần approval workflow đa cấp, chỉ cần RBAC Admin/Operator/Viewer; thiết kế hiện tại — 3 role + tự-confirm 2 bước (không phải duyệt qua người khác) + audit — đã đủ, không cần thêm bước phê duyệt liên người. Mục 7 câu 11: 20–50 pipeline/năm đầu xác nhận Designer chỉ cần **wizard/template**, không cần canvas kéo-thả tự do.)*

**Scope IN:**
- RBAC 3 role (viewer/operator/admin) enforce ở BFF middleware (không phải chỉ ẩn nút); role map từ Keycloak groups
- Write actions từ UI: restart/scale service, requeue DLQ — approval token 2 bước + audit; thiết kế 2-client ngay từ đầu (UI bây giờ, copilot tái dùng khi kích hoạt lại)
- Designer dạng wizard/template (chọn source có sẵn → transform từ template mapping → sink; property panel generate từ JSON Schema qua react-hook-form + zod resolver, đọc/ghi mapping P4 — đóng nợ P4). Không phải canvas kéo-thả tự do (mục 7 câu 11)
- Config service: revision immutable, diff, rollback; deploy = ghi config + trigger hot-reload router/Airflow (đóng nợ P3)
- Alert Management UI: form tạo/sửa rule (PromQL threshold + data quality rule), dry-run bắt buộc trên dữ liệu lịch sử; rule sync về repo dạng file (as-code, không clickops)
- Anomaly detector v1: Isolation Forest trên metric 5 phút, chạy shadow 2 tuần rồi mới bật alert
- Audit UI (xem mọi hành động ghi); Access Control UI read-only (sửa role vẫn qua Keycloak)

**Scope OUT (hoãn có chủ đích):**
- Copilot write tools → hoãn cùng copilot (cơ chế approval token đã sẵn sàng cho nó khi quay lại)
- Designer canvas kéo-thả tự do + DAG phức tạp (conditional branch/merge) → ngoài 6 tháng; chỉ wizard linear + fan-out vì ~80% pipeline thật là linear và quy mô 20–50 pipeline/năm chưa cần canvas (mục 7 câu 11); case phức tạp viết config tay — mở rộng khi có đủ 3 case thật
- Prophet/LSTM AE → ngoài 6 tháng (cần labeled data; Isolation Forest trước)
- Alert escalation UI → P6 dạng config file

**Definition of Done:**
1. E2E Playwright: tạo pipeline từ Designer → deploy → bơm message → thấy trên Visualizer — không sửa file tay ở bước nào
2. RBAC matrix test tự động cho 3 role × mọi write endpoint; gọi thẳng API bypass UI cũng bị chặn (test)
3. Revision: save tạo revision immutable; diff 2 revision hiển thị được; rollback hoạt động — test cả 3
4. Approval token cho write action: sinh ở bước confirm, TTL 5 phút, single-use; endpoint từ chối khi thiếu/hết hạn — test; audit ghi đủ (ai đề xuất, ai confirm, kết quả)
5. Alert rule từ form: validate PromQL + dry-run bắt buộc (hiện số lần rule sẽ bắn trên 7 ngày dữ liệu) trước khi apply; rule có mặt trong Prometheus < 30s; mọi thay đổi có revision + audit; CI verify file trong repo không lệch với rule đang chạy
6. Anomaly: inject spike nhân tạo → alert + halo trên UI < 60s; precision qua 2 tuần shadow ≥ 60% (đối chiếu tay với ops) trước khi bật alert thật
7. Pipeline config schema có version; config v1 cũ vẫn deploy được khi schema lên v1.1 — backward-compat test; hot-reload router/ETL < 10s không mất message (đóng nợ P3)
8. Coverage giữ ngưỡng các phase trước; 0 vi phạm lint i18n/token trong code mới

**Nợ kỹ thuật cho phép:**

| Mục | Shortcut là gì | Deadline trả | Phase trả |
| --- | --- | --- | --- |
| Designer chỉ linear + fan-out | Pipeline phức tạp phải viết tay | Review nhu cầu cuối tháng 6, quyết mở rộng hay giữ | Sau 6 tháng |
| Isolation Forest train offline weekly | Không online learning; drift chậm phát hiện | Đánh giá lại khi FP rate > 40% | Sau 6 tháng |
| Role per-platform (chưa per-pipeline) | Granularity thô | Chỉ làm per-pipeline nếu có yêu cầu compliance — quyết ở S13 | P6/quyết định |

**ADR cần viết:**
- ADR-016 Pipeline config format + revision model — đây là public contract của Designer; đổi format sau khi có N pipeline = data migration, nên phải versioned từ ngày đầu.
- ADR-017 Approval token pattern cho write action (1 cơ chế, 2 client: UI bây giờ, copilot khi kích hoạt lại).
- ADR-018 Anomaly pipeline (metric → feature → model artifact + manifest → alert). Đổi model ≈ người-ngày vì artifact có registry riêng.

**Chỉ số theo dõi:** số pipeline tạo qua Designer vs viết tay; tỷ lệ write action bị hủy ở bước confirm (đo friction); số alert rule tạo từ form vs sửa file; anomaly precision/recall trong shadow; FE velocity (rủi ro 2 FE nghẽn).

---

### Phase 6 — Hardening & Production Readiness (3 tuần, S12–S13)

**Vì sao ở đây:** KHÔNG phải vì "chất lượng để cuối" — chất lượng đã nằm trong DoD từng phase. P6 gom các việc **chỉ làm được khi hệ đủ hình hài**: load test tổng hợp, security review tổng, DR drill, bàn giao vận hành. Không muộn hơn được vì tháng 7 là pilot.

**Vertical slice giao được:** một SRE ngoài team, chỉ dùng runbook + UI, xử lý thành công 3 sự cố giả lập (consumer lag, DLQ đầy, nguồn ngoài sập) trên staging, trong khi hệ đang chịu 100 WS client + 500 msg/s (mục tiêu bắt buộc — mục 7 câu 3/8; 1.000 client là stretch đo headroom).

**Scope IN:**
- Load test: k6 WS 100 client (bắt buộc, mục 7 câu 3/8) + data plane 500 msg/s sustained; 1.000 WS client là chạy stretch đo headroom nếu còn thời gian; fix theo kết quả (kích hoạt nợ điều kiện WS backplane Redis nếu cần)
- Security: CSP không inline script (test header mọi route); dependency + container scan làm gate; secret quản lý qua `.env` + gitleaks (Vault hoãn tới khi chuyển K8s production — mục 7 câu 4); checklist OWASP ASVS L1 nội bộ
- Backup/restore drill (MySQL, Kafka config, schema registry) trên stack ServBay/docker-compose — K8s manifests dời sang giai đoạn chuyển production sau 6 tháng (đóng nợ P0 theo hình thức mới)
- 5 runbook + on-call guide; SLO + error budget dashboard; chaos toggle staging-only (RBAC-gated)
- E2E suite các luồng chính: xem pipeline → nhận alert → focus node lỗi → xem trace; thêm nguồn mới bằng connector SDK; tạo pipeline từ Designer → deploy (luồng copilot của brief mục 9 bổ sung khi kích hoạt lại)

**Scope OUT (stretch — cắt không ân hận nếu thiếu thời gian):**
- Replay/time-travel v1 (playback metric-based 1x–4x) — không chặn pilot
- Cost widget — cần dữ liệu pricing cloud, giá trị thấp hơn mọi mục hardening
- Dashboard drag/resize, light mode polish

**Definition of Done:**
1. 100 WS client đồng thời (bắt buộc): p95 delta latency < 2s, BFF không OOM trong 1 giờ, đồng thời data plane 500 msg/s sustained — report số liệu commit vào repo; 1.000 client là kết quả stretch, không chặn go/no-go nếu 100 client đạt
2. CSP đúng trên mọi route (automated header test); 0 secret trong repo/image (gitleaks + trivy); ASVS L1: 100% mục applicable đóng, mục N/A có lý do ghi lại
3. DR drill: khôi phục staging từ backup < 30 phút, script hóa, thực hiện bởi người không viết script
4. 3 kịch bản chaos (kill router, nghẽn Kafka, nguồn ngoài timeout hàng loạt) chạy trên staging, hệ phục hồi đúng như runbook mô tả
5. E2E suite xanh 5 lần chạy liên tiếp (flake check); các luồng chính kể trên nằm trong suite
6. Tech-debt cuối kỳ: ≤ 10 ticket mở, 0 ticket quá hạn, 0 ticket thiếu deadline; danh sách mock ([docs/MOCKS.md](MOCKS.md)) rỗng hoặc từng mục có quyết định "chấp nhận vĩnh viễn" ký tại architecture review
7. 5 runbook được người ngoài team thực thi thành công; on-call rotation + kênh alert chốt
8. Go/no-go review: **pilot = chạy shadow mode song song hệ hiện tại**, chưa thay thế nghiệp vụ thật (mục 7 câu 12); SLO đề xuất (99,5% uptime UI; data lag < ngưỡng chốt với stakeholder) được chấp nhận bằng văn bản

**Nợ kỹ thuật cho phép:** về nguyên tắc P6 không tạo nợ mới. Ngoại lệ duy nhất: finding từ load test **ngoài envelope pilot** (ví dụ > 200 client — 2× mốc bắt buộc) được phép defer thành ticket có deadline quý kế tiếp, ghi rõ ngưỡng kích hoạt.

**ADR cần viết:** ADR-019 SLO & error budget; ADR-020 WS scale-out (chỉ khi nợ điều kiện P2 kích hoạt).

**Chỉ số theo dõi:** kết quả load test vs target; số security finding mở theo severity (0 high khi go-live); flake rate E2E; số runbook được verify bởi người ngoài team.

---

## 3. Ma trận "Production-grade ngay" vs "Tối giản có kiểm soát"

Quy ước mức: **Không** = chưa tồn tại (có chủ đích) | **Toy** = tối giản có kiểm soát, có ticket nâng cấp kèm deadline | **Prod** = production-grade, không được hạ cấp.

| # | Hạng mục | Cuối Phase 1 | Cuối Phase 3 | Cuối Phase 6 |
| --- | --- | --- | --- | --- |
| 1 | AuthN | Toy: basic auth proxy dev-only (ticket S5) | Prod: OIDC Keycloak, token ≤ 15' (từ P2) | Prod: + refresh, revoke session |
| 2 | RBAC | Không (toàn hệ read-only) | Toy: write duy nhất (requeue) gate bằng admin group | Prod: 3 role enforce tại BFF + matrix test (từ P5) |
| 3 | Secret mgmt | Prod-tối-thiểu: .env + gitleaks gate CI, 0 secret trong repo | Như P1 + sealed values cho staging (ServBay) | Toy có kiểm soát: `.env` + gitleaks trên ServBay (mục 7 câu 4) — Vault/sealed-secrets hoãn tới khi chuyển K8s production, ticket điều kiện |
| 4 | mTLS nội bộ | Không (network dev riêng) | Không — ADR ghi điều kiện bật | Không — chỉ có ý nghĩa trên K8s; TLS tại edge là đủ trên ServBay, mTLS/network policy quyết khi chuyển K8s production (ADR) |
| 5 | Contract & codegen | Prod ngay từ P0: schema là nguồn, cấm viết tay type | Prod | Prod |
| 6 | Schema registry | Toy: buf breaking check trong CI (đủ khi 1 producer) | Prod: Karapace + compat check runtime | Prod: + HA/backup được test |
| 7 | DLQ | Không (lỗi → log + metric, ticket S7) | Prod: DLQ per pipeline + UI xem/requeue | Prod: + alert depth/age, replay từ DLQ |
| 8 | Retry / circuit breaker | Toy: retry đơn giản trong adapter | Prod: retry policy chuẩn trong lib chung | Prod: + circuit breaker mọi outbound (BFF → backend, adapter → nguồn ngoài) |
| 9 | Idempotency | Toy: UPSERT MySQL đơn giản theo business key (ticket S7) | Prod: key toàn tuyến, dedupe test tự động | Prod: + kiểm bằng chaos test |
| 10 | Tracing e2e | Prod ngay từ P1: propagation là DoD, 100% sample dev | Prod: + tail-sampling staging | Prod: sampling policy theo tải, retention 7–14 ngày |
| 11 | Structured log + trace_id | Prod ngay từ P0 (field bắt buộc, lint) | Prod | Prod |
| 12 | i18n | Prod-quy-ước từ P0: 100% string qua key (lint chặn) | Đủ tiếng Việt | + English cho màn hình chính |
| 13 | Design tokens / dark mode | Prod-quy-ước từ P0: tokens tập trung, lint chặn hex | Prod | Prod: + light mode, reduced-motion audit |
| 14 | LLM guardrails | — (copilot hoãn) | — (copilot hoãn) | Hoãn cùng copilot; approval token + audit pattern đã sẵn từ P5 cho UI write actions |
| 15 | LLM cost control | — (copilot hoãn) | — (copilot hoãn) | Hoãn cùng copilot |
| 16 | Audit log | Không (chưa có hành động ghi) | Toy: bảng append-only cho requeue | Prod: mọi write action, xem từ UI, retention chốt |
| 17 | Designer low-code | Không | Toy: topology read-only từ registry | Prod-phạm-vi-hẹp: linear + fan-out, revision/rollback |
| 18 | Replay / time-travel | Không | Không | Stretch: v1 metric-playback hoặc chuyển roadmap quý sau |
| 19 | Chaos toggle | Không | Toy: script inject tay phục vụ test P3 | Prod: UI toggle staging-only, RBAC-gated |
| 20 | Cost/resource widget | Không | Không | Toy: CPU/mem per service; cost estimate là stretch |
| 21 | CI security | SAST + gitleaks từ P0 | + dependency & container scan | + DAST staging, license check, gate theo severity |
| 22 | Backup/DR | Không | Toy: MySQL daily ở staging | Prod: backup đủ thành phần + restore drill script hóa |
| 23 | WS resilience | — (chưa có WS) | Prod: reconnect + snapshot/delta từ P2 | Prod: load-tested 100 client bắt buộc (1.000 client stretch), backplane nếu kích hoạt |
| 24 | Data quality checks | Không | Toy: validation trong ETL, lỗi → DLQ | Prod: pass/fail + freshness metric per nguồn, alert + dashboard (từ P4) |
| 25 | Connector SDK | Không (adapter viết tay có chủ đích) | Không — chưa đủ 3 case để chưng cất | Prod: SDK + template, thêm nguồn ≤ 3 người-ngày (từ P4) |

Nguyên tắc đọc ma trận: cột "Toy" luôn đi kèm ticket `tech-debt` có deadline (mục 4). Một hạng mục Prod không bao giờ quay lại Toy — nếu buộc phải hạ (ví dụ tắt check để chữa cháy), cần approve của EM và ticket khôi phục deadline ≤ 1 sprint.

---

## 4. Cơ chế chống nợ kỹ thuật (vận hành liên tục)

### 4.1 Quy trình review PR

Checklist trong PR template, reviewer tick từng mục — không tick đủ không merge:

1. Diff ≤ 400 dòng (không tính generated/lockfile); lớn hơn phải tách hoặc ghi lý do 1 dòng
2. CI xanh: build, lint (golangci-lint / ruff / eslint), test, coverage không giảm > 1%, gitleaks, SAST
3. 1 approve; **2 approve nếu chạm contract** (đường dẫn `shared/proto`, `packages/schemas`, OpenAPI spec) — enforce bằng CODEOWNERS
4. Code path mới có metric + log có trace_id chưa? (nếu không, ghi lý do)
5. Hành vi thay đổi → README / docs/ARCHITECTURE.md cập nhật trong cùng PR (không "sẽ bổ sung sau")
6. Có shortcut nào không? → link tới ticket `tech-debt` đã tạo, ngay trong description PR
7. Generated code không sửa tay (CI diff-check tự bắt)
8. String UI qua i18n key, màu qua token (lint tự bắt — reviewer chỉ xử ngoại lệ)

### 4.2 Tech-debt ticket

Một ticket `tech-debt` hợp lệ bắt buộc có đủ 5 field, thiếu 1 là triage trả lại:

- **Shortcut**: cái gì đang tối giản (1–2 câu, chỉ vào code/file cụ thể)
- **Lý do**: vì sao chấp nhận lúc này
- **Tác động nếu không trả**: cụ thể, không chung chung — ví dụ "không có schema registry thì breaking change chỉ phát hiện lúc runtime ở consumer, MTTR ước tính +2h/sự cố"
- **Interest**: chi phí lặp lại mỗi sprint khi chưa trả (ước lượng giờ)
- **Deadline**: sprint cụ thể (S-số), map với phase trả trong plan này

Ticket nằm **chung board** với feature (không backlog riêng — backlog riêng là nghĩa địa), có label `tech-debt`, quét được bằng filter. Mock/stub đăng ký thêm vào [docs/MOCKS.md](MOCKS.md) để grep được một chỗ.

### 4.3 Debt budget

- Mỗi sprint dành cố định **15–20% capacity** cho trả nợ: với team 6 người ≈ 1 người full-sprint, luân phiên ("debt duty"), không phải "ai rảnh thì làm".
- **Ngưỡng dừng feature** (chạm 1 trong 3 là sprint kế tiếp thành consolidation sprint — chỉ debt + bug, không feature mới):
  1. \> 12 ticket `tech-debt` đang mở, hoặc
  2. Bất kỳ ticket nào quá deadline > 1 sprint, hoặc
  3. ≥ 2 metric sức khỏe codebase (mục 4.5) đỏ 2 sprint liên tiếp
- Quyết định dừng feature do EM công bố tại sprint planning, không cần đồng thuận — đây là cơ chế, không phải đàm phán.

### 4.4 Cadence

- **Weekly debt triage** (30 phút, thứ Tư): EM + 1 đại diện mỗi stack. Việc: ticket mới đủ 5 field chưa; ticket sắp đến hạn ai nhận; ticket nào xin gia hạn (gia hạn tối đa 1 lần, lần 2 phải lên architecture review).
- **Monthly architecture review** (90 phút): duyệt ADR mới; rà metric sức khỏe; quyết định nâng cấp/chấm dứt các "toy version" đến hạn; rà nợ điều kiện (ví dụ WS backplane). Biên bản commit vào `docs/arch-review/`.
- **Sprint retro** có mục cố định: "Sprint này phát sinh shortcut nào — có ticket chưa?" (bắt shortcut chui).

### 4.5 Metric sức khỏe codebase (ngưỡng đỏ)

| Metric | Ngưỡng xanh | Đỏ khi |
| --- | --- | --- |
| Coverage (Go / Python / TS) | ≥ 60% / 70% / 60% | Dưới ngưỡng hoặc giảm > 3 điểm/tháng |
| Lint error | 0, warning không tăng | Warning tăng 2 sprint liên tiếp |
| Duplication (jscpd + dupl) | < 5% mỗi package | ≥ 5% |
| p95 CI time | < 12 phút | > 15 phút |
| MTTR khi `main` đỏ | < 4 giờ làm việc | > 1 ngày |
| Flaky test quarantine | ≤ 3, mỗi cái có ticket | > 3 hoặc có cái không ticket |
| ADR mới / tháng | ≥ 1 | 0 hai tháng liên tiếp (kiến trúc đang trôi không ghi nhận) |
| Tuổi trung bình ticket tech-debt | < 30 ngày | > 45 ngày, hoặc ticket già nhất > 60 ngày |

Dashboard các metric này dựng ngay ở P0–P1 (số liệu lấy từ CI + GitLab API), hiển thị trong Grafana nội bộ — chính hệ thống mình đang xây.

### 4.6 RACI

| Việc | EM/Architect | Stack lead (Go/Py/FE, kiêm nhiệm) | Dev |
| --- | --- | --- | --- |
| Debt budget, tuyên bố feature freeze | A + R | C | I |
| ADR | A (approve) | R (viết) | C |
| Tạo ticket khi tạo shortcut | I | C | **R** (người tạo shortcut tạo ticket) |
| Weekly debt triage | R | R | I |
| Monthly architecture review | A + R | R | C |
| Gate DoD cuối phase | **A** (ký exit) | R (chứng minh bằng số liệu) | C |

---

## 5. Anti-pattern cần tránh

| # | Anti-pattern | Dấu hiệu sớm | Cách xử lý |
| --- | --- | --- | --- |
| 1 | **God BFF** — business logic dồn vào BFF | Handler > 200 dòng; test BFF phải mock ≥ 5 service; BFF import domain package | BFF chỉ aggregate/translate/authz; logic đẩy về service sở hữu data; review size handler trong PR checklist |
| 2 | **Copy-paste giữa Go services** | dupl/jscpd báo file trùng > 70% với service khác; "sửa bug 1 chỗ, quên 2 chỗ" | Rule of three: lặp lần 3 mới extract vào lib chung có owner (retry, bus, otel đã là lib chung từ P3); cấm copy retry/DLQ logic |
| 3 | **UI gọi thẳng Prometheus/Jaeger/Loki** | URL Prom xuất hiện trong code FE; đề xuất mở CORS cho Prom | Chặn bằng network policy + lint rule cấm hostname infra trong `apps/web`; mọi query qua BFF — đã là quy ước từ P0 |
| 4 | **Write action không audit** (gồm cả LLM tool call khi copilot kích hoạt lại) | Endpoint write mới merge không kèm audit test; số hành động ghi tăng nhưng bảng audit không tăng tương ứng | Audit middleware đặt ở BFF/executor dùng chung (không thể quên từng endpoint); contract test "1 write = 1 audit row" chạy CI |
| 5 | **Airflow DAG chứa business logic** | DAG file > 100 dòng; import pandas/transform trong DAG; sửa logic phải deploy lại Airflow | DAG chỉ orchestrate; logic trong `pipelines/python-etl` có unit test; CI check độ dài + import whitelist cho thư mục dags |
| 6 | **Schema breaking không versioning** | PR sửa field proto không tăng version; consumer service khác đỏ sau merge | buf breaking + registry compat check là gate CI (P1/P3); quy tắc additive-only; breaking = topic/event type mới |
| 7 | **Mock vĩnh viễn** | [docs/MOCKS.md](MOCKS.md) chỉ tăng không giảm; ticket nâng cấp không có sprint | Mọi mock phải có mặt trong MOCKS.md + ticket 5-field; weekly triage rà; DoD P6 yêu cầu MOCKS.md rỗng hoặc từng mục được ký "chấp nhận" |
| 8 | **Horizontal sprint lén** | Sprint board toàn ticket 1 stack; demo sprint không click được từ UI | Sprint planning yêu cầu mỗi story ghi "demo path" bắt đầu từ UI; story thuần backend phải nêu slice nào nó phục vụ |
| 9 | **Chatty WebSocket** — gửi cả topology mỗi giây | Payload WS > 50KB/s/client; CPU BFF tăng tuyến tính theo số client | Snapshot-on-connect + delta là thiết kế bắt buộc từ P2; payload/client là metric có ngưỡng trong test |
| 10 | **Server state trong zustand** | Store chứa data fetch từ API + logic cache tay; bug "data cũ không refresh" | Quy ước từ P2: TanStack Query cho server state, zustand chỉ UI state; ESLint rule cấm gọi fetch trong store |
| 11 | **Notebook-to-prod model** | Model anomaly train trong notebook, file .pkl copy tay lên server | Training script versioned trong repo; artifact + manifest (ADR-018); deploy model qua pipeline như code |
| 12 | **Grafana clickops** | Dashboard/alert sửa tay trên UI, không có trong git; "dashboard của ai đây?" | Dashboard + alert rule as code trong repo từ P1/P3, CI sync; thay đổi trên UI bị ghi đè — nói rõ với team từ đầu |

---

## 6. Rủi ro & Kế hoạch dự phòng

| Rủi ro | Xác suất | Tác động | Trigger phát hiện | Kế hoạch B |
| --- | --- | --- | --- | --- |
| Kafka ops quá nặng cho team 6 người | Trung bình | Cao | Mất > 1 người-ngày/sprint cho vận hành Kafka dev/staging | Đổi sang Redpanda (Kafka-compatible, 1 binary) — chi phí ≈ 0 nhờ interface MessageBus (ADR-002) |
| React Flow không kham nổi topology lớn | Thấp–TB | Trung bình | Benchmark P2 < 30fps với 200 node sau khi đã tối ưu | Virtualization + group/collapse mặc định; nếu vẫn fail: tách edge-animation layer render riêng (canvas), giữ React Flow cho node — quyết tại arch review tháng 3 |
| 2 FE nghẽn (Visualizer + Dashboard + Designer + Alert UI) | Cao | Cao | FE velocity < 60% plan trong 2 sprint liên tiếp | Hoãn Designer (P5 co lại còn write actions + Alert UI + anomaly); Go dev đỡ phần BFF-for-frontend; dùng shadcn blocks thay vì tự thiết kế |
| OTel propagation qua Kafka đứt ở 1 ngôn ngữ | Trung bình | Cao | Integration test P1 fail kéo dài > 1 tuần | Fallback: tự đóng trace context vào message envelope (proto field) thay vì header — ADR-005 ghi sẵn phương án |
| Stakeholder yêu cầu kích hoạt lại copilot giữa kỳ | Trung bình | Trung bình | Yêu cầu chính thức xuất hiện trước tháng 5 | Chèn phase copilot read-only 4 tuần sau P5: anomaly + một phần Alert UI lùi thành stretch; nền RBAC/audit/approval của P5 giúp không tốn thời gian retrofit |
| Connector framework over-engineered ở P4 | Trung bình | Trung bình | Refactor 3 adapter cũ về SDK mất > 1 sprint; interface SDK đổi liên tục trong phase | Freeze interface ở đúng mức 3 case đã có; nguồn mới cứ viết "hơi tay" rồi hợp nhất sau — framework phục vụ nguồn, không phải ngược lại |
| Nguồn legacy (SOAP, Excel nhập tay) bẩn hơn mọi dự đoán | Cao | Trung bình | Nguồn #4/#5 trễ > 1 tuần ở P4; % mapping khai báo < 50% | Quarantine pattern: ingest thô vào staging table + DLQ, chuẩn hóa dần bằng mapping; không để 1 nguồn bẩn chặn các nguồn khác |
| Hệ thống nguồn thật bẩn/không ổn định hơn dự kiến | Cao | Trung bình | Adapter #2 trễ > 1 sprint ở P3 | Cắt còn 2 pipeline thật (bỏ #3), giữ nguyên DoD resilience; adapter #3 lùi sang P5 chạy song song |
| Anomaly detector nhiều false positive | Cao | Trung bình | Precision < 60% trong 2 tuần shadow ở P5 | Ship threshold-based alert (đã có từ P3) làm mặc định; anomaly ML giữ shadow mode, không bật alert — không phải thất bại, là dữ liệu chưa đủ |
| Bus factor: 1 stack chỉ còn 1 người active | Trung bình | Cao | 1 stack có 1 người nghỉ/quá tải > 2 tuần | Pairing rotation bắt buộc từ P1 (mỗi slice có 2 người chạm); runbook + README là DoD nên tri thức không nằm trong đầu 1 người; giảm scope phase kế thay vì dồn ép |

---

## 7. Câu hỏi đã chốt trước Phase 1 (Rev 3, 2026-07-04)

Toàn bộ 12 câu đã được stakeholder trả lời. Giữ nguyên câu hỏi gốc để truy vết lý do quyết định; đáp án + tác động ghi ngay dưới mỗi câu.

1. **Hệ thống nguồn đầu tiên cụ thể là gì?** → **MySQL**, cũng là hệ quản trị CSDL chính của tổ chức. Adapter #1 = MySQL incremental polling trước, CDC (Debezium) bổ sung sau nếu cần độ trễ giây (ADR-015). Walking skeleton P1 đổi từ "REST source" ban đầu sang "MySQL poller".
2. **Sink ưu tiên của pipeline đầu tiên?** → **MySQL** (không phải Postgres như giả định cũ). UPSERT theo Business Key + `UpdatedAt` để đảm bảo idempotency — áp dụng từ P1.
3. **Throughput thật dự kiến và kích thước message trung bình?** → MVP **50–200 msg/s**, message trung bình **5–20 KB**. Chưa tối ưu throughput lớn nhưng kiến trúc giữ khả năng mở rộng. Target load test P3/P6 hạ từ 500/1.000 msg/s xuống mức phản ánh đúng nhu cầu (P3 DoD 500 msg/s vẫn giữ làm headroom ≈ 2,5×, không đổi).
4. **Deploy target: đã có cụm K8s chưa, on-prem hay cloud nào?** → **MVP chạy trên ServBay** để phát triển/kiểm chứng; hạ tầng ngoài ServBay (Kafka, OTel, Jaeger, Prometheus, Loki, Grafana) qua docker-compose. Chuyển Docker/K8s cho production **sau khi MVP ổn định** (ngoài 6 tháng) — trong kỳ này không cần Helm/HPA/Vault; kiến trúc vẫn phải stateless để migrate dễ. K8s manifests dời khỏi P6 (mục 2, Phase 6).
5. **Tổ chức đang dùng SSO gì?** → **Chưa có SSO.** Keycloak nội bộ tự quản user; hỗ trợ federate Google Workspace/Microsoft Entra ID **sau này** (không phải P2). P2 vẫn dùng Keycloak nhưng scope federation IdP ngoài là OUT.
6. **Dữ liệu có PII không, áp quy định nào?** → **Có** dữ liệu học sinh, phụ huynh, nhân sự; tuân thủ **Nghị định 13/2023**; log bắt buộc masking dữ liệu nhạy cảm — không đổi so với thiết kế P1 (redaction trong log), nhưng nay là yêu cầu compliance chính thức, không còn là giả định.
7. **Danh sách đầy đủ 5–6 nguồn theo họ?** → Thứ tự chốt: **MySQL → REST API → SQL Server → Excel/SFTP → SOAP (nếu phát sinh)**. Phủ đủ 4 họ (database, API, file, legacy) cho mục tiêu ≥ 3 họ ở P4.
8. **5 user đầu tiên là ai, trực ca thế nào?** → **~5 người đội CNTT**. Kênh alert: **Email/Zalo** (không dùng Slack như giả định cũ) — áp dụng từ P3 (5 alert rule) và P5 (Alert Management UI).
9. **Quy trình phê duyệt thao tác vận hành hiện tại?** → **MVP chưa cần approval workflow phức tạp**, chỉ cần **RBAC Admin/Operator/Viewer**. Xác nhận thiết kế P5 hiện tại (3 role + tự-confirm 2 bước + audit, không phải duyệt qua người khác) là đủ — không cần thêm bước phê duyệt liên người vào ADR-017.
10. **Tổ chức đã có Prometheus/Grafana dùng chung chưa?** → Chưa; **giai đoạn MVP dùng logging + health check + Prometheus nếu cần**, chưa cần hệ thống monitoring phức tạp. Xác nhận P0–P2 dựng mới là đúng hướng, không cần tích hợp/né trùng nguồn sự thật.
11. **Cuối năm 1 dự kiến bao nhiêu pipeline?** → **20–50 pipeline**. Dưới ngưỡng cần canvas — Designer P5 chốt ở dạng **wizard/template**, chưa làm canvas kéo-thả (giải phóng capacity FE, giảm rủi ro "2 FE nghẽn").
12. **"Pilot production" tháng 7 nghĩa là gì?** → **Chạy shadow mode song song với hệ thống hiện tại** trước khi chuyển sang production — chưa thay thế nghiệp vụ thật. Go/no-go P6 đánh giá theo tiêu chí shadow-mode, không phải cutover.

---

## 8. Chốt trước khi kích hoạt lại AI Copilot

Copilot hoãn không đồng nghĩa mất tiến độ khi quay lại: RBAC, audit trail, approval token (P5) và SSE-ready BFF là những phần khó retrofit nhất và đều đã có. Kích hoạt lại copilot read-only ước tính **4 tuần** với 1 Python + 1 FE (scope như Phase 4 cũ của Rev 1: 4 tool read-only, RAG, eval harness, budget — dùng ADR-022..024). Trước khi bấm nút, phải chốt:

1. **Data residency**: log/trace/metric có được gửi ra API nước ngoài (Anthropic US) không — nếu không, phải self-host (vLLM + GPU, +3 tuần, procurement đi trước khi code).
2. **Budget LLM/tháng**: con số cụ thể — quyết định mix model (Sonnet cho reasoning, Haiku cho routing), prompt caching, ngưỡng token/user/ngày.
3. **PII policy đã duyệt**: quy tắc redaction được người chịu trách nhiệm dữ liệu ký, đặc biệt với dữ liệu học sinh theo Nghị định 13/2023.
4. **Bộ eval khởi điểm**: ngay từ bây giờ, log lại các câu hỏi vận hành mà ops thật sự hỏi nhau (Slack/ticket) — nguyên liệu miễn phí cho bộ eval 25 câu, thu thập được cả khi copilot đang hoãn.
