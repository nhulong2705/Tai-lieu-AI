# n8n — Phần 4: API, CLI & Tham khảo nhanh

> Biên soạn từ kiến thức tổng hợp về n8n (không fetch trực tiếp docs.n8n.io).
> Phạm vi: n8n Public REST API, n8n CLI, danh sách biến môi trường, bảo mật, cheat sheet tra cứu nhanh.

---

## Mục lục

1. [n8n Public REST API](#1-n8n-public-rest-api)
2. [n8n CLI (trong container/server)](#2-n8n-cli-trong-containerserver)
3. [Biến môi trường — tham khảo đầy đủ](#3-biến-môi-trường--tham-khảo-đầy-đủ)
4. [Bảo mật — checklist quan trọng](#4-bảo-mật--checklist-quan-trọng)
5. [Cheat sheet Expression](#5-cheat-sheet-expression)
6. [So sánh nhanh: khi nào dùng node nào](#6-so-sánh-nhanh-khi-nào-dùng-node-nào)
7. [FAQ](#7-faq)

---

## 1. n8n Public REST API

n8n có REST API riêng để quản lý workflow/execution/credential/user bằng HTTP — khác với việc **workflow gọi API ngoài** (đó là HTTP Request node). Đây là API để **điều khiển chính n8n từ bên ngoài** (ví dụ: từ Hermes/OpenClaw gọi vào n8n để tạo/kích hoạt workflow theo lệnh người dùng).

> ⚠️ API này **không khả dụng trong free trial** của n8n Cloud — cần bản trả phí hoặc self-host (self-host luôn có sẵn, miễn phí).

### Lấy API Key

```text
n8n UI → Settings → n8n API → Create an API Key
```
> Bản Enterprise có thể giới hạn **scope** cho từng key (chỉ đọc, chỉ workflow, không đụng credential...). Bản thường (Community) — key có toàn quyền trên mọi resource của account, cần bảo vệ như 1 secret thật sự.

### Base URL

```text
Self-host:    https://<n8n-domain>/api/v1
n8n Cloud:    https://<subdomain>.app.n8n.cloud/api/v1
```

### Authentication — header bắt buộc mọi request

```bash
curl https://n8n.example.com/api/v1/workflows \
  -H "X-N8N-API-KEY: <your_api_key>"
```

### Các endpoint chính

| Resource | Endpoint | Mô tả |
|---|---|---|
| Workflows | `GET /workflows` | Liệt kê workflow |
| | `GET /workflows/{id}` | Xem 1 workflow |
| | `POST /workflows` | Tạo workflow mới (body chứa `nodes`, `connections`, `settings`) |
| | `PUT /workflows/{id}` | Cập nhật workflow |
| | `DELETE /workflows/{id}` | Xoá workflow |
| | `POST /workflows/{id}/activate` | Kích hoạt (Active) |
| | `POST /workflows/{id}/deactivate` | Tắt Active |
| Executions | `GET /executions` | Liệt kê lần chạy (filter theo workflow, status) |
| | `GET /executions/{id}` | Xem chi tiết 1 lần chạy |
| | `DELETE /executions/{id}` | Xoá lịch sử 1 lần chạy |
| Credentials | `POST /credentials` | Tạo credential mới |
| | `DELETE /credentials/{id}` | Xoá credential |
| | *(Không có GET chi tiết — vì lý do bảo mật, API không cho đọc lại giá trị credential đã lưu)* | |
| Users | `GET /users`, `POST /users` | Quản lý user (cần User Management bật) |
| Tags | `GET /tags`, `POST /tags` | Quản lý tag gắn vào workflow |
| Variables | `GET /variables`, `POST /variables` | Quản lý `$vars` qua API |

### Ví dụ: lấy danh sách workflow đang Active

```bash
curl https://n8n.example.com/api/v1/workflows?active=true \
  -H "X-N8N-API-KEY: <your_api_key>"
```

### Ví dụ: kích hoạt 1 workflow từ xa (vd: từ script deploy)

```bash
curl -X POST https://n8n.example.com/api/v1/workflows/abc123/activate \
  -H "X-N8N-API-KEY: <your_api_key>"
```

### Gọi n8n API ngay trong 1 workflow n8n khác (node "n8n")

n8n có sẵn 1 node tên **"n8n"** để gọi API của chính nó (hoặc instance n8n khác) mà không cần dùng HTTP Request thủ công — tạo Credential kiểu "n8n API" với API Key + Base URL, rồi chọn resource/operation trực tiếp trong UI node.

### Dùng n8n API từ Python (ví dụ tích hợp với OpenClaw)

```python
import requests

N8N_BASE = "https://n8n.example.com/api/v1"
HEADERS = {"X-N8N-API-KEY": "your_api_key"}

def list_active_workflows():
    r = requests.get(f"{N8N_BASE}/workflows", params={"active": "true"}, headers=HEADERS)
    r.raise_for_status()
    return r.json()

def trigger_webhook(path: str, payload: dict):
    # Webhook không cần API key — chỉ cần URL webhook thật của workflow
    r = requests.post(f"https://n8n.example.com/webhook/{path}", json=payload)
    return r.json()
```

---

## 2. n8n CLI (trong container/server)

n8n có CLI riêng — thực chất là wrapper quanh REST API, tối ưu cho dùng trong terminal/CI-CD/script, kể cả gọi từ agent AI (Claude Code, Cursor...).

```bash
# Chạy bên trong container n8n, hoặc nếu cài qua npm thì gọi trực tiếp `n8n`
docker exec -it n8n-main n8n <command>
```

### Lệnh quản lý workflow

```bash
n8n export:workflow --all --output=backup/workflows.json
n8n export:workflow --id=123 --output=backup/single-workflow.json
n8n import:workflow --input=backup/workflows.json
n8n update:workflow --id=123 --active=true
```

### Lệnh quản lý credentials

```bash
n8n export:credentials --all --decrypted --output=backup/creds.json   # ⚠️ Xem cảnh báo Phần 3 mục 9
n8n import:credentials --input=backup/creds.json
```

### Lệnh vận hành server

```bash
n8n start                       # Chạy n8n (mặc định khi không truyền command)
n8n start --tunnel              # Chạy kèm tunnel tạm (test webhook nhanh, KHÔNG dùng production)
n8n worker                      # Chạy như Worker process (Queue Mode)
n8n webhook                     # Chạy như process chuyên nhận webhook (kiến trúc tách rời nâng cao)
n8n user-management:reset       # Reset toàn bộ user management (cẩn trọng — mất hết user/permission)
```

### Lệnh database

```bash
n8n db:revert         # Rollback migration DB gần nhất (dùng khi update lỗi)
```

---

## 3. Biến môi trường — tham khảo đầy đủ

### Cơ bản & Database

```bash
N8N_HOST=n8n.example.com
N8N_PORT=5678
N8N_PROTOCOL=https
N8N_ENCRYPTION_KEY=...                 # BẮT BUỘC đặt cố định, backup an toàn
WEBHOOK_URL=https://n8n.example.com/
GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
TZ=Asia/Ho_Chi_Minh

DB_TYPE=postgresdb                     # sqlite (mặc định) | postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=...
```

### Queue Mode

```bash
EXECUTIONS_MODE=queue                  # regular (mặc định) | queue
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
QUEUE_BULL_REDIS_PASSWORD=...
QUEUE_HEALTH_CHECK_ACTIVE=true
QUEUE_WORKER_CONCURRENCY=10
```

### Task Runners (n8n 2.0+)

```bash
N8N_RUNNERS_ENABLED=true
N8N_RUNNERS_MODE=external              # internal | external
N8N_RUNNERS_AUTH_TOKEN=...
```

### Bảo mật & Code node

```bash
N8N_BLOCK_ENV_ACCESS_IN_NODE=true      # Chặn $env trong Code node/expression
NODE_FUNCTION_ALLOW_EXTERNAL=moment,lodash   # Whitelist package npm dùng được trong Code node
NODE_FUNCTION_ALLOW_BUILTIN=crypto     # Whitelist module built-in Node.js
N8N_SECURE_COOKIE=true
```

### Tắt tính năng không cần (giảm gọi ra ngoài, tăng riêng tư)

```bash
N8N_METRICS=false
N8N_DIAGNOSTICS_ENABLED=false
N8N_PERSONALIZATION_ENABLED=false
N8N_HIRING_BANNER_ENABLED=false
N8N_VERSION_NOTIFICATIONS_ENABLED=false
```

### Execution data & pruning (tránh DB phình to)

```bash
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all    # all | none — lưu data thành công hay không
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_PRUNE=true             # Tự xoá execution cũ
EXECUTIONS_DATA_MAX_AGE=336             # Giờ — giữ tối đa 14 ngày (mặc định tham khảo)
```

### Webhook & Proxy

```bash
N8N_PUSH_BACKEND=websocket             # Cơ chế UI nhận update real-time
N8N_PROXY_HOPS=1                       # Số tầng reverse proxy phía trước (nginx/traefik = 1)
```

---

## 4. Bảo mật — checklist quan trọng

> ⚠️ **Cảnh báo thực tế:** đầu năm 2026 có báo cáo về lỗ hổng nghiêm trọng trên n8n self-host cho phép **remote code execution (RCE)** nếu attacker tạo/sửa được workflow. Vì n8n vốn được thiết kế để chạy code (Code node) và gọi lệnh hệ thống, **bất kỳ ai có quyền edit workflow gần như có quyền tương đương truy cập server** — phải coi n8n là hạ tầng nhạy cảm, không phải 1 app "vô hại".

### Checklist bảo mật bắt buộc khi self-host

- [ ] **Không** để Editor UI public ra Internet không bảo vệ — đặt sau VPN, Cloudflare Access/Zero Trust, hoặc ít nhất Basic Auth ở reverse proxy.
- [ ] Giới hạn ai có quyền tạo/sửa workflow — chỉ người tin tưởng (RCE risk nói trên).
- [ ] Luôn cập nhật n8n lên bản mới nhất khi có security advisory.
- [ ] Webhook endpoint (đường công khai duy nhất nên mở) — luôn có signature/secret xác thực (HMAC, header auth), thêm rate-limit ở reverse proxy/WAF.
- [ ] Xoay vòng (rotate) API key, OAuth token định kỳ.
- [ ] Đặt `N8N_BLOCK_ENV_ACCESS_IN_NODE=true` nếu không thực sự cần `$env` trong Code node.
- [ ] Backup `N8N_ENCRYPTION_KEY` ở nơi an toàn, tách biệt khỏi backup DB thường (không lưu chung 1 chỗ dễ bị đánh cắp cả 2).
- [ ] Theo dõi log truy cập bất thường (đăng nhập lạ, workflow bị sửa không rõ nguồn).

### Mô hình triển khai khuyến nghị cho cá nhân/dev (như tình huống của bạn)

```text
Internet → Cloudflare Tunnel (không mở port) → n8n (chỉ webhook path public, editor chỉ truy cập qua VPN/Tailscale)
```
Hoặc tối thiểu: đặt Basic Auth/Cloudflare Access trước toàn bộ domain n8n, chỉ riêng path `/webhook/...` cho phép public (nếu reverse proxy hỗ trợ rule theo path).

---

## 5. Cheat sheet Expression

```text
{{ $json.field }}                            Field của item hiện tại
{{ $json['field with space'] }}              Field có khoảng trắng
{{ $json.body.field }}                        Data từ Webhook (luôn nằm trong .body)
{{ $node["Node Name"].json.field }}           Lấy từ output gốc của 1 node bất kỳ
{{ $("Node Name").item.json.field }}          Cách viết mới, item tương ứng (matching item)
{{ $now.toISO() }}                            Thời gian hiện tại, ISO format
{{ $now.toFormat("dd/MM/yyyy") }}             Định dạng ngày tuỳ chỉnh
{{ $now.plus({ days: 7 }).toISODate() }}      +7 ngày
{{ $vars.MY_VAR }}                            Biến n8n Variables (UI, không cần restart)
{{ $env.MY_VAR }}                             Biến môi trường hệ thống (cần restart để đổi)
{{ $workflow.id }}                             ID workflow hiện tại
{{ $execution.id }}                            ID lần chạy hiện tại
{{ $itemIndex }}                               Vị trí item hiện tại trong mảng
{{ $json.status === "ok" ? "✅" : "❌" }}      Ternary (điều kiện)
{{ $jmespath($json.list, "[?age>`30`].name") }}  Query JSON phức tạp
```

```javascript
// Trong Code node — KHÔNG dùng {{ }}
const items = $input.all();          // Mode "Run Once for All Items"
return items.map(i => ({ json: { ...i.json, done: true } }));

return $input.item;                  // Mode "Run Once for Each Item" — trả lại chính item
```

---

## 6. So sánh nhanh: khi nào dùng node nào

| Cần làm gì | Dùng node |
|---|---|
| Đổi tên/thêm/sửa field đơn giản | **Set (Edit Fields)** |
| Lọc bớt item theo điều kiện | **Filter** |
| Rẽ 2 nhánh true/false | **IF** |
| Rẽ nhiều nhánh (>2) | **Switch** |
| Logic phức tạp, nhiều bước, cần biến tạm | **Code** |
| Gọi API không có node riêng | **HTTP Request** |
| Gộp 2 luồng data lại | **Merge** |
| Tách 1 array thành nhiều item | **Split Out** |
| Gộp nhiều item thành 1 array | **Aggregate** |
| Xử lý theo từng nhóm nhỏ (rate-limit API) | **Loop Over Items (Split in Batches)** |
| Tái sử dụng logic ở nhiều workflow | **Execute Workflow** (sub-workflow) |
| Custom response cho Webhook | **Respond to Webhook** |
| Bắt lỗi toàn workflow | **Error Trigger** (ở 1 workflow riêng, gán vào Settings) |
| Bắt lỗi từng node cụ thể | **On Error → Continue (error output)** của node đó |
| Trễ workflow 1 khoảng thời gian | **Wait** |
| Trích xuất dữ liệu từ HTML/text thô | **HTML Extract** hoặc **Code + regex** |
| Gọi LLM / xây AI Agent | **AI Agent node** (LangChain-based, chọn Chat Model + Tools) |

---

## 7. FAQ

**n8n có miễn phí không?**
Self-host là mã nguồn mở, miễn phí (fair-code license — có vài giới hạn về việc bán lại như SaaS cạnh tranh trực tiếp). n8n Cloud là dịch vụ trả phí theo execution.

**Self-host n8n có cần PostgreSQL không?**
Không bắt buộc cho dùng cá nhân (SQLite đủ), nhưng **bắt buộc** nếu dùng Queue Mode hoặc nhiều người dùng đồng thời.

**Webhook test và webhook production khác gì nhau?**
Test chỉ chạy khi đang mở editor + bấm nghe thử; production chỉ chạy khi workflow đã **Active**. Lỗi phổ biến nhất khi mới dùng n8n là quên đổi sang URL production.

**Code node JS hay Python tốt hơn?**
JavaScript — chạy native, nhanh, đầy đủ built-in. Python chạy qua Pyodide (WASM), chậm hơn và giới hạn package — chỉ chọn khi thực sự cần cú pháp Python quen thuộc cho logic đơn giản.

**Làm sao gọi n8n từ Hermes/OpenClaw?**
2 cách: (1) Webhook — OpenClaw gửi HTTP POST tới URL webhook của workflow n8n (đơn giản nhất, không cần API key); (2) REST API — nếu cần OpenClaw tự tạo/sửa/kích hoạt workflow n8n theo lệnh người dùng, dùng `X-N8N-API-KEY` gọi `/api/v1/...`.

**n8n có an toàn để expose webhook ra Internet không?**
Có thể, nhưng **bắt buộc** có xác thực (HMAC/secret) ở mức webhook, và **không** để Editor UI public không bảo vệ — xem Phần 4 mục 4.

---

*Tài liệu biên soạn tổng hợp — tham khảo thêm tại docs.n8n.io/api/ và docs.n8n.io/hosting/configuration/environment-variables/ khi cần xác minh chi tiết version mới nhất.*
