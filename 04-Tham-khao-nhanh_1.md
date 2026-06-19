# Hermes Agent — Phần 4: Tham khảo nhanh (CLI & Slash Commands)

> Biên soạn từ docs chính thức: hermes-agent.nousresearch.com/docs/reference
> Phạm vi: Toàn bộ lệnh `hermes ...` trong terminal + toàn bộ slash command `/...` trong chat (CLI và messaging).
> Đây là tài liệu **tra cứu nhanh** — dùng Ctrl+F / tìm kiếm để nhảy tới lệnh cần.

---

## Mục lục

1. [Cú pháp chung & tuỳ chọn toàn cục](#1-cú-pháp-chung--tuỳ-chọn-toàn-cục)
2. [Bảng tổng hợp lệnh `hermes` theo nhóm](#2-bảng-tổng-hợp-lệnh-hermes-theo-nhóm)
3. [Chi tiết các lệnh dùng thường xuyên](#3-chi-tiết-các-lệnh-dùng-thường-xuyên)
4. [Slash Commands — CLI](#4-slash-commands--cli)
5. [Slash Commands — Messaging (Telegram/Discord/...)](#5-slash-commands--messaging)
6. [Quick Commands & Model Aliases tự định nghĩa](#6-quick-commands--model-aliases-tự-định-nghĩa)
7. [Liên kết tới 2 file Skills Catalog](#7-liên-kết-tới-2-file-skills-catalog)

---

## 1. Cú pháp chung & tuỳ chọn toàn cục

```bash
hermes [global-options] <command> [subcommand/options]
```

| Tuỳ chọn | Mô tả |
|---|---|
| `--version`, `-V` | Hiện phiên bản |
| `--profile <name>`, `-p <name>` | Chọn profile cho lần chạy này |
| `--resume <session>`, `-r <session>` | Tiếp tục session theo ID/tên |
| `--continue [name]`, `-c [name]` | Tiếp tục session gần nhất (hoặc theo tên) |
| `--worktree`, `-w` | Chạy trong git worktree riêng |
| `--yolo` | Bỏ qua xác nhận lệnh nguy hiểm |
| `--ignore-user-config` | Bỏ qua `config.yaml`, dùng default gốc |
| `--ignore-rules` | Bỏ qua AGENTS.md/SOUL.md/.cursorrules/memory/skill preload |
| `--tui` | Mở TUI (tương đương `HERMES_TUI=1`) |
| `--cli` | Buộc dùng CLI cổ điển |

---

## 2. Bảng tổng hợp lệnh `hermes` theo nhóm

### Chat & Model
| Lệnh | Mô tả |
|---|---|
| `hermes chat` | Chat tương tác hoặc 1 lần |
| `hermes -z "<prompt>"` | One-shot thuần — chỉ trả về text cuối, không banner/spinner (cho script) |
| `hermes model` | Chọn provider/model mặc định (tương tác) |
| `hermes fallback` | Quản lý chuỗi provider dự phòng |

### Gateway & Messaging
| Lệnh | Mô tả |
|---|---|
| `hermes gateway` | Chạy/quản lý messaging gateway |
| `hermes whatsapp` | Pairing WhatsApp |
| `hermes slack` | Sinh manifest app Slack |
| `hermes send` | Gửi 1 tin nhắn ra platform (không cần agent loop) |
| `hermes pairing` | Duyệt/thu hồi pairing code |

### Setup & Auth
| Lệnh | Mô tả |
|---|---|
| `hermes setup` | Wizard cấu hình toàn diện |
| `hermes portal` | Trạng thái Nous Portal + Tool Gateway |
| `hermes auth` | Quản lý credential pool (API key, OAuth) |
| `hermes secrets` | Lấy secret từ Bitwarden Secrets Manager |

### Chẩn đoán & Bảo trì
| Lệnh | Mô tả |
|---|---|
| `hermes doctor` | Kiểm tra cấu hình & dependency |
| `hermes status` | Trạng thái agent/auth/platform |
| `hermes dump` | Tóm tắt setup để dán vào issue/hỏi hỗ trợ |
| `hermes debug share` | Upload debug report, lấy link chia sẻ |
| `hermes logs` | Xem/tail/filter log |
| `hermes prompt-size` | Phân tích kích thước system prompt |
| `hermes security audit` | Quét lỗ hổng supply-chain (OSV.dev) |

### Backup & Restore
| Lệnh | Mô tả |
|---|---|
| `hermes backup` | Backup toàn bộ `~/.hermes` ra zip |
| `hermes import <zip>` | Khôi phục từ backup |
| `hermes checkpoints` | Quản lý shadow git store cho `/rollback` |

### Cấu hình
| Lệnh | Mô tả |
|---|---|
| `hermes config` | Xem/sửa/migrate config.yaml |
| `hermes tools` | Cấu hình tool theo platform |
| `hermes plugins` | Quản lý plugin (general, memory provider, context engine) |

### Tự động hoá
| Lệnh | Mô tả |
|---|---|
| `hermes cron` | Quản lý job lập lịch |
| `hermes kanban` | Bảng Kanban đa agent/đa project |
| `hermes webhook` | Webhook subscription event-driven |
| `hermes hooks` | Shell-script hooks |

### Skill & Bundle
| Lệnh | Mô tả |
|---|---|
| `hermes skills` | Browse/install/audit/quản lý skill |
| `hermes bundles` | Gộp nhiều skill thành 1 slash command |
| `hermes curator` | Bảo trì skill nền tự động |

### Tích hợp khác
| Lệnh | Mô tả |
|---|---|
| `hermes memory` | Cấu hình memory provider ngoài |
| `hermes mcp` | Quản lý MCP server |
| `hermes acp` | Chạy Hermes như ACP server (editor) |
| `hermes computer-use` | Cài driver Computer Use (macOS) |

### Session & Phân tích
| Lệnh | Mô tả |
|---|---|
| `hermes sessions` | Browse/export/prune/rename session |
| `hermes insights` | Phân tích token/cost/hoạt động |

### Quản lý cài đặt
| Lệnh | Mô tả |
|---|---|
| `hermes profile` | Quản lý nhiều profile độc lập |
| `hermes claw migrate` | Migrate từ OpenClaw sang Hermes |
| `hermes dashboard` | Mở web dashboard |
| `hermes update` | Cập nhật Hermes lên bản mới |
| `hermes uninstall` | Gỡ cài đặt |
| `hermes completion` | In script auto-complete cho shell |

---

## 3. Chi tiết các lệnh dùng thường xuyên

### `hermes chat`

```bash
hermes chat [options]
```

| Tuỳ chọn | Mô tả |
|---|---|
| `-q`, `--query "..."` | Prompt 1 lần, không tương tác |
| `-m`, `--model <model>` | Override model cho lần chạy này |
| `-t`, `--toolsets <csv>` | Bật toolset cụ thể |
| `--provider <provider>` | Buộc dùng provider cụ thể |
| `-s`, `--skills <name>` | Nạp sẵn skill (lặp lại hoặc phân tách dấu phẩy) |
| `-v`, `--verbose` | Output chi tiết |
| `-Q`, `--quiet` | Chế độ lập trình — ẩn banner/spinner |
| `--image <path>` | Đính kèm ảnh local |
| `--worktree` | Worktree riêng cho lần chạy |
| `--max-turns <N>` | Số bước tối đa mỗi lượt (mặc định 90) |

```bash
hermes chat -q "Summarize the latest PRs"
hermes chat --provider openrouter --model anthropic/claude-sonnet-4.6
hermes chat --toolsets web,terminal,skills
hermes chat --quiet -q "Return only JSON"
```

### `hermes -z` — one-shot cho script (khuyến nghị cho automation)

```bash
hermes -z "What's the capital of France?"
# → Paris.

answer=$(hermes -z "summarize this" < /path/to/file.txt)
```
Override nhanh không sửa config:
```bash
hermes -z "…" --provider openrouter --model openai/gpt-5.5
```

### `hermes model` vs `/model` — phân biệt quan trọng

| | Chạy ở đâu | Chức năng |
|---|---|---|
| `hermes model` | Terminal, ngoài session | Thêm provider mới, chạy OAuth, nhập API key, chọn endpoint tuỳ chỉnh |
| `/model` | Trong session chat | Chỉ chuyển giữa provider/model **đã cấu hình sẵn** |

> Muốn thêm provider mới: thoát session (`Ctrl+C` hoặc `/quit`), chạy `hermes model` từ terminal.

```text
/model                              # Xem model hiện tại + tuỳ chọn
/model claude-sonnet-4               # Đổi model
/model zai:glm-5                     # Đổi provider + model
/model custom:qwen-2.5               # Dùng model trên custom endpoint
/model claude-sonnet-4 --global      # Đổi và lưu thành default
```

### `hermes gateway`

```bash
hermes gateway run        # Chạy foreground (khuyến nghị cho WSL/Docker/Termux)
hermes gateway start      # Chạy như service đã cài
hermes gateway stop
hermes gateway status
hermes gateway install              # Cài user service (systemd/launchd)
sudo hermes gateway install --system   # Linux: service hệ thống từ boot
hermes gateway setup      # Wizard cấu hình platform
hermes gateway list       # Liệt kê mọi profile + trạng thái gateway
```
> WSL: dùng `hermes gateway run` (không phải `start`) vì systemd trên WSL không ổn định. Bọc trong tmux: `tmux new -s hermes 'hermes gateway run'`.

### `hermes send` — gửi tin nhắn 1 lần (không cần agent/gateway)

```bash
hermes send --to telegram "deploy finished"
echo "RAM 92%" | hermes send --to telegram:-1001234567890
hermes send --to discord:#ops --file /tmp/report.md
hermes send --list                  # Xem mọi target đã cấu hình
```
Định dạng target: `platform`, `platform:chat_id`, `platform:chat_id:thread_id`, `platform:#channel-name`.

### `hermes doctor` / `hermes dump` / `hermes debug share`

```bash
hermes doctor --fix        # Tự sửa lỗi nếu có thể
hermes dump                # Tóm tắt setup (dán vào issue/Discord khi cần hỗ trợ)
hermes dump --show-keys    # Kèm prefix API key đã redact
hermes debug share         # Upload log + system info, lấy link chia sẻ
hermes debug share --local # Chỉ in ra terminal, không upload
```

### `hermes backup` / `hermes import`

```bash
hermes backup                                  # Backup đầy đủ
hermes backup --quick --label "pre-upgrade"     # Backup nhanh chỉ state quan trọng
hermes import ~/hermes-backup-20260423.zip      # Khôi phục (có xác nhận)
hermes import ~/hermes-backup-20260423.zip --force
```
> Dừng gateway trước khi import để tránh xung đột process đang chạy.

### `hermes logs`

```bash
hermes logs                          # 50 dòng cuối agent.log
hermes logs -f                       # Theo dõi real-time
hermes logs gateway -n 100           # 100 dòng cuối gateway.log
hermes logs --level WARNING --since 1h
hermes logs --session abc123         # Lọc theo session ID
hermes logs list                     # Xem mọi file log + dung lượng
```

### `hermes sessions`

```bash
hermes sessions list
hermes sessions browse                          # Picker tương tác
hermes sessions export backup.jsonl
hermes sessions delete <session-id>
hermes sessions prune
hermes sessions rename <session-id> "tên mới"
hermes sessions stats
```

### `hermes update`

```bash
hermes update --check       # Xem có bản mới không, chưa cài
hermes update                # Cập nhật
hermes update --backup       # Backup trước khi cập nhật
hermes update --yes          # Không hỏi xác nhận
```
> Sau update, gateway tự khởi động lại để dùng code mới.

### `hermes profile` — nhiều instance Hermes độc lập

```bash
hermes profile list
hermes profile create work --clone     # Tạo profile mới, copy config từ profile hiện tại
hermes profile use work                 # Đặt làm default
hermes -p work chat -q "Hello from work profile"
hermes profile export work -o work-backup.tar.gz
```

### `hermes claw migrate` — chuyển từ OpenClaw

```bash
hermes claw migrate --dry-run                          # Xem trước, không ghi gì
hermes claw migrate --preset full                       # Migrate toàn bộ (không gồm secret)
hermes claw migrate --preset full --migrate-secrets     # Gồm cả API key
```

### `hermes mcp`

```bash
hermes mcp                                  # Picker tương tác — browse MCP đã được Nous duyệt
hermes mcp catalog                          # Liệt kê MCP có sẵn (dạng text)
hermes mcp install n8n                      # Cài 1 MCP từ catalog
hermes mcp add <name> --url <URL>           # Thêm MCP server tuỳ chỉnh
hermes mcp list
hermes mcp test <name>
```

### `hermes kanban` — bảng đa agent

```bash
hermes kanban init
hermes kanban create "Restart server" --assignee ops
hermes kanban list
hermes kanban show <id>
hermes kanban claim <id>
hermes kanban complete <id> --result "done"
hermes kanban dispatch
hermes kanban boards create atm10-server --name "ATM10 Server"
hermes kanban boards switch atm10-server
```

### `hermes dashboard` — web UI quản lý

```bash
hermes dashboard                          # Mở http://127.0.0.1:9119
hermes dashboard --port 8080 --no-open
```
> Cần `pip install hermes-agent[web]`. Tab Chat nhúng cần thêm extra `pty`.

---

## 4. Slash Commands — CLI

> Gõ `/` trong CLI để mở menu autocomplete. Hỗ trợ prefix matching: `/h` → `/help`, `/mod` → `/model`.

### Session

| Lệnh | Mô tả |
|---|---|
| `/new [name]` (= `/reset`) | Session mới, tuỳ chọn đặt tên ngay |
| `/clear` | Xoá màn hình + session mới |
| `/history` | Xem lịch sử hội thoại |
| `/retry` | Gửi lại tin nhắn gần nhất |
| `/undo` | Xoá lượt hỏi-đáp gần nhất |
| `/title` | Đặt tên session |
| `/compress [chủ đề]` | Nén ngữ cảnh thủ công |
| `/rollback` | Xem/khôi phục checkpoint filesystem |
| `/snapshot [create\|restore <id>\|prune]` | Snapshot trạng thái Hermes |
| `/stop` | Dừng mọi process nền |
| `/queue <prompt>` (= `/q`) | Xếp prompt cho lượt sau, không ngắt hiện tại |
| `/steer <prompt>` | Tiêm hướng dẫn giữa lượt, sau lần gọi tool tiếp theo |
| `/goal <text>` | Đặt mục tiêu agent tự làm qua nhiều lượt (Ralph loop) |
| `/subgoal <text>` | Thêm tiêu chí phụ cho goal đang chạy |
| `/resume [name]` | Tiếp tục session theo tên |
| `/sessions` | Picker session tương tác |
| `/status` | Thông tin session + recap |
| `/agents` (= `/tasks`) | Xem agent/task đang chạy |
| `/background <prompt>` (= `/bg`, `/btw`) | Chạy prompt ở session nền |
| `/branch [name]` (= `/fork`) | Nhánh session để thử hướng khác |
| `/handoff <platform>` | **Chỉ CLI.** Chuyển session sang platform nhắn tin |

### Cấu hình

| Lệnh | Mô tả |
|---|---|
| `/config` | Xem cấu hình hiện tại |
| `/model [tên]` | Xem/đổi model |
| `/personality` | Đặt personality |
| `/verbose` | Xoay vòng mức hiện tool: off→new→all→verbose |
| `/fast [normal\|fast\|status]` | Bật chế độ xử lý ưu tiên (nếu provider hỗ trợ) |
| `/reasoning` | Quản lý mức suy luận |
| `/skin` | Đổi theme hiển thị |
| `/statusbar` (= `/sb`) | Tắt/mở status bar |
| `/voice [on\|off\|tts\|status]` | Voice mode (ghi âm = `Ctrl+B`) |
| `/yolo` | Bật/tắt bỏ qua xác nhận lệnh nguy hiểm |
| `/footer [on\|off\|status]` | Footer metadata runtime cuối reply |
| `/busy [queue\|steer\|interrupt\|status]` | **Chỉ CLI.** Hành vi khi Enter lúc agent đang bận |

### Tools & Skills

| Lệnh | Mô tả |
|---|---|
| `/tools [list\|disable\|enable] [tên...]` | Quản lý tool cho session |
| `/toolsets` | Liệt kê toolset có sẵn |
| `/browser [connect\|disconnect\|status]` | Kết nối Chromium-family CDP local |
| `/skills` | Tìm/cài/quản lý skill |
| `/cron` | Quản lý job lập lịch |
| `/curator` | Bảo trì skill nền |
| `/kanban <action>` | Điều khiển bảng Kanban từ chat |
| `/reload-mcp` | Tải lại MCP server từ config |
| `/reload-skills` | Quét lại skill mới cài/xoá |
| `/reload` | Tải lại `.env` vào session đang chạy |
| `/plugins` | Liệt kê plugin đã cài |

### Info

| Lệnh | Mô tả |
|---|---|
| `/help` | Trợ giúp |
| `/usage` | Token, chi phí, hạn mức tài khoản (nếu có) |
| `/insights` | Phân tích sử dụng 30 ngày |
| `/platforms` (= `/gateway`) | Trạng thái gateway/platform |
| `/platform <list\|pause\|resume> [tên]` | Điều khiển từng adapter platform |
| `/paste` | Đính kèm ảnh clipboard |
| `/copy [số]` | Copy response gần nhất ra clipboard |
| `/image <path>` | Đính kèm ảnh local |
| `/debug` | Upload debug report |
| `/profile` | Tên profile + home directory hiện tại |

### Thoát

```text
/quit (= /exit)
/exit --delete    # Thoát VÀ xoá luôn lịch sử SQLite + transcript của session này
```

---

## 5. Slash Commands — Messaging

> Hoạt động trong Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant, Teams...

| Lệnh | Mô tả |
|---|---|
| `/new` / `/reset` | Hội thoại mới |
| `/status` | Thông tin session + recap |
| `/stop` | Dừng agent + process nền |
| `/model [provider:model]` | Đổi model |
| `/personality [tên]` | Đặt personality cho session |
| `/retry` / `/undo` | Gửi lại / xoá lượt gần nhất |
| `/sethome` | Đặt chat này làm kênh nhận cron |
| `/compress [chủ đề]` | Nén ngữ cảnh |
| `/title [tên]` | Đặt/xem tên session |
| `/resume [tên]` | Tiếp tục session đã đặt tên |
| `/usage` | Token + chi phí + hạn mức |
| `/insights [ngày]` | Phân tích sử dụng |
| `/reasoning [level\|show\|hide]` | Mức suy luận |
| `/voice [on\|off\|tts\|join\|leave\|status]` | Điều khiển giọng nói (join/leave chỉ Discord) |
| `/rollback [số]` | Checkpoint filesystem |
| `/background <prompt>` | Task nền, trả kết quả về chat khi xong |
| `/queue <prompt>` (= `/q`) | Xếp hàng prompt |
| `/steer <prompt>` | Tiêm hướng dẫn giữa lượt |
| `/goal <text>` | Mục tiêu dài hạn |
| `/footer [on\|off\|status]` | Footer metadata |
| `/curator [status\|run\|pin\|archive]` | Bảo trì skill |
| `/kanban <action>` | Điều khiển Kanban |
| `/reload-mcp` | Tải lại MCP |
| `/yolo` | Bỏ qua xác nhận lệnh nguy hiểm |
| `/commands [trang]` | Xem mọi lệnh + skill (phân trang) |
| `/approve [session\|always]` | Duyệt lệnh nguy hiểm đang chờ |
| `/deny` | Từ chối lệnh đang chờ |
| `/update` | Cập nhật Hermes |
| `/restart` | Khởi động lại gateway (graceful) |
| `/debug` | Upload debug report |
| `/help` | Trợ giúp |
| `/<tên-skill>` | Gọi skill bất kỳ |

### Lệnh chỉ-CLI vs chỉ-messaging

| Chỉ CLI | Chỉ Messaging |
|---|---|
| `/skin`, `/snapshot`, `/reload`, `/tools`, `/toolsets`, `/browser`, `/config`, `/platforms`, `/paste`, `/image`, `/statusbar`, `/plugins`, `/busy`, `/indicator`, `/redraw`, `/clear`, `/history`, `/copy`, `/handoff`, `/quit` | `/sethome`, `/update`, `/restart`, `/approve`, `/deny`, `/topic`, `/commands` |

Hoạt động ở **cả hai**: `/status`, `/background`, `/queue`, `/steer`, `/voice`, `/reload-mcp`, `/reload-skills`, `/rollback`, `/debug`, `/fast`, `/footer`, `/curator`, `/kanban`, `/sessions`, `/yolo`.

### Xác nhận trước lệnh phá hủy dữ liệu

| Lệnh | Phá hủy gì |
|---|---|
| `/clear` | Xoá session ID + lịch sử trong RAM |
| `/new` / `/reset` | Tạo session mới (rỗng) |
| `/undo` | Xoá lượt hỏi-đáp gần nhất |
| `/exit --delete` | Xoá vĩnh viễn lịch sử SQLite + transcript |

Tắt hỏi xác nhận toàn cục:
```yaml
approvals:
  destructive_slash_confirm: false
```

---

## 6. Quick Commands & Model Aliases tự định nghĩa

### Quick Commands — phím tắt cho lệnh shell hoặc slash command khác

```yaml
# ~/.hermes/config.yaml
quick_commands:
  status:
    type: exec
    command: systemctl status hermes-agent
  deploy:
    type: exec
    command: scripts/deploy.sh
  inbox:
    type: alias
    target: /gmail unread
```
Dùng: `/status`, `/deploy`, `/inbox`. Loại `exec` chạy lệnh thật (0 token); `alias` chuyển hướng sang slash command khác.

### Model Aliases — đặt tên ngắn cho model hay dùng

**Cách 1 — sửa YAML:**
```yaml
model_aliases:
  fav:
    model: claude-sonnet-4.6
    provider: anthropic
  grok:
    model: grok-4
    provider: x-ai
```

**Cách 2 — từ shell, không cần sửa YAML:**
```bash
hermes config set model.aliases.fav anthropic/claude-opus-4.6
```

Dùng trong chat:
```text
/model fav            # chỉ áp dụng session này
/model grok --global  # lưu luôn thành default
```

---

## 7. Liên kết tới 2 file Skills Catalog

Danh mục đầy đủ Bundled Skills (~80 skill có sẵn) và Optional Skills (~50 skill cài thêm) đã được biên soạn riêng trong:

- **`Huong-dan-Hermes-CLI.md`** — hướng dẫn CLI cơ bản (file đầu tiên)
- **`Danh-muc-Skills-Hermes-Agent.md`** — toàn bộ danh mục skill, chia theo nhóm, kèm mô tả + lệnh cài

Lệnh cài skill nhanh:
```bash
hermes skills install official/<category>/<skill>
hermes skills browse
```

---

*Nguồn: hermes-agent.nousresearch.com/docs/reference (CLI Commands Reference, Slash Commands Reference) — Nous Research, MIT License.*
