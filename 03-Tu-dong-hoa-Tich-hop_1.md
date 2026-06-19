# Hermes Agent — Phần 3: Tự động hoá & Tích hợp

> Biên soạn từ docs chính thức: hermes-agent.nousresearch.com/docs
> Phạm vi: Cron (lập lịch), Subagent Delegation, Messaging Gateway tổng quan, Telegram setup chi tiết.

---

## Mục lục

1. [Cron — Tác vụ lập lịch](#1-cron--tác-vụ-lập-lịch)
2. [Subagent Delegation — Giao việc song song](#2-subagent-delegation--giao-việc-song-song)
3. [Messaging Gateway — Tổng quan](#3-messaging-gateway--tổng-quan)
4. [Telegram — Cài đặt chi tiết](#4-telegram--cài-đặt-chi-tiết)

---

## 1. Cron — Tác vụ lập lịch

Hermes quản lý cron qua một tool duy nhất `cronjob` (action-style), có thể: lập lịch 1 lần hoặc lặp lại, tạm dừng/tiếp tục/sửa/chạy ngay/xoá job, gắn 1 hoặc nhiều skill, gửi kết quả về chat gốc/file/platform bất kỳ, và chạy ở **chế độ không-agent** (script thuần, 0 token).

> Job cron dùng provider đã chọn ở `hermes model`. `hermes setup --portal` là lựa chọn ít rắc rối nhất cho job chạy không người trông vì OAuth tự refresh.

### Tạo job qua chat

```text
/cron add 30m "Remind me to check the build"
/cron add "every 2h" "Check server status"
/cron add "every 1h" "Summarize new feed items" --skill blogwatcher
```

### Tạo job qua CLI độc lập

```bash
hermes cron create "every 2h" "Check server status"
hermes cron create "every 1h" "Summarize new feed items" --skill blogwatcher
hermes cron create "every 1h" "Use both skills and combine the result" \
  --skill blogwatcher \
  --skill maps \
  --name "Skill combo"
```

### Tạo qua hội thoại tự nhiên

```text
Every morning at 9am, check Hacker News for AI news and send me a summary on Telegram.
```

### Chạy job trong 1 thư mục project cụ thể

```bash
hermes cron create "every 1d at 09:00" \
  "Audit open PRs, summarize CI health, and post to #eng" \
  --workdir /home/me/projects/acme
```
Khi đặt `workdir`: AGENTS.md/CLAUDE.md/.cursorrules trong thư mục đó được nạp, và mọi tool file/terminal dùng thư mục này làm cwd.

### Sửa & quản lý lifecycle

```bash
/cron edit <job_id> --schedule "every 4h"
/cron edit <job_id> --prompt "Use the revised task"
/cron edit <job_id> --skill blogwatcher --skill maps
/cron edit <job_id> --remove-skill blogwatcher
/cron edit <job_id> --clear-skills

/cron list
/cron pause <job_id>
/cron resume <job_id>
/cron run <job_id>
/cron remove <job_id>
```

CLI độc lập tương đương:
```bash
hermes cron list
hermes cron pause <job_id_or_name>
hermes cron resume <job_id_or_name>
hermes cron run <job_id_or_name>
hermes cron remove <job_id_or_name>
hermes cron edit <job_id_or_name> [...flags]
hermes cron status
hermes cron tick
```
> Có thể dùng **tên job** thay cho ID hex (không phân biệt hoa thường). Nếu nhiều job trùng tên, lệnh sẽ từ chối và in ra danh sách ID để bạn chọn rõ.

### Cách hoạt động

Gateway daemon tick scheduler mỗi **60 giây**:
```bash
hermes gateway install     # Cài như user service
sudo hermes gateway install --system   # Linux: service hệ thống chạy từ lúc boot
hermes gateway             # Hoặc chạy foreground
```
Mỗi tick: đọc job từ `~/.hermes/cron/jobs.json` → kiểm tra `next_run_at` → mở session `AIAgent` mới cho job đến hạn → nạp skill (nếu có) → chạy prompt → gửi kết quả → cập nhật lịch chạy tiếp.

> ⚠️ Job chạy trong cron **không thể** tự tạo thêm cron job khác (tránh vòng lặp lập lịch không kiểm soát).

### Nơi gửi kết quả (`deliver`)

| Giá trị | Mô tả |
|---|---|
| `"origin"` | Về nơi tạo job (mặc định trên platform nhắn tin) |
| `"local"` | Chỉ lưu file local `~/.hermes/cron/output/` (mặc định trên CLI) |
| `"telegram"` | Kênh home Telegram |
| `"telegram:123456"` | Chat Telegram cụ thể theo ID |
| `"telegram:-100123:17585"` | Topic Telegram cụ thể (`chat_id:thread_id`) |
| `"discord"`, `"slack"`, `"whatsapp"`, `"signal"`... | Kênh home của platform tương ứng |
| `"all"` | Gửi tới mọi kênh home đã kết nối (tính ở thời điểm chạy) |
| `"telegram,discord"` | Gửi tới danh sách cụ thể, phân tách bằng dấu phẩy |
| `"origin,all"` | Gửi về nơi gốc **và** mọi kênh khác |

Agent không cần gọi `send_message` trong prompt cho đích đến này — Hermes tự gửi response cuối cùng.

### Tắt wrapper & im lặng có điều kiện

```yaml
cron:
  wrap_response: false   # Gửi nguyên văn output, không kèm header/footer "Cronjob Response: ..."
```

Job giám sát chỉ báo khi có lỗi — dùng marker `[SILENT]`:
```text
Check if nginx is running. If everything is healthy, respond with only [SILENT].
Otherwise, report the issue.
```
> Job lỗi luôn được gửi báo, kể cả có `[SILENT]` — chỉ run thành công mới bị im lặng.

### Chế độ không-agent (script thuần, 0 token)

Cho các job watchdog/heartbeat đơn giản không cần LLM suy luận:

```bash
hermes cron create "every 5m" \
  --no-agent \
  --script memory-watchdog.sh \
  --deliver telegram \
  --name "memory-watchdog"
```

- stdout của script → gửi nguyên văn làm tin nhắn.
- stdout rỗng → im lặng, không gửi gì (pattern watchdog: "chỉ nói khi có gì bất thường").
- Exit code lỗi/timeout → luôn gửi alert (watchdog hỏng không thể im lặng).
- Script phải nằm trong `~/.hermes/scripts/`.

Có thể nhờ agent tự thiết lập:
```text
Ping me on Telegram if RAM is over 85%, every 5 minutes.
```

### Gate `wakeAgent` — tiết kiệm token cho job kiểm tra thường xuyên

Script kiểm tra trước (`script=`) có thể quyết định có cần "đánh thức" agent hay không bằng dòng stdout cuối:
```text
{"wakeAgent": false}
```
Khi bỏ qua, mặc định là `true`. Hữu ích cho job poll mỗi 1-5 phút chỉ cần gọi LLM khi trạng thái thực sự thay đổi.

### Nối chuỗi job với `context_from`

Job B nhận output gần nhất của Job A làm context tự động — không cần A "nhớ" B, vì mỗi job chạy session độc lập:

```python
# Job 1: thu thập dữ liệu thô
cronjob(action="create", prompt="Fetch top 10 AI stories...", schedule="0 7 * * *", name="AI News Collector")

# Job 2: nhận output Job 1 làm context
cronjob(action="create", prompt="Score each story 1-10...", schedule="30 7 * * *",
        context_from="<job1_id>", name="AI News Triage")

# Job 3: nhận output Job 2
cronjob(action="create", prompt="Write 3 tweet drafts...", schedule="0 8 * * *",
        context_from="<job2_id>", name="AI News Brief")
```
Có thể dùng tên hoặc danh sách nhiều job: `context_from=["job_a", "job_b"]`.

### Định dạng lịch chạy

```text
30m              → Chạy 1 lần sau 30 phút
2h                → Chạy 1 lần sau 2 giờ
every 30m         → Lặp mỗi 30 phút
every 2h          → Lặp mỗi 2 giờ
0 9 * * *         → Mỗi ngày 9:00 sáng (cron expression)
0 9 * * 1-5       → Các ngày trong tuần, 9:00 sáng
0 */6 * * *       → Mỗi 6 giờ
2026-03-15T09:00:00  → Một lần đúng vào thời điểm ISO timestamp
```

### Giới hạn toolset cho job cron

```text
cronjob(action="create", name="weekly-news-summary",
        schedule="every sunday 9am",
        enabled_toolsets=["web", "file"],   # chỉ web + file, không terminal/browser
        prompt="Summarize this week's AI news: ...")
```
Hoặc cấu hình mặc định cho toàn bộ cron qua `hermes tools` → chọn platform "cron".

### Lưu trữ & viết prompt tự đủ thông tin

Job lưu ở `~/.hermes/cron/jobs.json`, output ở `~/.hermes/cron/output/{job_id}/{timestamp}.md`.

> ⚠️ Mỗi job chạy trong session hoàn toàn mới — **không nhớ gì** từ lần chạy trước (trừ qua `context_from`). Prompt phải tự chứa đủ thông tin.

**Tệ:** `"Check on that server issue"`
**Tốt:** `"SSH vào server 192.168.1.100 với user 'deploy', kiểm tra nginx bằng 'systemctl status nginx', xác nhận https://example.com trả về HTTP 200."`

---

## 2. Subagent Delegation — Giao việc song song

Tool `delegate_task` sinh ra agent con (`AIAgent`) độc lập với context riêng, toolset hạn chế, terminal session riêng. Chỉ phần tóm tắt cuối cùng của con được đưa vào context của cha.

### Giao 1 việc

```python
delegate_task(
    goal="Debug why tests fail",
    context="Error: assertion in test_foo.py line 42",
    toolsets=["terminal", "file"]
)
```

### Giao nhiều việc song song (tối đa 3 mặc định)

```python
delegate_task(tasks=[
    {"goal": "Research topic A", "toolsets": ["web"]},
    {"goal": "Research topic B", "toolsets": ["web"]},
    {"goal": "Fix the build", "toolsets": ["terminal", "file"]}
])
```

> ⚠️ **Subagent không biết gì cả** — bắt đầu hội thoại hoàn toàn mới, không có lịch sử của cha. Mọi thông tin cần thiết phải nằm trong `goal` + `context`.

**Sai:** `delegate_task(goal="Fix the error")` — con không biết "lỗi" nào.
**Đúng:**
```python
delegate_task(
    goal="Fix the TypeError in api/handlers.py",
    context="""File api/handlers.py có TypeError ở dòng 47: 'NoneType' object has no attribute 'get'.
    Function process_request() nhận dict từ parse_body(), nhưng parse_body() trả None khi thiếu Content-Type.
    Project ở /home/user/myproject, dùng Python 3.11."""
)
```

### Chọn toolset theo loại task

| Mẫu toolset | Dùng cho |
|---|---|
| `["terminal", "file"]` | Code, debug, sửa file, build |
| `["web"]` | Research, tra cứu tài liệu |
| `["terminal", "file", "web"]` | Task full-stack (mặc định) |
| `["file"]` | Phân tích chỉ-đọc, review code không chạy |

**Tool luôn bị chặn với subagent (leaf):** `delegation`, `clarify`, `memory`, `code_execution`, `send_message`.

### Giới hạn số bước & timeout

```python
delegate_task(
    goal="Quick file check",
    context="Check if /etc/nginx/nginx.conf exists",
    max_iterations=10   # task đơn giản, không cần nhiều bước
)
```
Mặc định: 50 bước/con. Con bị coi là "treo" và bị kill nếu im lặng quá `delegation.child_timeout_seconds` (mặc định 600s = 10 phút).

### Theo dõi subagent đang chạy

```text
/agents     # (alias /tasks) — overlay quan sát cây subagent trong TUI, có kill/pause
```

### Đệ quy — orchestrator con (nâng cao)

Mặc định delegation là **flat**: con không thể giao việc tiếp. Muốn cho con làm "orchestrator" giao việc cho cháu:
```python
delegate_task(
    goal="Survey three code review approaches and recommend one",
    role="orchestrator",
    context="...",
)
```
Bị giới hạn bởi `delegation.max_spawn_depth` (mặc định 1 = flat, nên `role="orchestrator"` không có tác dụng ở default). Nâng lên 2+ để cho phép cây sâu hơn.

> ⚠️ **Cảnh báo chi phí:** `max_spawn_depth: 3` + `max_concurrent_children: 3` có thể tạo ra 27 agent con đồng thời. Mỗi tầng nhân chi phí lên — chỉ nâng khi thực sự cần.

### Quan trọng: delegate_task không bền (not durable)

`delegate_task` chạy **đồng bộ trong lượt hiện tại** — nếu cha bị ngắt (người dùng gửi tin mới, `/stop`, `/new`), mọi con đang chạy bị huỷ ngay, kết quả mất.

**Cho việc dài hạn cần sống sót qua ngắt quãng, dùng:**
- `cronjob` — chạy session riêng biệt, không bị ảnh hưởng khi cha bị ngắt.
- `terminal(background=True, notify_on_complete=True)` — lệnh shell chạy nền độc lập.

### Cấu hình

```yaml
delegation:
  max_iterations: 50
  # max_concurrent_children: 3
  # max_spawn_depth: 1
  model: "google/gemini-3-flash-preview"   # Model rẻ hơn cho subagent (tuỳ chọn)
  provider: "openrouter"
```

> Agent tự quyết định khi nào nên giao việc — bạn không cần yêu cầu rõ ràng, nó sẽ tự làm khi hợp lý.

---

## 3. Messaging Gateway — Tổng quan

Gateway là 1 process chạy nền duy nhất, kết nối tới tất cả platform đã cấu hình (Telegram, Discord, Slack, WhatsApp, Signal, Email, SMS, Matrix, Mattermost, DingTalk, Feishu, WeCom, Weixin, BlueBubbles, QQ, Yuanbao, Microsoft Teams, LINE, Home Assistant...), xử lý session, chạy cron, gửi voice message.

### So sánh nhanh tính năng theo platform (rút gọn)

| Platform | Voice | Ảnh | File | Thread | Streaming |
|---|---|---|---|---|---|
| Telegram | ✅ | ✅ | ✅ | ✅ | ✅ |
| Discord | ✅ | ✅ | ✅ | ✅ | ✅ |
| Slack | ✅ | ✅ | ✅ | ✅ | ✅ |
| WhatsApp | — | ✅ | ✅ | — | ✅ |
| Signal | — | ✅ | ✅ | — | ✅ |
| Email | — | ✅ | ✅ | ✅ | — |
| SMS | — | — | — | — | — |

### Cài đặt nhanh

```bash
hermes gateway setup        # Wizard tương tác cho mọi platform
```

### Lệnh quản lý gateway

```bash
hermes gateway              # Chạy foreground
hermes gateway setup        # Cấu hình platform
hermes gateway install      # Cài user service (Linux/macOS launchd)
sudo hermes gateway install --system   # Linux: service hệ thống từ lúc boot
hermes gateway start
hermes gateway stop
hermes gateway status
```

### Lệnh chat phổ biến (dùng trong mọi platform)

| Lệnh | Chức năng |
|---|---|
| `/new` hoặc `/reset` | Bắt đầu hội thoại mới |
| `/model [provider:model]` | Xem/đổi model |
| `/status` | Thông tin session |
| `/stop` | Dừng agent đang chạy |
| `/approve` / `/deny` | Duyệt/từ chối lệnh nguy hiểm đang chờ |
| `/sethome` | Đặt chat này làm kênh nhận kết quả cron |
| `/compress` | Nén ngữ cảnh thủ công |
| `/title [tên]` | Đặt tên session |
| `/usage` | Xem token đã dùng |
| `/voice [on|off|tts]` | Điều khiển trả lời bằng giọng nói |
| `/background <prompt>` | Chạy task nền riêng |
| `/<tên-skill>` | Gọi skill bất kỳ |

### Bảo mật — Allowlist (mặc định: chặn tất cả)

```bash
TELEGRAM_ALLOWED_USERS=123456789,987654321
DISCORD_ALLOWED_USERS=123456789012345678
# Hoặc generic cho mọi platform:
GATEWAY_ALLOWED_USERS=123456789,987654321
# Cho phép tất cả (KHÔNG khuyến nghị với bot có quyền terminal):
GATEWAY_ALLOW_ALL_USERS=true
```

### DM Pairing — cấp quyền động không cần biết trước user ID

```bash
# Người dùng lạ DM bot sẽ nhận: "Pairing code: XKGH5N7P"
hermes pairing approve telegram XKGH5N7P
hermes pairing list
hermes pairing revoke telegram 123456789
```

### Phân quyền Admin vs User thường (theo từng platform)

```yaml
gateway:
  platforms:
    discord:
      extra:
        allow_from: ["111", "222", "333"]
        allow_admin_from: ["111"]                 # admin → mọi slash command
        user_allowed_commands: [status, model]    # user thường chỉ được dùng các lệnh này
```
Kiểm tra quyền của chính mình: `/whoami`.

### Ngắt agent giữa chừng — 3 chế độ

```yaml
display:
  busy_input_mode: steer   # interrupt (mặc định) | queue | steer
```
- `interrupt` — tin nhắn mới ngắt ngay agent đang chạy.
- `queue` — tin nhắn mới chờ tới khi việc hiện tại xong.
- `steer` — tin nhắn mới được tiêm vào ngay sau lần gọi tool tiếp theo, không ngắt hoàn toàn.

### Background Sessions trên platform nhắn tin

```text
/background Check all servers in the cluster and report any that are down
```
Spawn agent instance hoàn toàn riêng, không chặn chat chính, kết quả tự gửi về khi xong:
```text
✅ Background task complete
```

### Chạy gateway như service (Linux/macOS)

```bash
# Linux (systemd) — user service
hermes gateway install
sudo loginctl enable-linger $USER   # giữ chạy sau khi logout
journalctl --user -u hermes-gateway -f   # xem log

# Linux — system service (cho VPS/server headless, chạy từ lúc boot)
sudo hermes gateway install --system
sudo hermes gateway start --system

# macOS (launchd)
hermes gateway install
tail -f ~/.hermes/logs/gateway.log
```

### Quản lý nhiều platform cùng lúc

```text
/platform list                  # Xem trạng thái mọi adapter
/platform pause <name>          # Tạm dừng nhận tin từ 1 platform
/platform resume <name>         # Tiếp tục
```
Có **circuit breaker tự động**: lỗi lặp lại (mạng, rate-limit, 5xx) sẽ tự pause adapter — không tự resume, bạn phải `/platform resume` thủ công khi platform đã ổn định lại. Kiểm tra log: `~/.hermes/logs/gateway.log`.

### Mặc định tối ưu cho Telegram (mobile-friendly)

- `tool_progress` mặc định **off** — không spam breadcrumb mỗi tool call.
- `interim_assistant_messages` mặc định **on** — agent vẫn báo đang làm gì giữa lượt.
- `long_running_notifications` mặc định **on** — 1 bubble cập nhật "⏳ Working — N min" mỗi vài phút.

```yaml
display:
  platforms:
    telegram:
      tool_progress: new          # bật lại nếu muốn xem chi tiết hơn
      cleanup_progress: true      # tự xoá bubble tiến trình sau khi có kết quả cuối
```

---

## 4. Telegram — Cài đặt chi tiết

### Bước 1: Tạo bot qua @BotFather

1. Mở Telegram, tìm **@BotFather**.
2. Gửi `/newbot`.
3. Chọn tên hiển thị (vd "Hermes Agent").
4. Chọn username — phải kết thúc bằng `bot` (vd `my_hermes_bot`).
5. BotFather trả về token, dạng: `123456789:ABCdefGHIjklMNOpqrSTUvwxYZ`

> ⚠️ Giữ token bí mật. Nếu lộ, revoke ngay bằng `/revoke` trong BotFather.

### Bước 2: Tuỳ chỉnh bot (tuỳ chọn)

| Lệnh BotFather | Chức năng |
|---|---|
| `/setdescription` | Text "Bot này làm gì?" |
| `/setabouttext` | Text ngắn trên trang profile |
| `/setuserpic` | Avatar bot |
| `/setcommands` | Định nghĩa menu lệnh (`/`) |
| `/setprivacy` | Quyết định bot có thấy hết tin trong group không |

Gợi ý set commands:
```text
help - Show help information
new - Start a new conversation
sethome - Set this chat as the home channel
```

### Bước 3: Privacy Mode — QUAN TRỌNG cho group

Mặc định **bật** — bot chỉ thấy: lệnh bắt đầu bằng `/`, reply trực tiếp vào tin của bot, service message.

**Tắt privacy mode:**
1. Message @BotFather → `/mybots` → chọn bot → **Bot Settings → Group Privacy → Turn off**.

> ⚠️ **Phải xoá và thêm lại bot vào group** sau khi đổi setting — Telegram cache trạng thái privacy lúc bot join.

> 💡 Cách khác: thăng bot làm **admin nhóm** — admin bot luôn thấy hết tin bất kể privacy mode.

### Bước 4: Lấy User ID của bạn

User ID **không phải** username — là 1 số (vd `123456789`). Message **@userinfobot** để lấy ngay.

### Bước 5: Cấu hình Hermes

**Cách A — Wizard (khuyến nghị):**
```bash
hermes gateway setup
```
Chọn Telegram, nhập token + user ID được phép.

**Cách B — Thủ công**, thêm vào `~/.hermes/.env`:
```bash
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
TELEGRAM_ALLOWED_USERS=123456789    # Phân tách bằng dấu phẩy nếu nhiều người
```

**Khởi động gateway:**
```bash
hermes gateway
```

### Webhook Mode — cho triển khai cloud (Fly.io, Railway...)

Mặc định Hermes dùng **long polling** (gateway tự hỏi Telegram liên tục — máy phải luôn chạy). Với cloud platform có auto-wake khi có traffic vào, **webhook mode** tiết kiệm hơn (máy có thể ngủ khi rảnh):

```bash
# ~/.hermes/.env
TELEGRAM_WEBHOOK_URL=https://my-app.fly.dev/telegram
# TELEGRAM_WEBHOOK_PORT=8443        # tuỳ chọn, mặc định 8443
# TELEGRAM_WEBHOOK_SECRET=mysecret  # tuỳ chọn, khuyến nghị cho production
```

Ví dụ deploy Fly.io:
```bash
fly secrets set TELEGRAM_WEBHOOK_URL=https://my-app.fly.dev/telegram
fly secrets set TELEGRAM_WEBHOOK_SECRET=$(openssl rand -hex 32)
fly deploy
```

### Proxy (nếu Telegram bị chặn ở khu vực của bạn)

```yaml
# config.yaml
telegram:
  proxy_url: "socks5://127.0.0.1:1080"
```
Hoặc env: `TELEGRAM_PROXY=socks5://127.0.0.1:1080`. Hỗ trợ `http://`, `https://`, `socks5://`.

### Home Channel — nơi nhận kết quả cron

```text
/sethome
```
Hoặc thủ công trong `.env`:
```bash
TELEGRAM_HOME_CHANNEL=-1001234567890
TELEGRAM_HOME_CHANNEL_NAME="My Notes"
```
> Group chat ID là số âm. DM cá nhân dùng chính user ID của bạn.

### Voice Messages

**Nhận (Speech-to-Text):** tự động transcribe — `local` (faster-whisper, không cần key), `groq` (cần `GROQ_API_KEY`), `openai` (cần `VOICE_TOOLS_OPENAI_KEY`).

**Gửi (Text-to-Speech):** trả về dạng voice bubble Telegram thật. OpenAI/ElevenLabs ra Opus sẵn. Edge TTS (mặc định, free) cần **ffmpeg** để convert sang Opus:
```bash
sudo apt install ffmpeg   # Ubuntu/Debian
```

### Dùng trong Group Chat

```yaml
# config.yaml
telegram:
  require_mention: true       # Chỉ trả lời khi được @mention/reply/lệnh slash
  mention_patterns:
    - "^\\s*chompy\\b"         # Wake word tuỳ chỉnh (regex)
  ignored_threads:
    - 31                       # Bỏ qua hoàn toàn các topic này
```

### Private Chat Topics — nhiều "workspace" tách biệt trong 1 DM

Từ Bot API 9.4 (2/2026), có thể tạo topic ngay trong DM 1-1, mỗi topic có session/context riêng hoàn toàn:

```yaml
platforms:
  telegram:
    extra:
      dm_topics:
      - chat_id: 123456789
        topics:
        - name: General
        - name: Website
        - name: Research
          skill: arxiv          # Tự nạp skill khi mở topic này
```
Hermes tự gọi `createForumTopic` cho topic chưa có `thread_id`, lưu lại tự động.

### Group Forum Topic — gắn skill theo topic trong supergroup

```yaml
platforms:
  telegram:
    extra:
      group_topics:
      - chat_id: -1001234567890
        topics:
        - name: Engineering
          thread_id: 5
          skill: software-development
        - name: Research
          thread_id: 12
          skill: arxiv
```
> Tìm `thread_id`: mở topic trên Telegram Web/Desktop, xem URL `t.me/c/1234567890/5` → `5` là thread_id.

### Model Picker tương tác

Gửi `/model` không kèm gì → hiện inline keyboard chọn provider rồi model. Hoặc gõ trực tiếp `/model <tên model>` để bỏ qua picker, thêm `--global` để áp dụng xuyên session.

### Per-Channel Prompt — system prompt riêng theo group/topic

```yaml
telegram:
  channel_prompts:
    "-1001234567890": |
      You are a research assistant. Focus on academic sources, citations.
    "42": |
      This topic is for creative writing feedback. Be warm and constructive.
```
Topic-level ưu tiên hơn group-level.

### Khắc phục sự cố

| Vấn đề | Cách sửa |
|---|---|
| Bot không phản hồi gì | Kiểm tra `TELEGRAM_BOT_TOKEN` đúng chưa, xem log `hermes gateway` |
| Bot trả "unauthorized" | User ID chưa có trong `TELEGRAM_ALLOWED_USERS` |
| Bot không thấy tin nhắn group | Privacy mode đang bật — tắt đi (Bước 3), **nhớ xoá & add lại bot** |
| Voice không transcribe | Cài `faster-whisper` hoặc set `GROQ_API_KEY`/`VOICE_TOOLS_OPENAI_KEY` |
| Voice reply ra file thường, không phải bubble | Cài `ffmpeg` |
| Webhook không nhận update | Kiểm tra URL public reachable, SSL/TLS hoạt động, firewall |

### Exec Approval — duyệt lệnh nguy hiểm ngay trong chat

```text
⚠️ This command is potentially dangerous (recursive delete). Reply "yes" to approve.
```
Trả lời "yes"/"y" để duyệt, "no"/"n" để từ chối.

---

*Nguồn: hermes-agent.nousresearch.com/docs (Features → Automation: Cron, Delegation; Messaging Platforms: Gateway, Telegram) — Nous Research, MIT License.*
