# n8n — Phần 2: Expression, Data & Logic

> Biên soạn từ kiến thức tổng hợp về n8n (không fetch trực tiếp docs.n8n.io).
> Phạm vi: Cú pháp Expression, các biến built-in ($json/$node/$input...), Code node JS/Python, Luxon datetime, lỗi thường gặp.

---

## Mục lục

1. [Cú pháp Expression cơ bản](#1-cú-pháp-expression-cơ-bản)
2. [Các biến built-in quan trọng](#2-các-biến-built-in-quan-trọng)
3. [Code Node — JavaScript chi tiết](#3-code-node--javascript-chi-tiết)
4. [Code Node — Python](#4-code-node--python)
5. [Luxon — xử lý ngày giờ](#5-luxon--xử-lý-ngày-giờ)
6. [JMESPath — query JSON phức tạp](#6-jmespath--query-json-phức-tạp)
7. [Lỗi thường gặp & cách debug](#7-lỗi-thường-gặp--cách-debug)
8. [Workflow Static Data — lưu trạng thái giữa các lần chạy](#8-workflow-static-data--lưu-trạng-thái-giữa-các-lần-chạy)

---

## 1. Cú pháp Expression cơ bản

Mọi field trong node (trừ Code node) dùng cú pháp 2 dấu ngoặc nhọn:

```text
{{ JavaScript expression }}
```

```text
{{ "Hello World" }}
{{ 2 + 3 }}
{{ $json.email }}
{{ $json.status === "active" ? "Yes" : "No" }}
{{ $json.name.toUpperCase() }}
{{ $json.price * 1.1 }}
```

### Truy cập field lồng nhau (nested) và field có khoảng trắng

```text
{{ $json.contact.profile.company.name }}     # Dot notation — field thường
{{ $json['field name with spaces'] }}         # Bracket notation — field có khoảng trắng/ký tự đặc biệt
{{ $json.body.email }}                        # Data từ Webhook luôn nằm trong .body
```

### Biểu thức nhiều dòng (IIFE pattern)

Bình thường expression chỉ là 1 dòng. Muốn gán biến/chạy logic nhiều bước trong 1 ô field (không mở Code node), bọc bằng Immediately Invoked Function Expression:

```text
{{ (function() {
  const a = $json.price;
  const b = $json.quantity;
  return a * b;
})() }}
```
> Cách này dùng được nhưng nếu logic phức tạp hơn vài dòng, nên chuyển sang **Code node** — dễ đọc và debug hơn nhiều.

### Xem trước kết quả expression ngay trong UI

Mỗi field hỗ trợ expression có icon `fx` — bấm vào sẽ mở **Expression Editor**, hiển thị kết quả thực tế resolve từ data của lần chạy gần nhất (hoặc pinned data). Luôn kiểm tra ở đây trước khi chạy thật.

---

## 2. Các biến built-in quan trọng

| Biến | Ý nghĩa | Ví dụ |
|---|---|---|
| `$json` | Data JSON của item hiện tại (node đang xử lý) | `{{ $json.email }}` |
| `$input.item` | Item hiện tại đang xử lý (object đầy đủ, có `.json`) | `{{ $input.item.json.name }}` |
| `$input.all()` | Toàn bộ item input — **chỉ dùng trong Code node**, trả về array | `$input.all()` |
| `$input.first()` | Item đầu tiên trong input | `$input.first().json.id` |
| `$node["Tên Node"]` | Truy cập output của 1 node bất kỳ theo tên (cần đặt trong dấu nháy) | `{{ $node["Webhook"].json.body }}` |
| `$("Tên Node")` | Cách viết mới hơn, tương đương `$node[...]` nhưng có thêm method | `{{ $("HTTP Request").item.json.id }}` |
| `$now` | Thời điểm hiện tại — đối tượng Luxon DateTime | `{{ $now.toISO() }}` |
| `$today` | Ngày hiện tại, giờ = 00:00:00 | `{{ $today.toISODate() }}` |
| `$workflow` | Thông tin workflow hiện tại (id, name, active) | `{{ $workflow.id }}` |
| `$execution` | Thông tin lần chạy hiện tại (id, mode) | `{{ $execution.id }}` |
| `$vars` | Workflow/Environment Variables đã định nghĩa trước | `{{ $vars.API_BASE_URL }}` |
| `$env` | Biến môi trường hệ thống (cần bật, có thể bị chặn vì lý do bảo mật) | `{{ $env.NODE_ENV }}` |
| `$itemIndex` | Vị trí (index) của item hiện tại trong mảng | `{{ $itemIndex }}` |
| `$runIndex` | Lần chạy thứ mấy của node (quan trọng trong loop) | — |
| `$jmespath()` | Query JSON phức tạp bằng cú pháp JMESPath | `{{ $jmespath($json, "people[?age>30].name") }}` |
| `$parameter` | Đọc giá trị 1 parameter khác của cùng node | — |

> ⚠️ **Quan trọng:** `$json` và `$input.item` chỉ dùng được trong field thường (`{{ }}`) hoặc Code node mode "Run Once for Each Item". Trong Code node mode "Run Once for All Items", phải dùng `$input.all()` và tự `.map()`/`.forEach()`.

### `$node[...]` vs item hiện tại — phân biệt quan trọng

```text
{{ $json.email }}                         → Field "email" của item ĐANG xử lý ở node hiện tại
{{ $node["Webhook"].json.email }}          → Field "email" từ output gốc của node "Webhook" (bỏ qua mọi xử lý ở giữa)
```
Dùng `$node[...]` khi cần lấy lại data gốc từ 1 node nào đó phía trước, mà không phải truyền tay qua từng node ở giữa bằng Set node.

### `.item` vs `.all()` vs `.first()` trên 1 reference node — phân biệt quan trọng

```text
$("HTTP Request").item        → Item tương ứng (matching item) với item hiện tại đang xử lý
$("HTTP Request").all()       → Toàn bộ item mà node đó đã output
$("HTTP Request").first()     → Item đầu tiên node đó output
```

---

## 3. Code Node — JavaScript chi tiết

### 2 chế độ chạy (Execute Once)

| Chế độ | Khi nào dùng | Biến chính |
|---|---|---|
| **Run Once for All Items** (mặc định) | Cần xử lý toàn bộ dataset 1 lần — tổng hợp, sắp xếp, lọc, gộp | `$input.all()`, `items` |
| **Run Once for Each Item** | Logic chỉ liên quan tới 1 item, không cần biết item khác | `$json`, `$input.item` |

### Template cơ bản — "Run Once for All Items"

```javascript
const items = $input.all();

const processed = items.map(item => ({
  json: {
    ...item.json,
    processed: true,
    timestamp: new Date().toISOString(),
  }
}));

return processed;
```

> ⚠️ **Luôn return 1 array of objects, mỗi object có key `json`.** Đây là format bắt buộc của n8n — quên bọc trong `{ json: ... }` là lỗi phổ biến nhất.

### Template cơ bản — "Run Once for Each Item"

```javascript
const fullName = `${$json.firstName} ${$json.lastName}`;
return {
  json: {
    ...$json,
    fullName
  }
};
```

### Lọc item (thay cho Filter node khi logic phức tạp)

```javascript
const items = $input.all();

return items.filter(item => {
  const isActive = item.json.status === 'active';
  const hasEmail = item.json.email && item.json.email.includes('@');
  const recentLogin = new Date(item.json.lastLogin) > new Date('2026-01-01');
  return isActive && hasEmail && recentLogin;
});
```

### Tính tổng/trung bình trên toàn bộ item

```javascript
const allItems = $input.all();
const total = allItems.reduce((sum, item) => sum + (item.json.amount || 0), 0);

return [{
  json: {
    total,
    count: allItems.length,
    average: total / allItems.length
  }
}];
```

### Built-in có sẵn trong Code node (không cần import)

| Built-in | Mô tả |
|---|---|
| `DateTime` | Class Luxon — xử lý ngày giờ |
| `$jmespath()` | Query JSON |
| `$helpers.httpRequest()` | Gọi HTTP request không cần credential (giới hạn) |
| `$helpers.httpRequestWithAuthentication()` | Gọi HTTP kèm credential đã lưu (nếu được phép) |
| `console.log()` | In ra log để debug (xem ở tab Console của NDV) |

> `require()` thường **bị chặn** trừ khi admin self-host đã allowlist package cụ thể qua biến môi trường `NODE_FUNCTION_ALLOW_EXTERNAL`. `$env` cũng có thể bị chặn nếu đặt `N8N_BLOCK_ENV_ACCESS_IN_NODE=true`.

### ⚠️ Pitfall thực tế: Loop Over Items (Split in Batches)

Node này có **2 output, thứ tự dễ gây nhầm**:

```text
Output 0 ("done")  → Chỉ fire DUY NHẤT 1 LẦN, sau khi xử lý hết mọi batch
Output 1 ("loop")  → Fire MỖI LẦN cho từng batch — đây là "thân" của vòng lặp, nối ngược lại
```

Nếu cần dùng dữ liệu đã xử lý của **toàn bộ** các batch (không chỉ batch cuối):
```javascript
// $("Node Inside Loop").all() chỉ trả item của LẦN CHẠY CUỐI, không phải cộng dồn!
// → Phải tự cộng dồn bằng Workflow Static Data (xem mục 8) hoặc 1 node tổng hợp riêng ở cuối.
```
Mẹo an toàn: luôn thêm 1 node **Limit (giữ 1 item)** ngay sau output "done" để chặn các trường hợp edge-case fire kèm dữ liệu thừa.

---

## 4. Code Node — Python

Chọn **Language: Python** trong dropdown ở Code node. Cú pháp tương tự nhưng dùng snake_case thay vì camelCase cho built-in:

```python
items = _input.all()

processed = []
for item in items:
    item['json']['full_name'] = f"{item['json']['first_name']} {item['json']['last_name']}"
    processed.append(item)

return processed
```

> ⚠️ Python trong n8n chạy qua **Pyodide** (Python compile sang WebAssembly) — chậm hơn JS đáng kể, và **không cài được pip package ngoài** danh sách built-in sẵn có (numpy, pandas cơ bản có hỗ trợ nhưng hạn chế). Nếu cần xử lý nặng, ưu tiên gọi ra 1 service Python riêng qua HTTP Request thay vì viết trực tiếp trong Code node.

---

## 5. Luxon — xử lý ngày giờ

n8n dùng thư viện **Luxon** cho mọi thứ liên quan ngày giờ — không dùng `Date` thuần của JavaScript trong expression (`$now`, `$today` đã là đối tượng Luxon `DateTime`).

```text
{{ $now }}                          → Thời điểm hiện tại
{{ $now.toISO() }}                  → "2026-06-19T14:30:00.000+07:00"
{{ $now.toISODate() }}              → "2026-06-19"
{{ $now.toISOTime() }}              → "14:30:00.000+07:00"
{{ $now.toSeconds() }}              → Unix timestamp (giây)
{{ $now.toFormat("MMMM d, yyyy") }} → "June 19, 2026"
{{ $now.plus({ days: 7 }) }}        → +7 ngày từ hiện tại
{{ $now.minus({ days: 7 }).toISODate() }}   → Ngày của 7 ngày trước
{{ $now.diff($json.createdAt, "days").days }}  → Số ngày chênh lệch
{{ DateTime.fromISO($json.dateString) }}    # Trong Code node — parse string thành Luxon DateTime
```

### Parse 1 string ngày giờ tuỳ ý

```javascript
const dt = DateTime.fromFormat($json.dateString, "dd/MM/yyyy");
return { json: { isoDate: dt.toISODate() } };
```

---

## 6. JMESPath — query JSON phức tạp

Khi cần lọc/trích xuất sâu từ JSON lồng nhau mà dot-notation không đủ mạnh:

```text
{{ $jmespath($json.people, "[?age > `30`].name") }}
   → Lấy tên (name) của mọi người trong array "people" có age > 30
```

> JMESPath hữu ích khi response API trả về cấu trúc phức tạp (nhiều cấp lồng, cần filter/projection) — nhưng với logic phức tạp hơn vài dòng, Code node thường dễ đọc và dễ maintain hơn JMESPath.

---

## 7. Lỗi thường gặp & cách debug

| Lỗi/triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| Expression trả `undefined` | Field không tồn tại ở item đó, hoặc sai tên node | Mở Expression Editor xem preview với data thật |
| `$node["X"]` lỗi "node not found" | Tên node viết sai/có khoảng trắng thừa, hoặc node X chưa từng chạy | Kiểm tra đúng tên hiển thị trên canvas (case-sensitive) |
| Code node lỗi "must return array of objects with json key" | Quên bọc `{ json: ... }` khi return | Luôn `return [{ json: {...} }]` hoặc `return items.map(i => ({json: ...}))` |
| Expression đúng trong Editor nhưng sai khi chạy thật | Editor dùng pinned/data cũ, không phải data live | Bỏ Pin Data, chạy lại bằng dữ liệu thật |
| `$input.all()` chỉ trả về 1 item dù input có nhiều | Bug hiếm gặp khi node trước trả về cấu trúc lạ, hoặc do thiếu 1 node "buffer" giữa 2 node | Thêm 1 IF/NoOp node trung gian (luôn true) giữa node nguồn và Code node để "ép" n8n gom đủ item |
| Webhook nhận data nhưng field rỗng | Quên rằng webhook body nằm ở `$json.body`, không phải `$json` trực tiếp | Dùng `{{ $json.body.fieldName }}` |
| Response Header không truy cập được | Quên bật "Include Response Headers" trong HTTP Request node | Bật option đó → header có ở `$json.headers['content-type']` (luôn viết thường) |
| SplitInBatches mất dữ liệu của các batch trước | Hiểu sai rằng `.all()` trả cộng dồn — thực ra chỉ trả batch cuối | Dùng Workflow Static Data để tự cộng dồn (xem mục 8) |

### Debug bằng `console.log()` trong Code node

```javascript
console.log("Debug:", JSON.stringify($json, null, 2));
return $input.all();
```
Xem output ở tab dưới panel Code node (hoặc trong log file nếu chạy server-side).

---

## 8. Workflow Static Data — lưu trạng thái giữa các lần chạy

Dùng khi cần "nhớ" giá trị xuyên qua các vòng loop hoặc giữa các lần execution (ví dụ: cộng dồn kết quả qua từng batch của SplitInBatches, hoặc lưu lại "lần chạy gần nhất" để so sánh).

```javascript
// Trong Code node
const staticData = $getWorkflowStaticData('node');   // hoặc 'global' để share toàn workflow

if (!staticData.accumulated) {
  staticData.accumulated = [];
}
staticData.accumulated.push(...$input.all().map(i => i.json));

return [{ json: { count: staticData.accumulated.length } }];
```

> Static data được lưu cùng workflow trong database — tồn tại qua các lần Execute Workflow khác nhau (không bị reset mỗi lần chạy), nhưng **nên dùng cẩn thận** vì dễ gây trạng thái "ẩn" khó debug nếu không quản lý rõ khi nào reset.

---

*Tài liệu biên soạn tổng hợp — tham khảo thêm tại docs.n8n.io/code/builtin/ khi cần xác minh chi tiết.*
