# n8n — Phần 3: Tự động hoá & Tích hợp

> Biên soạn từ kiến thức tổng hợp về n8n (không fetch trực tiếp docs.n8n.io).
> Phạm vi: Webhook nâng cao, Credentials, Sub-workflow, Queue Mode (Redis/Workers/Task Runners), Self-host production (Docker Compose + reverse proxy + SSL).

---

## Mục lục

1. [Webhook nâng cao](#1-webhook-nâng-cao)
2. [Credentials — quản lý xác thực](#2-credentials--quản-lý-xác-thực)
3. [Sub-workflow — gọi workflow từ workflow khác](#3-sub-workflow--gọi-workflow-từ-workflow-khác)
4. [Variables & Environment](#4-variables--environment)
5. [Queue Mode — scale production](#5-queue-mode--scale-production)
6. [Task Runners (n8n 2.0+)](#6-task-runners-n8n-20)
7. [Self-host Production đầy đủ](#7-self-host-production-đầy-đủ)
8. [Reverse Proxy & SSL](#8-reverse-proxy--ssl)
9. [Backup & Migration](#9-backup--migration)

---

## 1. Webhook nâng cao

### Respond to Webhook — tự custom response

Mặc định Webhook trigger trả response tự động (theo Response Mode đã chọn). Muốn kiểm soát chính xác status code/body/header trả về:

```text
Webhook node → Response Mode: "Using 'Respond to Webhook' Node"
...
[xử lý logic]
...
Respond to Webhook node:
  Response Code: 200
  Response Body: {{ JSON.stringify({ status: "ok", id: $json.id }) }}
  Response Headers: Content-Type: application/json
```
> Chỉ dùng được **1 lần** mỗi execution — nếu workflow có nhiều nhánh dẫn tới nhiều "Respond to Webhook", chỉ nhánh chạy tới trước mới có hiệu lực.

### Xác thực Webhook đến (Authentication)

| Kiểu | Cách hoạt động |
|---|---|
| **None** | Ai có URL cũng gọi được — chỉ dùng khi có lớp bảo vệ khác (VD: HMAC tự kiểm tra trong Code node) |
| **Basic Auth** | Username/password chuẩn HTTP Basic |
| **Header Auth** | Kiểm tra 1 header cụ thể khớp giá trị bí mật |
| **JWT Auth** | Verify JWT token gửi kèm |

### Tự verify HMAC signature (pattern phổ biến khi tích hợp với hệ thống tự build)

```javascript
// Code node, đặt ngay sau Webhook
const crypto = require('crypto');
const secret = $env.WEBHOOK_SECRET;   // cần bật quyền $env nếu self-host chặn
const signature = $json.headers['x-hmac-signature'];
const body = JSON.stringify($json.body);

const expected = crypto.createHmac('sha256', secret).update(body).digest('hex');

if (signature !== expected) {
  throw new Error('Invalid signature');
}

return $input.all();
```

### Nhiều HTTP method trên 1 webhook

Webhook node hỗ trợ chọn 1 method, nhưng có thể thêm nhiều **Webhook node** trỏ cùng 1 path với method khác nhau (GET/POST riêng) trong cùng workflow, hoặc dùng node "Switch" theo `{{ $json.headers['x-http-method'] }}` nếu cần logic chung.

### Webhook nhận file (multipart/form-data)

```text
Webhook node → Options → "Binary Data": bật
→ File nhận được nằm ở $binary, truy cập qua node sau bằng:
  {{ $binary.data.fileName }}
  {{ $binary.data.mimeType }}
```

### Test URL vs Production URL — nhắc lại điểm quan trọng

```text
/webhook-test/...  → Chỉ hoạt động khi đang mở editor + bấm "Listen for test event" (1 lần)
/webhook/...        → Chỉ hoạt động khi workflow đã Active (toggle ở top bar)
```
> Lỗi rất phổ biến: tích hợp app ngoài bằng URL test, sau đó quên đổi sang URL production khi deploy thật — webhook sẽ "im lặng không nhận gì" vì test URL hết hiệu lực ngay khi đóng editor.

---

## 2. Credentials — quản lý xác thực

Credential lưu **mã hoá riêng** trong DB (dùng `N8N_ENCRYPTION_KEY`), tách biệt khỏi workflow — 1 credential dùng lại được cho nhiều workflow/node, và khi share workflow, credential **không** bị lộ.

### Các kiểu Credential phổ biến

| Kiểu | Dùng cho |
|---|---|
| **API Key** | Hầu hết API REST đơn giản (header hoặc query param) |
| **OAuth2** | Google, Microsoft, Slack, GitHub... (cần redirect URL đúng) |
| **Basic Auth** | Username/password chuẩn |
| **Generic Credential (HTTP Header Auth)** | Khi node không có credential type riêng — tự định nghĩa header |
| **Predefined Credential Type** | n8n có sẵn cấu hình cho hàng trăm app (Telegram, Slack, OpenAI...) |

### Tạo Credential cho node không có sẵn (Generic HTTP)

```text
HTTP Request node → Authentication: Generic Credential Type → Header Auth
  Name: Authorization
  Value: Bearer sk-xxxxxxxxxxxx
```

### OAuth2 Redirect URL — lưu ý khi self-host

Khi đăng ký OAuth app (Google, Slack...), Redirect URL phải khớp:
```text
https://your-n8n-domain.com/rest/oauth2-credential/callback
```
> Nếu chạy local (`localhost:5678`) sẽ không nhận được callback từ nhiều provider yêu cầu HTTPS — cần dùng domain thật hoặc tunnel (ngrok/Cloudflare Tunnel) khi test OAuth.

### Credential Sharing & Permission (n8n có hệ thống user/role)

Trong bản có quản lý nhiều user (Team/Enterprise hoặc self-host bật user management): Credential có thể giới hạn ai được dùng, tách biệt theo Project — tránh việc 1 người vô tình "mượn" credential của người khác qua workflow chia sẻ.

---

## 3. Sub-workflow — gọi workflow từ workflow khác

Dùng node **"Execute Workflow"** (hoặc "Execute Sub-workflow" ở bản mới) để tách logic dùng lại nhiều lần ra 1 workflow riêng — giống gọi function trong code.

```text
Execute Workflow node:
  Source: Database (chọn workflow đã lưu) | Local File | Parameter (JSON inline) | URL
  Workflow: "Send Telegram Notification"
  Input: truyền data vào qua Execute Workflow Trigger của workflow con
```

### Workflow con cần 1 trigger đặc biệt: "Execute Workflow Trigger"

```text
[Execute Workflow Trigger] → [Logic xử lý] → [trả output về workflow cha]
```
Output của node cuối trong workflow con sẽ trở thành output của node "Execute Workflow" ở workflow cha.

### Khi nào nên tách sub-workflow

- Logic gửi thông báo (Telegram/Discord/Email) dùng lại ở nhiều workflow khác nhau.
- Logic xác thực/parse dữ liệu chung.
- Muốn giới hạn quyền (workflow A chỉ gọi workflow B qua input/output rõ ràng, không thấy logic nội bộ).
- Giảm độ phức tạp 1 workflow quá dài — chia nhỏ dễ maintain.

### Chạy đồng bộ vs không chờ kết quả

```text
Execute Workflow node → Options → "Wait For Sub-Workflow Completion":
  true (mặc định)  → Chờ workflow con chạy xong mới tiếp tục
  false             → Bắn workflow con chạy, không chờ (fire-and-forget)
```

---

## 4. Variables & Environment

### Environment Variables (biến môi trường hệ thống)

Đặt khi khởi động n8n (qua `.env`, Docker `environment:`...), truy cập bằng `$env` trong expression/Code node (có thể bị admin chặn vì lý do bảo mật — xem mục Self-host).

```bash
# .env
API_BASE_URL=https://api.example.com
WEBHOOK_SECRET=supersecret123
```
```text
{{ $env.API_BASE_URL }}
```

### n8n Variables (Workflow/Environment Variables trong UI — bản Cloud/Enterprise hoặc self-host bật)

Khai báo trong **Settings → Variables** trên UI, không cần restart server, dùng qua `$vars`:
```text
{{ $vars.API_BASE_URL }}
```
> Khác biệt: `$env` đọc từ hệ thống (cần restart để thay đổi), `$vars` đọc từ DB qua UI (đổi ngay, không cần restart) — ưu tiên `$vars` cho giá trị hay thay đổi, `$env` cho secret nhạy cảm cấu hình ở tầng hạ tầng.

---

## 5. Queue Mode — scale production

Mặc định n8n chạy ở **regular mode**: 1 process xử lý cả UI, webhook, và chạy workflow — dễ nghẽn khi tải cao (theo benchmark thực tế: ~23 request/s, dễ timeout khi nhiều workflow chạy đồng thời).

**Queue mode** tách: 1 process **Main** (UI + nhận webhook + đẩy job vào Redis) ra khỏi nhiều process **Worker** (lấy job từ Redis, thực thi thật) — đạt hiệu năng cao hơn nhiều lần và không bị 1 workflow nặng làm nghẽn cả hệ thống.

```text
[Webhook đến] → [Main process] → đẩy job vào → [Redis Queue] → [Worker 1, Worker 2, ...] lấy job ra chạy
```

### Biến môi trường bắt buộc cho Queue Mode

```bash
EXECUTIONS_MODE=queue
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
QUEUE_HEALTH_CHECK_ACTIVE=true       # Kiểm tra sâu kết nối Redis

# PostgreSQL bắt buộc cho queue mode (SQLite không phù hợp multi-process)
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
```

### Docker Compose — Queue Mode đầy đủ

```yaml
# docker-compose.yml
x-n8n-base: &n8n-base
  image: docker.n8n.io/n8nio/n8n
  restart: unless-stopped
  environment:
    - DB_TYPE=postgresdb
    - DB_POSTGRESDB_HOST=postgres
    - DB_POSTGRESDB_DATABASE=n8n
    - DB_POSTGRESDB_USER=n8n
    - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
    - EXECUTIONS_MODE=queue
    - QUEUE_BULL_REDIS_HOST=redis
    - QUEUE_BULL_REDIS_PORT=6379
    - QUEUE_HEALTH_CHECK_ACTIVE=true
    - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    - GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
    - TZ=Asia/Ho_Chi_Minh
  depends_on:
    - postgres
    - redis
  volumes:
    - n8n_data:/home/node/.n8n

services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data

  n8n-main:
    <<: *n8n-base
    ports:
      - "5678:5678"
    environment:
      - <<: *n8n-base-env   # (nếu dùng YAML anchor cho env, tuỳ cấu trúc)
      - N8N_HOST=n8n.example.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.example.com/

  n8n-worker:
    <<: *n8n-base
    command: worker --concurrency=10
    deploy:
      replicas: 2          # Chạy 2 worker container, mỗi worker xử lý 10 job đồng thời

volumes:
  postgres_data:
  redis_data:
  n8n_data:
```

> Lưu ý cú pháp YAML anchor (`x-n8n-base`, `<<: *n8n-base`) có thể cần điều chỉnh tuỳ phiên bản Docker Compose — nếu lỗi, cách an toàn nhất là viết đầy đủ `environment:` riêng cho từng service thay vì dùng anchor.

### Scale số worker

```bash
docker compose up -d --scale n8n-worker=3
```
Hoặc tăng `--concurrency` để 1 worker xử lý nhiều job song song hơn (giới hạn bởi CPU/RAM thật của máy).

### Khi nào cần Queue Mode

| Tình huống | Cần Queue Mode? |
|---|---|
| Dùng cá nhân, vài workflow, ít webhook | Không cần — regular mode đủ |
| < 1.000 execution/ngày | Thường chưa cần — PostgreSQL + regular mode là đủ |
| Nhiều webhook đồng thời, workflow chạy lâu (browser automation, AI Agent...) | Nên dùng |
| Cần tách biệt: webhook đến luôn được nhận, dù 1 workflow đang chạy rất lâu | Bắt buộc |

### Khắc phục lỗi Worker không nhận job

```bash
docker compose logs n8n-worker      # Kiểm tra log worker
```
Nguyên nhân thường gặp: Worker không kết nối được Redis (sai `QUEUE_BULL_REDIS_HOST`/password), hoặc Worker và Main không cùng trỏ tới 1 PostgreSQL database (mỗi instance trong cluster phải dùng chung DB).

---

## 6. Task Runners (n8n 2.0+)

Từ n8n 2.0, **Code node** (JavaScript/Python) chạy trong process cô lập riêng gọi là **Task Runner** — không chạy trực tiếp trong process Main/Worker nữa.

```text
[n8n Main/Worker] → giao code cần chạy → [Task Runner] (container riêng) → trả kết quả về
```

### Vì sao tách riêng

- **An toàn:** code lỗi/độc hại trong Code node không thể làm crash n8n.
- **Cô lập:** Task Runner không truy cập được biến môi trường hay filesystem của n8n.
- **Ổn định:** script tốn quá nhiều RAM bị kill riêng, không ảnh hưởng phần còn lại.

### Cấu hình (External Mode — mỗi process Main/Worker cần 1 sidecar runner riêng)

```bash
N8N_RUNNERS_ENABLED=true
N8N_RUNNERS_MODE=external
N8N_RUNNERS_AUTH_TOKEN=${RUNNERS_AUTH_TOKEN}
```

```yaml
# Thêm vào docker-compose.yml — 1 runner sidecar cho mỗi n8n-main / n8n-worker
n8n-runner:
  image: docker.n8n.io/n8nio/runners      # PHẢI cùng version với image n8n đang dùng
  restart: unless-stopped
  environment:
    - N8N_RUNNERS_AUTH_TOKEN=${RUNNERS_AUTH_TOKEN}
    - N8N_RUNNERS_TASK_BROKER_URI=http://n8n-main:5679
```

> ⚠️ **Version `n8nio/runners` phải khớp chính xác với version `n8nio/n8n`** — lệch version là nguyên nhân phổ biến nhất khiến Task Runner báo "unhealthy".

---

## 7. Self-host Production đầy đủ

### Checklist trước khi đưa lên production

- [ ] Dùng **PostgreSQL**, không dùng SQLite.
- [ ] Đặt `N8N_ENCRYPTION_KEY` cố định, backup giá trị này an toàn.
- [ ] Bật HTTPS (qua reverse proxy — xem mục 8).
- [ ] `WEBHOOK_URL` trỏ đúng domain public (không phải `localhost`).
- [ ] Tắt các tính năng không cần: `N8N_DIAGNOSTICS_ENABLED=false`, `N8N_PERSONALIZATION_ENABLED=false`, `N8N_HIRING_BANNER_ENABLED=false`.
- [ ] Có cơ chế backup định kỳ (DB + thư mục `.n8n`).
- [ ] Đặt giới hạn tài nguyên container (CPU/RAM) tránh 1 workflow lỗi chiếm hết máy.
- [ ] Nếu nhiều người dùng chung, bật User Management + giới hạn quyền Credential theo Project.

### Biến môi trường bảo mật & vận hành quan trọng

```bash
N8N_METRICS=false
N8N_DIAGNOSTICS_ENABLED=false
N8N_PERSONALIZATION_ENABLED=false
N8N_HIRING_BANNER_ENABLED=false

# Tắt truy cập biến môi trường từ trong Code node (an toàn hơn, nhưng mất tiện lợi $env)
N8N_BLOCK_ENV_ACCESS_IN_NODE=true

# Cho phép thêm vài package Node.js ngoài (nếu cần) trong Code node — mặc định bị chặn
NODE_FUNCTION_ALLOW_EXTERNAL=moment,lodash
NODE_FUNCTION_ALLOW_BUILTIN=crypto
```

### Cấu hình PostgreSQL tối ưu (tham khảo, tuỳ RAM máy thật)

```ini
# postgresql.conf — máy 4GB RAM dành cho Postgres
shared_buffers = '1GB'          # ~25% RAM
effective_cache_size = '3GB'    # ~75% RAM
work_mem = '64MB'
maintenance_work_mem = '256MB'
```

### Giới hạn tài nguyên container

```yaml
services:
  n8n-worker:
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 2G
```

---

## 8. Reverse Proxy & SSL

### Caddy (đơn giản nhất — tự xin SSL Let's Encrypt)

```caddyfile
# Caddyfile
n8n.example.com {
    reverse_proxy n8n-main:5678
}
```
```bash
docker run -d -p 80:80 -p 443:443 \
  -v $PWD/Caddyfile:/etc/caddy/Caddyfile \
  -v caddy_data:/data \
  caddy
```

### Nginx + Certbot

```nginx
server {
    listen 80;
    server_name n8n.example.com;
    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Quan trọng cho n8n: hỗ trợ WebSocket (editor real-time)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
```bash
sudo certbot --nginx -d n8n.example.com
```

### Traefik (phù hợp khi đã dùng Traefik cho nhiều service khác trên VPS)

```yaml
services:
  n8n-main:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`n8n.example.com`)"
      - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"
```

### Cloudflare Tunnel (không cần mở port, phù hợp VPS sau NAT/firewall chặt)

```bash
cloudflared tunnel create n8n
cloudflared tunnel route dns n8n n8n.example.com
cloudflared tunnel run n8n
```
```yaml
# config.yml cho cloudflared
ingress:
  - hostname: n8n.example.com
    service: http://localhost:5678
  - service: http_status:404
```

---

## 9. Backup & Migration

### Backup thủ công

```bash
# Backup DB (PostgreSQL)
docker exec -t postgres pg_dump -U n8n n8n > n8n_backup_$(date +%Y%m%d).sql

# Backup thư mục cấu hình (SQLite/encryption key/credentials đã mã hoá)
docker run --rm -v n8n_data:/data -v $PWD:/backup alpine \
  tar czf /backup/n8n_data_$(date +%Y%m%d).tar.gz -C /data .
```

### Export/Import workflow qua CLI (n8n có CLI riêng trong container)

```bash
docker exec -it n8n-main n8n export:workflow --all --output=/home/node/.n8n/backup/workflows.json
docker exec -it n8n-main n8n import:workflow --input=/home/node/.n8n/backup/workflows.json

docker exec -it n8n-main n8n export:credentials --all --decrypted --output=/home/node/.n8n/backup/creds.json
# ⚠️ --decrypted xuất ra dạng ĐỌC ĐƯỢC — chỉ dùng nội bộ, xoá ngay sau khi dùng xong, tuyệt đối không commit lên git.
```

### Restore lên máy mới

1. Cài n8n version **giống hệt** (hoặc mới hơn) bản đã backup.
2. Đặt lại đúng `N8N_ENCRYPTION_KEY` cũ — **bắt buộc**, nếu không sẽ không đọc được credential cũ.
3. Restore DB (PostgreSQL `pg_restore`/`psql`) hoặc copy lại volume SQLite.
4. Restart n8n, kiểm tra `n8n.io` UI load đúng workflow + credential còn hoạt động.

---

*Tài liệu biên soạn tổng hợp — tham khảo thêm tại docs.n8n.io/hosting/ khi cần xác minh chi tiết version mới nhất.*
