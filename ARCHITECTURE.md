# Kiến trúc VA Hub

Tài liệu kiến trúc hiện hành của repo. Nguồn yêu cầu gốc: [BRIEF-v2.md](BRIEF-v2.md); kế hoạch triển khai: [DELIVERY-PLAN.md](DELIVERY-PLAN.md) — khi các tài liệu lệch nhau, file này phản ánh quyết định mới nhất.

> **Rev 2 (2026-07-04):** AI Copilot tạm hoãn; ưu tiên số 1 là tích hợp & ETL từ nhiều nguồn khác nhau (connector SDK + ETL mapping khai báo).
>
> **Rev 3 (2026-07-04):** stakeholder đã trả lời đủ 12 câu hỏi mở (chi tiết: [DELIVERY-PLAN.md mục 7](DELIVERY-PLAN.md)). Quyết định ảnh hưởng kiến trúc: nguồn #1 và sink #1 đều là **MySQL** (UPSERT theo business key + `UpdatedAt`); MVP chạy trên **ServBay**, chuyển Docker/K8s cho production sau khi MVP ổn định (chưa cần Helm/HPA/Vault, giữ nguyên tắc stateless); Keycloak tự quản user (chưa federate SSO ngoài); alert qua **Email/Zalo**; RBAC Admin/Operator/Viewer không cần approval đa cấp; Designer chốt dạng wizard/template; pilot tháng 7 = shadow mode song song hệ hiện tại.

## Tổng quan

VA Hub là nền tảng ESB / Middleware Hub / Data Pipeline kèm tầng trực quan hóa: tích hợp & ETL đa hệ thống (real-time + batch — ưu tiên số 1), tracing xuyên suốt, anomaly detection, và một Web UI hiển thị topology pipeline dạng DAG động. AI Copilot vận hành nằm trong scope dài hạn nhưng đang tạm hoãn.

## Sơ đồ tổng thể

```text
Web UI (Next.js)  ──WebSocket/REST/SSE──►  BFF (Go)
                                            │
                        ┌───────────────────┼──────────────────┐
                        ▼                                      ▼
              Core Data Plane                        AI Copilot Service (Python)
        (Go adapters, Go router,                     — TẠM HOÃN (2026-07-04) —
         Python ETL, Kafka, Airflow)                 FastAPI + LangGraph, tool calling + RAG
                        │
                        ▼
        Observability: OTel Collector → Jaeger / Prometheus / Loki
        + Anomaly Detection (Isolation Forest / Prophet / LSTM AE)
```

Sơ đồ chi tiết và mô tả từng luồng: mục 2 của brief.

## Thành phần

| Đường dẫn | Stack | Vai trò |
| --- | --- | --- |
| `apps/web` | Next.js 14 App Router, TS strict, Tailwind + shadcn/ui, React Flow | Pipeline Visualizer, Live Metrics Dashboard, Pipeline Designer, Alert UI; Copilot chat khi kích hoạt lại |
| `apps/copilot` | Python, FastAPI, LangGraph, Anthropic API | **Tạm hoãn (2026-07-04)** — LLM orchestration, tool calling, RAG, guardrails + audit; giữ placeholder, điều kiện kích hoạt lại ở mục 8 của DELIVERY-PLAN |
| `services/go-bff` | Go (Fiber/Echo) | Aggregation, auth OIDC, rate limit, WebSocket fan-out cho web; browser không gọi trực tiếp Prometheus/Jaeger |
| `services/go-adapters` | Go | Adapter kết nối hệ thống nguồn/đích vào data plane |
| `services/go-router` | Go | Định tuyến message giữa các pipeline |
| `pipelines/python-etl` | Python | Transform/ETL batch và streaming |
| `infra/orchestration/airflow-dags` | Airflow | Điều phối batch pipeline; đích deploy của Pipeline Designer |
| `infra/observability/otel-collector` | OTel Collector config | Thu thập trace/metric/log về Jaeger/Prometheus/Loki |
| `infra/observability/anomaly-detector` | Python | Isolation Forest / Prophet / LSTM AE trên metric stream |
| `packages/ui` | React + shadcn/ui | Component dùng chung giữa các app web |
| `packages/schemas` | JSON Schema + zod + protobuf | Hợp đồng dữ liệu dùng chung (event WebSocket, config pipeline, API) |
| `packages/sdk-ts` | TypeScript | Client cho BFF API, generate từ schemas |
| `shared/proto` | Protobuf | Định nghĩa message giữa các service Go/Python |

## Luồng chính

1. **Live topology:** Data plane phát trạng thái node/edge → BFF tổng hợp → fan-out qua WebSocket → `apps/web` cập nhật Visualizer mỗi 1s (không reload).
2. **Metrics:** Web query qua BFF (REST) → BFF proxy sang Prometheus. Prometheus/Jaeger/Loki không bao giờ expose cho browser.
3. **Write actions:** hành động ghi (restart/scale/requeue, deploy pipeline) đi từ UI qua BFF với approval token 2 bước + RBAC + audit log. *(Copilot tạm hoãn — khi kích hoạt lại: Web mở SSE tới BFF → `apps/copilot`, tool ghi tái dùng đúng cơ chế approval/audit này; copilot down không ảnh hưởng phần UI còn lại.)*
4. **Designer:** Web tạo/sửa DAG → xuất config YAML/JSON có version → deploy qua Airflow hoặc config service; mỗi lần save tạo revision, diff/rollback được.

## Quyết định công nghệ đã chốt

- **Monorepo tooling:** pnpm workspace + Turborepo cho phần JS/TS (`apps/web`, `packages/*`). Service Go là các Go module độc lập — sẽ thêm `go.work` ở root khi service Go đầu tiên có code. Python app quản lý dependency riêng trong thư mục của nó.
- **Frontend:** theo bảng ở mục 3 và 5 của brief (React Flow, zustand + TanStack Query, framer-motion, next-intl với tiếng Việt mặc định, NextAuth OIDC, Vitest + Playwright).
- **Realtime:** WebSocket native (socket.io fallback), SSE riêng cho AI streaming.
- **Tích hợp đa nguồn (ưu tiên số 1):** nguồn #1 = MySQL (incremental polling, CDC bổ sung sau nếu cần độ trễ giây), sink #1 = MySQL (UPSERT theo business key + `UpdatedAt`); thứ tự nguồn tiếp theo: REST API → SQL Server → Excel/SFTP → SOAP. Adapter viết tay cho 3 nguồn đầu, sau đó chưng cất thành Connector SDK (Go) + ETL mapping khai báo YAML (Python) — chi tiết ở Phase 4 của DELIVERY-PLAN.
- **Deploy target:** MVP chạy trên **ServBay**; hạ tầng ngoài ServBay (Kafka, OTel Collector, Jaeger, Prometheus, Loki, Grafana) qua docker-compose. Docker/K8s cho production chỉ sau khi MVP ổn định — chưa cần Helm/HPA/Vault trong 6 tháng đầu; mọi service vẫn phải stateless để migrate rẻ.
- **AuthN:** Keycloak nội bộ, tự quản user; federate Google Workspace/Microsoft Entra ID để sau, chưa có yêu cầu.
- **Alerting:** kênh Email/Zalo (không dùng Slack).
- **LLM:** tạm hoãn tích hợp (2026-07-04). Khi kích hoạt lại: Anthropic API, client bọc interface provider-agnostic, đổi model qua config.

## Giả định đang áp dụng

12 câu hỏi mở ở mục 7 DELIVERY-PLAN đã được stakeholder trả lời ở Rev 3 (2026-07-04) — xem [DELIVERY-PLAN.md mục 7](docs/DELIVERY-PLAN.md). Bảng dưới chỉ còn các giả định thật sự chưa có câu trả lời chính thức:

| Câu hỏi mở | Giả định hiện tại |
| --- | --- |
| Audience Web UI | Ops/SRE nội bộ (đội CNTT ~5 người); desktop-first ≥ 1280px, không tối ưu mobile |
| Quy mô | ≤ ~500 node/topology → React Flow + virtualization là đủ, chưa cần canvas WebGL |
| LLM hosting | Copilot tạm hoãn; câu hỏi self-host/data residency chốt trước khi kích hoạt lại (mục 8 DELIVERY-PLAN) |
| Compliance | Chưa có yêu cầu SOC2/ISO/GDPR chính thức ngoài Nghị định 13/2023 (đã chốt — PII học sinh/phụ huynh/nhân sự); vẫn giữ audit log cho mọi hành động ghi |
| Môi trường dev | **ServBay.dev** MySQL `:3306` (user `root`) + docker compose cho Kafka/obs — [docs/SERVBAY.md](docs/SERVBAY.md); MySQL container `:3307` cho CI/không ServBay |

## Trạng thái hiện tại

**Phase 0 (Contracts & Dev Platform) — đã đóng DoD.** Onboarding: [PHASE-0.md](PHASE-0.md).

- **Dev stack 1 lệnh:** `make dev-up` (`docker compose up -d --build --wait`) — 4 skeleton service + Kafka, MySQL, OTel Collector, Jaeger, Prometheus, Loki, Grafana; healthcheck trên skeleton service.
- **4 service skeleton:** `/healthz`, `/readyz`, `/metrics`, structured JSON log (`service`, `severity`, `trace_id`); Go dùng `shared/go-kit`; Python `logging_setup`; Web log JSON trên route `/api/*`.
- **Contract codegen (ADR-003):** proto → Go/Python; zod → JSON Schema + **OpenAPI sinh từ zod** (`generate:openapi-spec`); OpenAPI → TS + Go types. `make generate` + CI `generate-check`; `buf lint` trên proto.
- **Vertical slice P0:** `/status` qua BFF; aggregator gồm **mysql** (TCP) cùng kafka/jaeger/prometheus/loki.
- **MessageBus (ADR-002):** `shared/bus/` — interface P0, Kafka impl P1.
- **CI/CD:** GitLab CI + golangci-lint v2 + SAST + Secret Detection. ADR-001..004, MR template, CODEOWNERS, [TECH-DEBT.md](TECH-DEBT.md), [MOCKS.md](MOCKS.md).

**Ports dev:** web `:3000`, go-bff `:8080`, go-adapters `:8081`, python-etl `:8082`, Grafana `:3001`, Jaeger `:16686`, Prometheus `:9090`, Loki `:3100`, **ServBay MySQL `:3306`**, Docker MySQL `:3307`, Kafka `:29092`.

**Tiếp theo:** Phase 1 walking skeleton — [DELIVERY-PLAN.md](DELIVERY-PLAN.md). `apps/copilot` tạm hoãn.

### Nợ kỹ thuật còn mở (P0 cho phép + follow-up)

Chi tiết đủ 5 field: [TECH-DEBT.md](TECH-DEBT.md). Tóm tắt: Kafka dev không auth; OTel Collector pin 0.112.0; SAST default ruleset; ServBay thay K8s; distroless debug cho healthcheck dev.
