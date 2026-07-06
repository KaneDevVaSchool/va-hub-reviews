# BRIEF v2: Hệ thống ESB / Middleware Hub / Data Pipeline + Pipeline Visualization Platform

> **Thay đổi so với v1:** Bổ sung tầng **Web UI trực quan hóa pipeline real-time**, **AI Copilot** để vận hành/hỏi đáp hệ thống, chuẩn animation/UX cho luồng dữ liệu, và mở rộng chức năng quản trị pipeline qua giao diện (no-code/low-code).

---

## 1. Mục tiêu (mở rộng)

Ngoài các mục tiêu cũ (tích hợp đa hệ thống, real-time/batch, anomaly detection, tracing xuyên suốt), hệ thống bổ sung:

1. **Pipeline Visualization Web** — hiển thị toàn bộ topology pipeline dạng đồ thị động (DAG), có animation dòng chảy dữ liệu real-time, trạng thái từng node, throughput, latency, error rate.
2. **AI Copilot** — giao diện chat tích hợp trong web, cho phép hỏi tự nhiên: *"Vì sao consumer X đang lag?"*, *"Trace giúp tôi request có ID Y"*, *"Tóm tắt các anomaly trong 1 giờ qua"*, *"Tạo pipeline mới từ Kafka topic A sang Snowflake"*.
3. **Pipeline Designer (low-code)** — kéo thả node để tạo/sửa pipeline, generate ra config (YAML/JSON) và deploy tự động.
4. **Replay & Time-travel** — chọn một khoảng thời gian trong quá khứ và xem lại luồng dữ liệu như video tua lại.
5. **Alerting UI** — cấu hình alert rule qua giao diện, không cần sửa file Prometheus.

---

## 2. Kiến trúc tổng thể (v2)

```
┌────────────────────────────────────────────────────────────────────┐
│                      WEB UI (Next.js + React)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────┐ │
│  │ Pipeline     │  │ Live Metrics │  │ AI Copilot   │  │ Designer│ │
│  │ Visualizer   │  │ Dashboard    │  │ (Chat)       │  │ (DnD)   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────┘ │
└────────┬───────────────┬──────────────────┬──────────────┬─────────┘
         │ WebSocket     │ REST/GraphQL     │ SSE stream   │ REST
         ▼               ▼                  ▼              ▼
┌────────────────────────────────────────────────────────────────────┐
│                    BFF Layer (Go — Fiber/Echo)                      │
│   Aggregation • Auth (OIDC) • Rate limit • WebSocket fan-out        │
└────────┬────────────────────────────────┬──────────────────────────┘
         │                                │
         ▼                                ▼
┌──────────────────────┐        ┌────────────────────────────────────┐
│  Core Data Plane     │        │  AI Copilot Service (Python)       │
│  (Go adapters,       │        │  • LLM orchestration (LangGraph)   │
│   Python ETL,        │        │  • RAG trên trace/log/metric       │
│   Kafka, Airflow…)   │        │  • Tool calling: query Prom/Jaeger │
└──────────┬───────────┘        │  • Guardrails                      │
           │                    └────────────────────────────────────┘
           ▼
┌──────────────────────────────────────────────────────────────────┐
│    Observability (OTel Collector → Jaeger / Prometheus / Loki)   │
│    + Anomaly Detection (Isolation Forest / Prophet / LSTM AE)    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Web UI — Nguyên tắc thiết kế

### 3.1 Design system

| Hạng mục | Chuẩn |
|---|---|
| Framework | **Next.js 14+ (App Router)** + React 18 + TypeScript strict |
| Styling | **Tailwind CSS** + `shadcn/ui` (Radix primitives) |
| Icon | `lucide-react` |
| Theme | Dark mode mặc định (developer/ops audience), light mode tùy chọn; palette dựa trên `zinc/slate` + accent màu semantic (green=healthy, amber=warning, red=error, blue=info) |
| Typography | `Inter` cho UI, `JetBrains Mono` cho log/code/ID |
| Layout | Sidebar cố định + main canvas + right drawer cho detail; responsive breakpoints ≥ 1280px ưu tiên (audience desktop) |

### 3.2 Pipeline Visualizer (component trọng tâm)

**Thư viện:** `React Flow` (xyflow) — chuẩn de-facto cho node-based UI, hỗ trợ custom node, minimap, zoom, pan, và edge animation.

**Yêu cầu chức năng:**

- **Node types tùy biến:** Source, Adapter (Go), Router, Transform (Python), Sink, Broker (Kafka topic), DLQ. Mỗi node có icon riêng, badge trạng thái (running/idle/error), sparkline throughput 60s cuối.
- **Edge animation:** dòng chảy dữ liệu render bằng `<animate>` SVG hoặc CSS `stroke-dashoffset` animation. Tốc độ animation tỉ lệ với throughput thực (msg/s) — throughput cao thì "hạt" chạy nhanh và dày hơn. Khi có backpressure, animation chậm lại và edge đổi màu amber.
- **Live update:** subscribe qua WebSocket, cập nhật node status/edge throughput mỗi 1s không reload page. Dùng `zustand` hoặc `jotai` cho state, tránh Redux boilerplate.
- **Hover/click node** → mở right drawer hiển thị: metrics chi tiết (p50/p95/p99 latency, error rate), trace mẫu gần nhất, log tail, config hiện tại, nút "Restart" / "Scale" / "View in Jaeger".
- **Minimap + zoom controls** cho pipeline lớn (>50 node).
- **Group/collapse** theo domain (ví dụ gom cụm ETL của một tenant lại).

### 3.3 Animation & motion

| Yếu tố | Cách làm |
|---|---|
| Data flow trên edge | SVG `<animateMotion>` với circle chạy dọc path, hoặc CSS animated dash |
| Node pulse khi có event | `framer-motion` `animate={{ scale: [1, 1.05, 1] }}` khi nhận message |
| State transition (idle → running → error) | Color morph 300ms ease-out |
| Anomaly alert | Node shake nhẹ + halo đỏ pulse, giới hạn ≤ 3s để không gây khó chịu |
| Page/route transition | `framer-motion` `AnimatePresence`, fade + slide 200ms |
| Loading skeleton | `shadcn/ui` skeleton, không dùng spinner cho block lớn |
| Micro-interaction | Button press, drawer open/close 150–200ms, easing `cubic-bezier(0.4, 0, 0.2, 1)` |

**Nguyên tắc:** animation phục vụ ngữ nghĩa (hiểu hệ thống đang làm gì), không phải trang trí. Tắt được toàn bộ qua `prefers-reduced-motion`.

### 3.4 Live Metrics Dashboard

- Grid card có thể kéo thả/resize (dùng `react-grid-layout`).
- Chart: `Recharts` cho line/area/bar chuẩn; `visx` hoặc `d3` khi cần custom (ví dụ flame graph, heatmap).
- Data source: query trực tiếp Prometheus qua BFF (không expose Prom cho browser).
- Auto-refresh 5s, cho phép pause khi user đang phân tích.

### 3.5 Pipeline Designer (low-code)

- Cùng `React Flow` nhưng ở chế độ editable.
- Palette bên trái: các node type kéo được vào canvas.
- Property panel bên phải: form config theo JSON Schema, validate real-time.
- Xuất ra: file YAML/JSON deploy được bằng Airflow DAG hoặc config service.
- Version control: mỗi lần save tạo revision, có thể diff và rollback.

---

## 4. AI Copilot — Chi tiết

### 4.1 UX

- Chat panel dạng drawer bên phải, có thể pin.
- Streaming response (SSE), render markdown, code block có copy button.
- Message có thể chứa **rich attachment**: mini chart, trace timeline, link tới node trong Visualizer (click là focus node đó).
- Suggested prompts theo context (đang xem trang nào thì gợi ý câu hỏi liên quan).

### 4.2 Kiến trúc

- **Backend:** Python + FastAPI + `LangGraph` (workflow bền vững, có state) hoặc `PydanticAI` (nhẹ hơn).
- **LLM:** Claude Sonnet 4.6 mặc định qua Anthropic API. Cho phép cấu hình đổi model.
- **Tool calling** — copilot có các tool:
  - `query_prometheus(promql, range)` — chạy PromQL.
  - `search_traces(service, error, time_range)` — Jaeger API.
  - `tail_logs(service, filter, lines)` — Loki API.
  - `get_pipeline_topology()` — trả về DAG hiện tại.
  - `describe_anomaly(anomaly_id)` — join thông tin từ anomaly detector.
  - `restart_service(name)` / `scale_service(name, replicas)` — **cần confirm 2 bước từ user**, có RBAC.
- **RAG:** index runbook, ARCHITECTURE.md, README từng service vào vector DB (Qdrant/pgvector) để copilot trả lời "cách xử lý sự cố X".
- **Guardrails:** không cho copilot tự ý gọi tool ghi (restart/scale/delete) mà không có approval; log toàn bộ tool call vào audit trail.

### 4.3 Prompt system cho copilot

Copilot được prime bằng: vai trò (SRE assistant cho platform này), danh sách service, quy ước đặt tên, các runbook chính. Mọi câu trả lời phải cite nguồn (trace ID, log timestamp, metric query đã chạy).

---

## 5. Công nghệ Frontend (bổ sung vào bảng cũ)

| Hạng mục | Lựa chọn |
|---|---|
| Framework | Next.js 14 App Router, React Server Components khi hợp lý |
| Ngôn ngữ | TypeScript strict mode, `zod` cho runtime validation |
| State | `zustand` cho global UI state, `TanStack Query` cho server state |
| Realtime | WebSocket (native) + `socket.io` fallback; SSE cho AI streaming |
| Node graph | `React Flow` (xyflow) |
| Animation | `framer-motion` + CSS animations, tránh JS animation loop khi có thể |
| Chart | Recharts (chuẩn), visx/d3 (nâng cao) |
| Grid dashboard | `react-grid-layout` |
| Form | `react-hook-form` + `zod` resolver |
| Table | `TanStack Table` |
| i18n | `next-intl` — tiếng Việt mặc định, English tùy chọn |
| Auth | NextAuth.js với OIDC (Keycloak/Auth0) |
| Testing | Vitest + Testing Library + Playwright (E2E) |
| Perf | Bundle analyzer, dynamic import cho Visualizer, virtualize list dài |
| Accessibility | WCAG 2.1 AA, keyboard nav đầy đủ cho canvas React Flow |

---

## 6. Chức năng mở rộng

### 6.1 Replay & Time-travel
- Chọn khoảng thời gian → BFF query trace + metric history → Visualizer phát lại animation theo timeline như video (play/pause/seek/speed 1x-8x).
- Hữu ích cho post-mortem: "10 phút trước sự cố, luồng dữ liệu trông thế nào?"

### 6.2 Data lineage view
- Chuyển từ topology sang góc nhìn **dữ liệu**: một record/message đã đi qua node nào, transform gì. Kết hợp OpenLineage + Marquez nếu cần chuẩn.

### 6.3 Alert Management UI
- Tạo/sửa alert rule (PromQL + threshold hoặc anomaly-based) qua form.
- Test rule với dữ liệu lịch sử trước khi apply.
- Route alert đến kênh (Slack/Email/PagerDuty) với escalation policy cấu hình được.

### 6.4 Access Control UI
- Quản lý user/role/permission cho pipeline (viewer, operator, admin).
- Audit log mọi hành động ghi (deploy pipeline, restart service, đổi alert rule).

### 6.5 Cost/resource insight
- Widget hiển thị CPU/mem/network của từng service, ước tính chi phí cloud tương ứng (nếu deploy AWS/GCP).

### 6.6 Chaos toggle (dev/staging)
- Nút inject fault (latency, error, kill pod) để test resilience — chỉ enable ở non-prod, gated bằng RBAC.

---

## 7. Cấu trúc monorepo (v2)

```
/apps
  /web                    # Next.js UI
  /copilot                # Python FastAPI + LangGraph
/services
  /go-adapters
  /go-router
  /go-bff                 # BFF layer cho web
/pipelines
  /python-etl
/orchestration
  /airflow-dags
/observability
  /otel-collector
  /anomaly-detector
/packages
  /ui                     # shared React components (shadcn based)
  /schemas                # JSON Schema + zod + protobuf
  /sdk-ts                 # TS client cho BFF API
/shared
  /proto
ARCHITECTURE.md
CLAUDE.md
```

---

## 8. Yêu cầu Production cho tầng Web/AI (bổ sung)

- **Bảo mật:** CSP nghiêm ngặt, không inline script; auth qua OIDC + short-lived token; mọi tool call của copilot ghi audit; PII redaction trước khi gửi log/trace vào LLM prompt.
- **Performance:** LCP < 2.5s, INP < 200ms; Visualizer render mượt với ≥ 200 node (dùng virtualization của React Flow); WebSocket reconnect exponential backoff.
- **Reliability:** BFF stateless, scale ngang; nếu Copilot down thì UI vẫn dùng được (graceful degradation).
- **Cost control LLM:** cache prompt phổ biến, giới hạn token/user/ngày, hiển thị usage cho admin.
- **Observability của chính UI:** Sentry cho FE error, Web Vitals gửi về Prometheus.

---

## 9. Checklist bổ sung (Frontend/AI)

- [ ] Design tokens tập trung, không hardcode màu
- [ ] Toàn bộ animation tôn trọng `prefers-reduced-motion`
- [ ] Keyboard accessible cho canvas (tab/arrow di chuyển giữa node)
- [ ] i18n từ ngày đầu (không hardcode string tiếng Việt trong component)
- [ ] LLM prompt và tool schema versioned, có eval regression
- [ ] Rate limit + circuit breaker cho gọi LLM
- [ ] Redact secret/PII trong mọi payload gửi copilot
- [ ] E2E test luồng: xem pipeline → hỏi copilot → nhận trace → focus node
- [ ] Load test WebSocket với 1000+ client đồng thời

---

## 10. Thông tin cần bổ sung để triển khai chi tiết (mở rộng)

Ngoài các câu hỏi cũ (hệ thống nguồn/đích, SLA, môi trường deploy…), cần thêm:

- **Audience của Web UI:** chỉ ops/SRE nội bộ, hay có cả business user (ảnh hưởng phức tạp UX và low-code designer)?
- **Số lượng pipeline & node dự kiến** (100 hay 10.000 — quyết định chọn React Flow thường hay phải custom canvas WebGL).
- **Có bắt buộc self-host LLM không** (data sensitivity) — nếu có thì bổ sung vLLM + Llama 3.x thay Anthropic API.
- **Ngân sách LLM/tháng** — quyết định model tier (Sonnet vs Haiku, mở cache hay không).
- **Yêu cầu compliance:** SOC2, ISO 27001, GDPR — ảnh hưởng audit log, data residency, RBAC granularity.
- **Mobile/tablet có cần không** — mặc định desktop-first, nếu cần mobile thì cắt tính năng Designer.

---

**Ghi chú sử dụng:** Đây là brief tổng, dùng làm nguồn duy nhất cho các prompt con (ví dụ: "sinh scaffold Next.js theo mục 3 và 5", "viết SKILL.md cho pipeline-visualizer-component", "thiết kế schema WebSocket event giữa BFF và Web"). Khi có đủ thông tin ở mục 10, tách brief này thành các PRD/BRD nhỏ theo domain (Web, Copilot, Data Plane) để triển khai song song.
