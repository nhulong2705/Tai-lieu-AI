# n8n — Phần 1: Cơ bản, Cài đặt, Giao diện, Node cơ bản

> Biên soạn từ kiến thức tổng hợp về n8n (không fetch trực tiếp docs.n8n.io).
> ⚠️ n8n cập nhật khá nhanh — nếu gặp khác biệt nhỏ về UI/tên node, ưu tiên tin vào những gì bạn thấy trong app, coi tài liệu này là khung tham khảo.

---

## Mục lục

1. [n8n là gì](#1-n8n-là-gì)
2. [Cài đặt](#2-cài-đặt)
3. [Giao diện Editor](#3-giao-diện-editor)
4. [Khái niệm cốt lõi](#4-khái-niệm-cốt-lõi)
5. [Trigger Node — điểm khởi đầu workflow](#5-trigger-node--điểm-khởi-đầu-workflow)
6. [Node xử lý dữ liệu cơ bản](#6-node-xử-lý-dữ-liệu-cơ-bản)
7. [Node điều kiện & rẽ nhánh](#7-node-điều-kiện--rẽ-nhánh)
8. [HTTP Request — gọi API bên ngoài](#8-http-request--gọi-api-bên-ngoài)
9. [Chạy thử & Debug](#9-chạy-thử--debug)
10. [Lưu, Active, Version Control](#10-lưu-active-version-control)

---

## 1. n8n là gì

n8n (đọc "n-eight-n", viết tắt "nodemation") là công cụ **workflow automation** mã nguồn mở (fair-code license) — kết nối API, app, database, AI model thành quy trình tự động bằng giao diện kéo-thả node, nhưng vẫn cho viết code (JavaScript/Python) khi cần linh hoạt hơn no-code thuần.

**Điểm khác biệt so với Zapier/Make:**
- **Self-host được** — chạy trên VPS riêng, không phụ thuộc giá theo task của bên thứ 3.
- **Code node** — chèn JavaScript/Python ngay trong workflow khi node có sẵn không đủ.
- **Giá theo execution** (không theo từng "task" nhỏ như Zapier) — rẻ hơn nhiều ở quy mô lớn.
- **400+ integration** có sẵn + có thể gọi bất kỳ API qua HTTP Request node.
- **AI-native** — node AI Agent, LangChain tích hợp sẵn, dùng được với hầu hết LLM provider.

---

## 2. Cài đặt

### Cách 1 — npx (thử nhanh, không cần cài gì)
```bash
npx n8n
```
Mở `http://localhost:5678` trong browser.

### Cách 2 — npm global (local dev)
```bash
npm install n8n -g
n8n start
```

### Cách 3 — Docker (khuyến nghị cho production/VPS)
```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e GENERIC_TIMEZONE="Asia/Ho_Chi_Minh" \
  -e TZ="Asia/Ho_Chi_Minh" \
  docker.n8n.io/n8nio/n8n
```

### Cách 4 — Docker Compose (khuyến nghị nhất cho self-host lâu dài)

```yaml
# docker-compose.yml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
      - TZ=Asia/Ho_Chi_Minh
      - N8N_HOST=n8n.example.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.example.com/
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```
```bash
docker compose up -d
docker compose logs -f n8n
```

> ⚠️ **`N8N_ENCRYPTION_KEY`** mã hoá toàn bộ credential lưu trong DB. Đặt 1 lần và **không bao giờ đổi** sau khi đã lưu credential — đổi key sẽ làm mất quyền giải mã credential cũ. Backup giá trị này cẩn thận.

### Cách 5 — n8n Desktop App
Tải từ trang chủ n8n — chạy local trên máy, giao diện riêng, dữ liệu lưu local. Phù hợp dùng cá nhân không cần luôn online.

### Cách 6 — n8n Cloud (SaaS chính chủ)
Không cần tự quản lý hạ tầng, có free trial, trả phí theo execution/tháng.

### Database — SQLite (mặc định) vs PostgreSQL

| | SQLite (mặc định) | PostgreSQL (khuyến nghị production) |
|---|---|---|
| Cài đặt | Tự động, không cần gì thêm | Cần thêm container/service riêng |
| Phù hợp | Dev, test, ít execution | Production, queue mode, nhiều worker |
| Giới hạn | Lock khi concurrent cao | Scale tốt, hỗ trợ multi-worker |

Cấu hình PostgreSQL:
```yaml
environment:
  - DB_TYPE=postgresdb
  - DB_POSTGRESDB_HOST=postgres
  - DB_POSTGRESDB_PORT=5432
  - DB_POSTGRESDB_DATABASE=n8n
  - DB_POSTGRESDB_USER=n8n
  - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
```

---

## 3. Giao diện Editor

### Các vùng chính

| Vùng | Mô tả |
|---|---|
| **Canvas** | Vùng giữa — kéo-thả node, nối dây giữa node |
| **Node panel** (nhấn `+` hoặc `Tab`) | Tìm & thêm node mới |
| **NDV** (Node Detail View) | Click đúp vào node → mở panel cấu hình chi tiết + xem input/output data |
| **Top bar** | Tên workflow, nút Save, Active toggle, nút Execute Workflow |
| **Left sidebar** | Workflows, Credentials, Executions, Templates |

### Phím tắt hữu ích

| Phím | Hành động |
|---|---|
| `Tab` | Mở node panel tìm kiếm nhanh |
| `Ctrl/Cmd + S` | Lưu workflow |
| `Ctrl/Cmd + Enter` | Chạy node hiện tại (trong NDV) |
| `Ctrl/Cmd + A` | Chọn tất cả node |
| `Ctrl/Cmd + C` / `V` | Copy/Paste node (giữ nguyên kết nối) |
| `Ctrl/Cmd + D` | Duplicate node |
| `Shift + S` | Bật/tắt "Sticky Note" (ghi chú trên canvas) |
| Double-click vào dây nối | Thêm node mới ngay giữa 2 node đã nối |

### NDV (Node Detail View) — 3 cột quan trọng

Khi mở 1 node, NDV chia 3 phần:
1. **INPUT** (trái) — data nhận vào từ node trước
2. **Cấu hình node** (giữa) — các field cần điền
3. **OUTPUT** (phải) — kết quả sau khi chạy thử node này

> Đây là công cụ debug quan trọng nhất của n8n — luôn xem được data thật ở mỗi bước, không phải đoán.

---

## 4. Khái niệm cốt lõi

| Thuật ngữ | Ý nghĩa |
|---|---|
| **Workflow** | Toàn bộ quy trình tự động — tập hợp node nối với nhau |
| **Node** | 1 bước xử lý (nhận data vào → xử lý → trả data ra) |
| **Trigger** | Node đặc biệt khởi động workflow (Webhook, Schedule, Manual...) |
| **Item** | 1 đơn vị data (giống 1 "row"). Node thường xử lý 1 **array of items** |
| **Execution** | 1 lần workflow chạy thực tế (có lịch sử lưu lại) |
| **Connection** | Dây nối giữa 2 node, quyết định luồng data |
| **Credential** | Thông tin xác thực (API key, OAuth...) lưu riêng, mã hoá, tái dùng nhiều workflow |
| **Sub-workflow** | Workflow được gọi từ workflow khác (qua node "Execute Workflow") |
| **Pin Data** | "Đóng băng" data mẫu của 1 node để test các node sau mà không cần chạy lại API thật |

### Mọi node xử lý 1 mảng item, không chỉ 1 object

Đây là điểm quan trọng nhất cần hiểu: nếu node trước trả về 3 item, node sau (mặc định) chạy **lặp qua từng item một** tự động — bạn không cần viết loop thủ công cho hầu hết trường hợp.

---

## 5. Trigger Node — điểm khởi đầu workflow

Mỗi workflow cần ít nhất 1 trigger để biết khi nào chạy.

| Trigger | Khi nào dùng |
|---|---|
| **Manual Trigger** | Test thủ công bằng nút "Execute Workflow" |
| **Webhook** | Nhận HTTP request từ bên ngoài (API, app khác gọi tới) |
| **Schedule Trigger** | Chạy theo lịch (cron expression hoặc interval đơn giản) |
| **Chat Trigger** | Khởi động qua giao diện chat (dùng cho AI Agent workflow) |
| **Email Trigger (IMAP)** | Khởi động khi có email mới |
| **App-specific Trigger** | VD: Telegram Trigger, Gmail Trigger, Slack Trigger... — lắng nghe event riêng của app đó |
| **Error Trigger** | Khởi động khi 1 workflow khác bị lỗi (dùng để báo cáo lỗi) |

### Webhook Trigger — chi tiết

```text
Webhook URL (test):       https://your-n8n.com/webhook-test/abc123
Webhook URL (production): https://your-n8n.com/webhook/abc123
```
- URL **test** chỉ hoạt động khi bạn đang mở editor và bấm "Listen for test event".
- URL **production** chỉ hoạt động khi workflow đã **Active** (toggle ở top bar).
- HTTP Method: GET, POST, PUT, DELETE... chọn được trong node.
- Response Mode: trả response ngay khi nhận (immediately) hoặc chờ tới khi workflow chạy xong ("When Last Node Finishes") hoặc tự custom response bằng node "Respond to Webhook".

```text
# Test webhook bằng curl
curl -X POST https://your-n8n.com/webhook/abc123 \
  -H "Content-Type: application/json" \
  -d '{"name": "test", "value": 123}'
```
> Data từ webhook nằm ở `$json.body`, không phải `$json` trực tiếp.

### Schedule Trigger

```text
Interval: Every minute / Every hour / Every day / Every week / Custom (Cron)
Cron expression: 0 9 * * *   (mỗi ngày 9:00 sáng)
```

---

## 6. Node xử lý dữ liệu cơ bản

### Set (Edit Fields) — tạo/sửa field

Node phổ biến nhất — định hình lại shape của data.

```text
Mode: Manual Mapping | JSON
Fields to Set:
  name        → {{ $json.fullName }}
  email       → {{ $json.contact.email }}
  isActive    → true
```
Tuỳ chọn **"Keep Only Set"**: bật để chỉ giữ field bạn vừa định nghĩa, tắt để giữ cả field cũ + field mới.

### Code — viết JavaScript/Python khi node có sẵn không đủ

```javascript
// Mode "Run Once for All Items"
return items.map(item => {
  item.json.fullName = `${item.json.firstName} ${item.json.lastName}`;
  return item;
});
```
```javascript
// Mode "Run Once for Each Item" — không cần map, chỉ xử lý 1 item
$input.item.json.fullName = `${$input.item.json.firstName} ${$input.item.json.lastName}`;
return $input.item;
```

> Code node **không dùng cú pháp `{{ }}`** — viết JS thuần, truy cập data trực tiếp qua `$json`, `$input`, `items`.

Python cũng được hỗ trợ (chọn "Python" trong dropdown Language) nhưng chạy chậm hơn JS vì qua Pyodide (WASM).

### Merge — gộp data từ 2 nhánh

| Mode | Mô tả |
|---|---|
| Append | Nối tất cả item từ input 1 và input 2 lại |
| Combine (Merge by Key) | Gộp theo field khoá chung (giống SQL JOIN) |
| Combine (Merge by Position) | Gộp item thứ N của input 1 với item thứ N của input 2 |
| Choose Branch | Chỉ giữ output của 1 nhánh (dùng cho fallback) |

### Split Out / Item Lists — tách mảng thành nhiều item

```text
Split Out: lấy 1 field dạng array (vd "orders": [...]) → tách thành nhiều item riêng, mỗi item là 1 phần tử trong array đó.
```

### Aggregate — gộp nhiều item thành 1

Ngược với Split Out — gộp toàn bộ item lại thành 1 item duy nhất chứa array.

### Filter — giữ lại item thoả điều kiện

```text
Condition: {{ $json.status }} equals "active"
```
Item không thoả bị loại khỏi output (khác với IF — chỉ có 1 output, không rẽ nhánh).

### Remove Duplicates

So sánh item theo field chỉ định, loại bỏ trùng lặp.

### Sort

Sắp xếp item theo 1 hoặc nhiều field, tăng/giảm dần.

### Limit

Chỉ giữ N item đầu hoặc N item cuối.

---

## 7. Node điều kiện & rẽ nhánh

### IF — rẽ 2 nhánh (true/false)

```text
Condition: {{ $json.amount }} > 1000
→ Output "true": item thoả điều kiện
→ Output "false": item không thoả
```

### Switch — rẽ nhiều nhánh (giống switch-case)

```text
Mode: Rules
Rule 1: {{ $json.category }} equals "urgent"   → Output 0
Rule 2: {{ $json.category }} equals "normal"   → Output 1
Fallback Output: Output 2 (mọi trường hợp khác)
```

### Compare Datasets

So sánh 2 dataset (input A vs input B), trả ra: item chỉ có ở A, chỉ có ở B, có ở cả 2 nhưng khác nhau, giống nhau hoàn toàn. Dùng cho đồng bộ dữ liệu (sync 2 nguồn).

### Loop Over Items (Split in Batches)

Chia item thành các batch nhỏ, xử lý từng batch — dùng khi cần rate-limit khi gọi API (tránh gọi 1000 request đồng thời), hoặc khi cần logic "xử lý xong batch này mới qua batch sau".

```text
Batch Size: 10
→ Output 1 "loop": batch hiện tại, nối ngược lại đầu loop
→ Output 2 "done": khi hết item, đi tiếp ra ngoài loop
```

---

## 8. HTTP Request — gọi API bên ngoài

Node quan trọng nhất khi tích hợp app không có node riêng (hoặc gọi API tuỳ chỉnh, ví dụ gọi tới Hermes/OpenClaw qua webhook).

```text
Method: GET | POST | PUT | PATCH | DELETE
URL: https://api.example.com/v1/users/{{ $json.userId }}
Authentication: None | Predefined Credential Type | Generic Credential Type
Headers:
  Content-Type: application/json
  Authorization: Bearer {{ $credentials.apiKey }}
Body Content Type: JSON | Form-Data | Form-Urlencoded | Raw | Binary
Body:
  {
    "name": "{{ $json.name }}",
    "email": "{{ $json.email }}"
  }
```

### Tuỳ chọn quan trọng (mục "Options")

| Tuỳ chọn | Dùng khi |
|---|---|
| **Response Format** | JSON / Text / File — chọn đúng kiểu response trả về |
| **Timeout** | API chậm, cần tăng thời gian chờ |
| **Retry on Fail** | API hay rớt mạng, tự thử lại N lần |
| **Batching** | Gọi nhiều request, giới hạn số lượng đồng thời + delay giữa các lần gọi |
| **Ignore SSL Issues** | Test với endpoint dùng self-signed cert (KHÔNG dùng cho production) |
| **Split Into Items** | Tự tách response array thành nhiều item |

### Gọi webhook Hermes/OpenClaw từ n8n (ví dụ thực tế)

```text
Method: POST
URL: http://openclaw:8000/api/task
Headers:
  Content-Type: application/json
  X-HMAC-Signature: {{ $json.signature }}
Body:
  {
    "task": "{{ $json.message }}",
    "user_id": "{{ $json.user_id }}"
  }
```

---

## 9. Chạy thử & Debug

### Chạy thử 1 node

Click vào node → nút **"Test step"** (hoặc `Ctrl/Cmd+Enter`) — chạy riêng node đó, không chạy toàn workflow.

### Chạy thử cả workflow

Nút **"Execute Workflow"** ở top bar (hoặc bấm vào trigger và "Listen for test event" nếu là Webhook).

### Pin Data — đóng băng kết quả để test nhanh

Click chuột phải vào node đã chạy → **"Pin data"** — n8n dùng data đã pin thay vì chạy lại API thật mỗi lần test các node phía sau. Bỏ pin khi deploy thật.

### Xem lịch sử Execution

Sidebar → **Executions** — xem mọi lần chạy (thành công/lỗi), click vào để xem chi tiết data ở từng node tại thời điểm đó. Cực hữu ích khi debug lỗi production.

### Error Workflow — bắt lỗi tự động

```text
Workflow Settings → Error Workflow: chọn 1 workflow khác (có Error Trigger)
→ Khi workflow chính lỗi, workflow lỗi sẽ tự chạy, nhận thông tin lỗi qua Error Trigger,
   có thể gửi alert Telegram/Slack/Email.
```

### Try/Catch trong workflow (Node-level Error Handling)

Click chuột phải vào 1 node → **"On Error" → "Continue (using error output)"** — node lỗi sẽ đẩy ra 1 output riêng (`error`) thay vì làm fail toàn workflow, cho phép xử lý lỗi linh hoạt ngay trong workflow.

---

## 10. Lưu, Active, Version Control

### Save vs Active

| | Save | Active (toggle) |
|---|---|---|
| Tác dụng | Lưu thay đổi vào DB | Bật webhook/schedule chạy thật ngoài production |
| Khi nào cần | Mọi lần sửa | Chỉ khi muốn workflow chạy tự động không cần mở editor |

> Workflow inactive vẫn chạy được bằng "Execute Workflow" thủ công, nhưng **không** nhận webhook production hoặc chạy theo Schedule.

### Versioning (n8n bản mới có sẵn)

n8n lưu lịch sử các phiên bản đã save của workflow — có thể xem diff và khôi phục về bản cũ (tuỳ tier/cấu hình self-host).

### Export / Import workflow (JSON)

```text
Workflow menu (...) → Download → lưu file .json
Workflow menu (...) → Import from File → chọn file .json
```
Dùng để backup, chia sẻ, hoặc đưa vào git để version control thủ công.

---

*Tài liệu biên soạn tổng hợp — tham khảo thêm tại docs.n8n.io khi cần xác minh chi tiết UI mới nhất.*
