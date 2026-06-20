# 🚀 Hướng dẫn Setup — Fanpage Automation Full System

> File `n8n-fanpage-full-system.json` chứa **46 nodes / 3 workflow** gộp trong 1 file duy nhất.

---

## Tổng quan hệ thống

```
┌─────────────────────────────────────────────────────┐
│  WORKFLOW 1 — Content Generation & Publishing        │
│  ⏰ Schedule → ⚙️ Config → 📋 Brand Sheets           │
│  → 🧠 AI Analyze → ✍️ AI Write → ✅ Quality Gate     │
│  → 🤖 Auto-approve / 💬 Telegram Duyệt              │
│  → 📘 Facebook / 📸 Instagram → 📊 Log              │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  WORKFLOW 2 — Telegram Approval Bot                  │
│  📡 Webhook → 🔍 Parse → 🔀 Route                   │
│  → APPROVE → Publish → ✅ Confirm                    │
│  → REJECT  → Update status → ❌ Confirm             │
│  → REGENERATE → 🤖 AI viết lại → 💬 Preview mới    │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  WORKFLOW 3 — Analytics & Learning Loop              │
│  ⏰ 22h mỗi ngày → 📈 Fetch metrics                 │
│  → 🤖 AI Analyst → 📨 Telegram report               │
│  → 🔁 Learning Loop → 📝 Update Brand Config        │
└─────────────────────────────────────────────────────┘
```

---

## BƯỚC 1 — Tạo Google Sheets

Tạo 1 file Google Sheets với **4 tab** sau:

---

### Tab 1: `Brand Config`

| Cột | Bắt buộc | Mô tả | Ví dụ |
|---|:---:|---|---|
| `brand_id` | ✅ | ID duy nhất, không đổi | `brand_001` |
| `brand_name` | ✅ | Tên fanpage | `Fanpage Công Nghệ ABC` |
| `brand_voice` | ✅ | Giọng văn | `friendly` / `professional` / `humorous` |
| `target_audience` | ✅ | Mô tả đối tượng | `25-35 tuổi, quan tâm AI và startup` |
| `core_topics` | ✅ | Chủ đề chính | `AI, công nghệ, khởi nghiệp` |
| `platforms` | ✅ | Kênh đăng (cách nhau dấu phẩy) | `fb,ig` |
| `post_times` | ✅ | Giờ đăng theo 24h | `8,12,18` |
| `page_id` | ✅ | Facebook Page ID | `123456789012345` |
| `ig_account_id` | ⬜ | Instagram Business Account ID | `987654321` |
| `rss_urls` | ⬜ | URL RSS nguồn tin | `https://techcrunch.com/feed/` |
| `telegram_chat_id` | ✅ | ID chat nhận thông báo | `-100123456789` |
| `auto_approve_threshold` | ⬜ | Điểm tự duyệt (để trống = dùng default 80) | `85` |
| `quality_gate_min_score` | ⬜ | Điểm tối thiểu (để trống = dùng default 60) | `65` |
| `content_language` | ⬜ | Ngôn ngữ (để trống = `vi`) | `vi` |
| `ai_model` | ⬜ | AI model (để trống = Gemini Flash) | `gemini-1.5-flash` |
| `suggested_topics` | ⬜ | Tự động cập nhật bởi Analytics | *(để trống)* |
| `optimal_post_times` | ⬜ | Tự động cập nhật bởi Analytics | *(để trống)* |
| `last_avg_engagement` | ⬜ | Tự động cập nhật bởi Analytics | *(để trống)* |
| `active` | ✅ | Bật/tắt brand | `true` / `false` |

**Ví dụ hàng mẫu:**
```
brand_001 | Fanpage ABC | friendly | 25-35 tuổi | AI,công nghệ | fb,ig | 8,12,18 | 123456 | 987654 | https://... | -100xxx | 80 | 60 | vi | | | | | true
```

---

### Tab 2: `Pending Queue`

Để trống — Workflow 1 tự ghi khi có bài chờ duyệt.

Cột sẽ được tạo tự động:
`brand_id` | `brand_name` | `triggered_at` | `platforms` | `page_id` | `captions_json` | `quality_score` | `status` | `post_id` | `published_at` | `approved_by` | `telegram_chat_id`

---

### Tab 3: `Publish Log`

Để trống — Workflow 1 tự ghi sau mỗi lần đăng thành công.

Cột tự động: `timestamp` | `brand_id` | `brand_name` | `platform` | `quality_score` | `post_id` | `caption_preview` | `status` | `approved_by`

---

### Tab 4: `Analytics Log`

Để trống — Workflow 3 tự ghi mỗi tối.

Cột tự động: `date` | `brand_id` | `brand_name` | `post_count` | `total_reach` | `total_engagement` | `avg_engagement_rate` | `best_post_preview` | `topics_tomorrow` | `optimal_times` | `full_report_json`

---

## BƯỚC 2 — Khai báo n8n Variables (Tầng 1)

Vào **Settings → Variables** trong n8n, thêm các biến sau:

| Variable | Mô tả | Lấy ở đâu |
|---|---|---|
| `FB_PAGE_ACCESS_TOKEN` | Facebook Long-lived Page Token | [Meta for Developers](https://developers.facebook.com) |
| `GEMINI_API_KEY` | Google Gemini API Key (miễn phí) | [aistudio.google.com](https://aistudio.google.com) |
| `DEFAULT_AI_MODEL` | Model mặc định | `gemini-1.5-flash` |

> **Lưu ý:** Telegram Bot Token khai báo trong n8n Credentials (không phải Variables) vì dùng node Telegram chính thức.

---

## BƯỚC 3 — Khai báo n8n Credentials

Vào **Settings → Credentials**, tạo:

| Credential | Loại | Dùng bởi |
|---|---|---|
| `Google Sheets OAuth2` | OAuth2 | Tất cả node đọc/ghi Sheets |
| `Telegram Bot` | Telegram API | Tất cả node Telegram |
| `Google Gemini` | API Key | Tất cả AI Agent nodes |

---

## BƯỚC 4 — Import Workflow

1. Mở n8n → **Workflows → Import from File**
2. Chọn file `n8n-fanpage-full-system.json`
3. n8n sẽ hiển thị **3 khu vực** riêng biệt trên canvas (3 workflow trong 1 file)

---

## BƯỚC 5 — Cập nhật Google Sheets ID

Tìm và thay `YOUR_GOOGLE_SHEET_ID` trong các node sau (dùng Ctrl+F trên canvas):

| Node | Workflow |
|---|---|
| `📋 Fetch Brand Config (Sheets)` | W1 |
| `📊 Log to Sheets` | W1 |
| `📋 Fetch Pending Content (Approve/Reject/Regen)` | W2 |
| `✅ Update Status: Published` | W2 |
| `❌ Update Status: Rejected` | W2 |
| `🔄 Update Status: Regenerate` | W2 |
| `📋 Fetch Active Brands` | W3 |
| `📊 Fetch Today's Posts` | W3 |
| `📊 Log Analytics to Sheets` | W3 |
| `📝 Update Brand Strategy` | W3 |

> **Tip nhanh:** Export file JSON → Ctrl+H thay `YOUR_GOOGLE_SHEET_ID` bằng ID thật → Import lại. Xong trong 30 giây.

Lấy Sheet ID từ URL:
```
https://docs.google.com/spreadsheets/d/[ĐÂY_LÀ_SHEET_ID]/edit
```

---

## BƯỚC 6 — Setup Telegram Bot

1. Nhắn `/newbot` với [@BotFather](https://t.me/BotFather) → lấy Bot Token
2. Thêm bot vào group/channel fanpage của bạn
3. Lấy Chat ID: Nhắn tin trong group → truy cập `https://api.telegram.org/bot<TOKEN>/getUpdates`
4. Điền `telegram_chat_id` vào Sheets (Brand Config)
5. Set webhook cho Workflow 2:
```
https://api.telegram.org/bot<TOKEN>/setWebhook?url=<N8N_WEBHOOK_URL>
```
> N8N_WEBHOOK_URL lấy từ node `📡 Telegram Webhook` → Copy Webhook URL

---

## BƯỚC 7 — Test từng workflow

### Test Workflow 1:
```
1. Thêm 1 brand vào Sheets với active=true
2. Chạy thủ công node "⏰ Schedule Trigger" (Execute Node)
3. Kiểm tra từng bước: Sheets đọc được? → AI chạy được? → Telegram nhận preview?
```

### Test Workflow 2:
```
1. Nhấn nút ✅ trên tin nhắn preview Telegram
2. Kiểm tra: Sheets có cập nhật status=published? → Facebook có bài mới?
```

### Test Workflow 3:
```
1. Đảm bảo có ít nhất 1 bài trong Publish Log
2. Chạy thủ công "⏰ Daily Analytics"
3. Kiểm tra: Analytics Log có ghi? → Telegram nhận báo cáo?
```

---

## Mở rộng về sau

### ➕ Thêm fanpage mới
Chỉ cần thêm 1 hàng vào **Sheets tab Brand Config**. Không đụng vào workflow.

### ➕ Thêm platform mới (TikTok, Threads...)
1. Thêm case trong node `🔀 Route Platform` (Workflow 1)
2. Thêm node Publish tương ứng
3. Cập nhật cột `platforms` trong Sheets

### ➕ Thêm AI model mới (Groq, Ollama...)
1. Đổi node AI Agent sang loại model mới
2. Cập nhật cột `ai_model` trong Sheets per brand

### 🔧 Override config per brand
Điền vào các cột optional trong Sheets:
- `auto_approve_threshold` — brand nào cần kiểm soát chặt hơn thì tăng lên 90
- `quality_gate_min_score` — brand mới có thể để 50 để dễ qua hơn lúc đầu
- `post_times` — mỗi brand 1 lịch đăng riêng

---

## Cấu trúc Config 3 tầng (tóm tắt)

```
Tầng 1 — n8n Variables
  FB_PAGE_ACCESS_TOKEN, GEMINI_API_KEY
  → Không bao giờ thay đổi, bảo mật cao

Tầng 2 — Workflow Defaults (node ⚙️)  
  auto_approve_threshold: 80
  retry_attempts: 3
  quality_gate_min_score: 60
  → Thay đổi khi muốn đổi hành vi toàn hệ thống

Tầng 3 — Google Sheets (Brand Config)
  Mỗi hàng = 1 brand, có thể override Tầng 2
  → Thay đổi không cần mở n8n
  → Analytics tự cập nhật cột này mỗi tối
```

---

## Troubleshooting thường gặp

| Lỗi | Nguyên nhân | Cách fix |
|---|---|---|
| Workflow không chạy đúng giờ | `post_times` trong Sheets không khớp giờ hiện tại | Kiểm tra timezone n8n (Settings → Timezone) |
| AI trả về lỗi | Hết quota Gemini free | Dùng Groq API làm fallback |
| Facebook API lỗi 190 | Access token hết hạn | Refresh Long-lived Token (hạn 60 ngày) |
| Telegram không nhận callback | Webhook chưa set | Chạy lại lệnh setWebhook |
| Sheets không ghi được | Google OAuth hết hạn | Reconnect credential trong n8n |

---

*Tạo bởi n8n Expert — Phiên bản 1.0*
