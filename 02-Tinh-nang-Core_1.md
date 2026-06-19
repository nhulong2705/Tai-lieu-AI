# Hermes Agent — Phần 2: Tools, Skills, Memory, Context Files

> Biên soạn từ docs chính thức: hermes-agent.nousresearch.com/docs
> Phạm vi: Tools & Toolsets, Skills System, Persistent Memory, Context Files (AGENTS.md/SOUL.md), Personality.

---

## Mục lục

1. [Tools & Toolsets](#1-tools--toolsets)
2. [Skills System](#2-skills-system)
3. [Persistent Memory](#3-persistent-memory)
4. [Context Files (AGENTS.md, SOUL.md...)](#4-context-files-agentsmd-soulmd)
5. [Personality](#5-personality)

---

## 1. Tools & Toolsets

Tool là các hàm mở rộng khả năng của agent, được nhóm thành **toolset** — có thể bật/tắt theo từng platform.

### Các nhóm tool chính

| Nhóm | Ví dụ | Mô tả |
|---|---|---|
| **Web** | `web_search`, `web_extract` | Tìm kiếm & trích xuất nội dung web |
| **Terminal & Files** | `terminal`, `process`, `read_file`, `patch` | Chạy lệnh và xử lý file |
| **Browser** | `browser_navigate`, `browser_snapshot`, `browser_vision` | Điều khiển browser tự động (text + vision) |
| **Media** | `vision_analyze`, `image_generate`, `text_to_speech` | Phân tích/sinh ảnh, âm thanh |
| **Điều phối agent** | `todo`, `clarify`, `execute_code`, `delegate_task` | Lập kế hoạch, hỏi rõ yêu cầu, chạy code, giao task cho subagent |
| **Memory & recall** | `memory`, `session_search` | Trí nhớ lâu dài và tra cứu session cũ |
| **Tự động hoá** | `cronjob`, `send_message` | Job lập lịch + gửi tin nhắn ra ngoài |
| **Tích hợp** | `ha_*`, MCP server tools | Home Assistant, MCP... |

### Dùng toolset

```bash
hermes chat --toolsets "web,terminal"   # Chỉ dùng toolset cụ thể
hermes tools                            # Xem & cấu hình tool theo platform (tương tác)
```

Toolset phổ biến: `web`, `search`, `terminal`, `file`, `browser`, `vision`, `image_gen`, `skills`, `tts`, `todo`, `memory`, `session_search`, `cronjob`, `code_execution`, `delegation`, `clarify`, `homeassistant`, `messaging`, `discord`, `debugging`, `safe`.

### Terminal Backend (tóm tắt — xem chi tiết ở Phần 1)

```yaml
terminal:
  backend: local    # local | docker | ssh | singularity | modal | daytona
  cwd: "."
  timeout: 180
```

### Quản lý process nền

```python
terminal(command="pytest -v tests/", background=true)
# Trả về: {"session_id": "proc_abc123", "pid": 12345}

process(action="list")                              # Xem mọi process đang chạy
process(action="poll", session_id="proc_abc123")    # Kiểm tra trạng thái
process(action="wait", session_id="proc_abc123")    # Chờ tới khi xong
process(action="log", session_id="proc_abc123")     # Xem log đầy đủ
process(action="kill", session_id="proc_abc123")    # Dừng
process(action="write", session_id="proc_abc123", data="y")  # Gửi input
```
`pty=true` cho phép dùng CLI tương tác như Codex, Claude Code.

### Sudo

Nếu lệnh cần sudo, Hermes hỏi mật khẩu (cache trong session). Hoặc đặt sẵn trong `.env`:
```bash
SUDO_PASSWORD=...
```

---

## 2. Skills System

Skill là tài liệu kiến thức theo nhu cầu (on-demand) mà agent tải khi cần — dùng pattern **progressive disclosure** để tiết kiệm token, tương thích chuẩn mở agentskills.io.

Tất cả skill nằm ở **`~/.hermes/skills/`** — nguồn chính. Agent có thể tự sửa/xoá bất kỳ skill nào.

### Dùng skill

```text
/gif-search funny cats
/axolotl help me fine-tune Llama 3 on my dataset
/github-pr-workflow create a PR for the auth refactor
/plan design a rollout for migrating our auth provider
/excalidraw                  # chỉ tên skill → load và để agent hỏi cần gì
```

Hoặc qua hội thoại tự nhiên:
```bash
hermes chat --toolsets skills -q "What skills do you have?"
```

### Progressive Disclosure — cách tải skill tiết kiệm token

```text
Level 0: skills_list()           → danh sách {tên, mô tả, category}   (~3k token)
Level 1: skill_view(name)        → Nội dung đầy đủ
Level 2: skill_view(name, path)  → File tham khảo cụ thể
```
Agent chỉ tải nội dung đầy đủ khi thực sự cần.

### Cấu trúc 1 file SKILL.md

```markdown
---
name: my-skill
description: Mô tả ngắn skill làm gì
version: 1.0.0
platforms: [macos, linux]     # Tuỳ chọn — giới hạn theo OS
metadata:
  hermes:
    tags: [python, automation]
    category: devops
---

# Skill Title

## When to Use
Điều kiện kích hoạt skill này.

## Procedure
1. Bước một
2. Bước hai

## Pitfalls
- Lỗi thường gặp và cách sửa

## Verification
Cách xác nhận đã làm đúng.
```

### Skill output — gửi file media tự động

Khi response chứa đường dẫn tuyệt đối tới file media (ví dụ `/home/user/screenshots/diagram.png`), gateway tự nhận diện và gửi file thật (ảnh Telegram, attachment Discord...) thay vì để lộ đường dẫn thô trong tin nhắn.

- `[[audio_as_voice]]` — gửi file âm thanh dạng voice message (Telegram, WhatsApp).
- `[[as_document]]` — gửi dạng file đính kèm thay vì ảnh preview (giữ chất lượng gốc, tránh bị nén nhỏ).

### Cài skill từ Skills Hub

```bash
hermes skills browse                              # Duyệt tất cả skill có sẵn
hermes skills search kubernetes                   # Tìm theo từ khoá
hermes skills inspect openai/skills/k8s           # Xem trước khi cài
hermes skills install openai/skills/k8s           # Cài (có quét an ninh)
hermes skills install official/security/1password
hermes skills install https://sharethis.chat/SKILL.md   # Cài trực tiếp từ URL
hermes skills list --source hub                   # Xem skill đã cài từ hub
hermes skills check                               # Kiểm tra có bản cập nhật upstream
hermes skills update                              # Cập nhật skill đã cài
hermes skills uninstall k8s                       # Gỡ
hermes skills reset google-workspace              # Bỏ trạng thái "user-modified" của skill bundled
hermes skills reset google-workspace --restore    # Khôi phục về bản gốc bundled
```

### Trust level khi cài skill từ bên ngoài

| Mức | Nguồn | Chính sách |
|---|---|---|
| `builtin` | Có sẵn trong Hermes | Luôn tin tưởng |
| `official` | `optional-skills/` trong repo | Tin tưởng built-in |
| `trusted` | openai/skills, anthropics/skills, huggingface/skills... | Chính sách thoáng hơn |
| `community` | skills.sh, GitHub repo tuỳ ý, marketplace khác | Cảnh báo non-dangerous có thể override bằng `--force`; verdict "dangerous" luôn bị chặn |

> Nếu gặp lỗi rate-limit khi cài/tìm skill, đặt `GITHUB_TOKEN` trong `.env` để tăng giới hạn từ 60 lên 5,000 request/giờ.

### Skill Bundles — gộp nhiều skill thành 1 slash command

```bash
hermes bundles create backend-dev \
  --skill github-code-review \
  --skill test-driven-development \
  --skill github-pr-workflow \
  -d "Backend feature work — review, test, PR workflow"
```
Sau đó:
```text
/backend-dev refactor the auth middleware
```
Agent nhận cả 3 skill cùng lúc. Quản lý:
```bash
hermes bundles list
hermes bundles show backend-dev
hermes bundles delete backend-dev
```

### Agent tự tạo skill (skill_manage — trí nhớ thủ tục)

Agent tự lưu skill khi: hoàn thành task phức tạp (5+ lần gọi tool), gặp lỗi rồi tìm ra cách đúng, được người dùng sửa cách làm, hoặc phát hiện workflow đáng lưu lại.

| Hành động | Dùng khi | Tham số chính |
|---|---|---|
| `create` | Skill mới hoàn toàn | `name`, `content` |
| `patch` | Sửa nhỏ (ưu tiên — tiết kiệm token) | `name`, `old_string`, `new_string` |
| `edit` | Viết lại toàn bộ | `name`, `content` |
| `delete` | Xoá skill | `name` |
| `write_file` | Thêm file phụ trợ | `name`, `file_path`, `file_content` |

### Yêu cầu phê duyệt trước khi agent ghi skill

```yaml
skills:
  write_approval: false   # true = mọi thay đổi skill phải chờ bạn duyệt
```
Khi bật:
```text
/skills pending             # liệt kê thay đổi đang chờ duyệt
/skills diff <id>           # xem diff đầy đủ
/skills approve <id>        # áp dụng (hoặc 'all')
/skills reject <id>         # bỏ (hoặc 'all')
```

### Skill ngoài (External Skill Directories)

Nếu có thư mục skill dùng chung giữa nhiều tool AI:
```yaml
skills:
  external_dirs:
    - ~/.agents/skills
    - /home/shared/team-skills
```
Skill cùng tên ở local luôn ưu tiên hơn external.

---

## 3. Persistent Memory

Hermes có trí nhớ giới hạn, được lọc kỹ, tồn tại qua các session — để nhớ sở thích, dự án, môi trường, và những gì đã học được.

### Hai file trí nhớ

| File | Mục đích | Giới hạn ký tự |
|---|---|---|
| **MEMORY.md** | Ghi chú cá nhân của agent — môi trường, quy ước, bài học | 2.200 chars (~800 token) |
| **USER.md** | Hồ sơ người dùng — sở thích, cách giao tiếp, kỳ vọng | 1.375 chars (~500 token) |

Cả hai lưu ở `~/.hermes/memories/`, được chèn vào system prompt như 1 snapshot "đông cứng" tại đầu session. Agent tự quản lý qua tool `memory` (add/replace/remove).

> Trí nhớ **không tự nén** — khi ghi vượt giới hạn, tool trả lỗi và agent phải tự dọn (gộp/xoá entry) trước khi thử lại.

### Nên lưu gì / không nên lưu gì

**Nên lưu (agent tự làm, không cần nhắc):**
- Sở thích người dùng: "Tôi thích TypeScript hơn JavaScript" → lưu vào `user`
- Thông tin môi trường: "Server này chạy Debian 12 với PostgreSQL 16" → `memory`
- Sửa lỗi đã được chỉ ra: "Không dùng sudo cho lệnh Docker, user đã trong group docker" → `memory`
- Quy ước: "Project dùng tab, dòng tối đa 120 ký tự" → `memory`
- Việc đã hoàn thành: "Đã chuyển database từ MySQL sang PostgreSQL ngày 15/01/2026"
- Yêu cầu rõ ràng: "Nhớ rằng API key xoay vòng mỗi tháng"

**Không nên lưu:**
- Thông tin hiển nhiên/mơ hồ
- Thông tin dễ tra lại (có thể web search)
- Dữ liệu thô lớn (code block, log file)
- Thông tin chỉ dùng 1 lần trong session
- Thông tin đã có trong SOUL.md/AGENTS.md

### Yêu cầu phê duyệt trước khi ghi nhớ

```yaml
memory:
  write_approval: false   # true = mọi lần ghi nhớ phải chờ bạn duyệt
```
Khi bật, trong CLI agent hỏi ngay; trên platform nhắn tin/review nền tự động, việc ghi được "staging" để duyệt sau:
```text
/memory pending
/memory approve <id>
/memory reject <id>
```

### Thông báo khi agent tự cập nhật trí nhớ ở nền

```yaml
display:
  memory_notifications: on   # off | on (mặc định) | verbose
```

### Session Search — tra cứu hội thoại cũ (khác với Memory)

| | Persistent Memory | Session Search |
|---|---|---|
| Dung lượng | ~1.300 token | Không giới hạn (mọi session) |
| Tốc độ | Tức thì (đã trong prompt) | ~20ms FTS5 query |
| Chi phí | Tốn token mỗi prompt | Miễn phí, chỉ tìm khi cần |
| Dùng khi | Thông tin quan trọng luôn cần | "Tuần trước mình có nói về X không?" |

```bash
hermes sessions list    # Xem lại các session cũ
```

### Cấu hình memory

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  write_approval: false
```

---

## 4. Context Files (AGENTS.md, SOUL.md...)

Hermes tự động phát hiện và nạp các file ngữ cảnh định hình hành vi của nó.

### Các loại file được hỗ trợ

| File | Mục đích | Cách tìm |
|---|---|---|
| **.hermes.md** / **HERMES.md** | Hướng dẫn project (ưu tiên cao nhất) | Đi ngược lên git root |
| **AGENTS.md** | Hướng dẫn project, quy ước, kiến trúc | Thư mục làm việc + thư mục con |
| **CLAUDE.md** | File ngữ cảnh của Claude Code (cũng nhận diện) | Thư mục làm việc + thư mục con |
| **SOUL.md** | Tính cách & giọng văn toàn cục của agent | Chỉ ở `HERMES_HOME/SOUL.md` |
| **.cursorrules** | Quy ước code của Cursor IDE | Chỉ ở thư mục làm việc |

> Chỉ **MỘT** loại file ngữ cảnh project được nạp mỗi session (ưu tiên: `.hermes.md` → `AGENTS.md` → `CLAUDE.md` → `.cursorrules`). **SOUL.md** luôn nạp riêng, độc lập, như identity của agent.

### AGENTS.md — file ngữ cảnh project chính

Phát hiện tiến triển theo thư mục con: khi agent đọc file trong `frontend/`, nó tự tìm `frontend/AGENTS.md` nếu có — không cần nạp hết mọi AGENTS.md ngay từ đầu (tránh phình system prompt).

```text
my-project/
├── AGENTS.md              ← Nạp lúc khởi động
├── frontend/
│   └── AGENTS.md          ← Phát hiện khi agent đọc file trong frontend/
└── backend/
    └── AGENTS.md          ← Phát hiện khi agent đọc file trong backend/
```

### Ví dụ AGENTS.md thực tế

```markdown
# Project Context

This is a Next.js 14 web application with a Python FastAPI backend.

## Architecture
- Frontend: Next.js 14 with App Router in `/frontend`
- Backend: FastAPI in `/backend`, uses SQLAlchemy ORM
- Database: PostgreSQL 16
- Deployment: Docker Compose on a Hetzner VPS

## Conventions
- Use TypeScript strict mode for all frontend code
- Python code follows PEP 8, use type hints everywhere
- All API endpoints return JSON with `{data, error, meta}` shape

## Important Notes
- Never modify migration files directly — use Alembic commands
- The `.env.local` file has real API keys, don't commit it
- Frontend port is 3000, backend is 8000, DB is 5432
```

### Bảo mật — chống prompt injection

Mọi file ngữ cảnh được quét trước khi nạp, chặn các pattern như: yêu cầu "bỏ qua hướng dẫn trước", chỉ thị ẩn trong HTML comment/div, lệnh rút trộm credential (`curl ... $API_KEY`), ký tự ẩn vô hình. Nếu phát hiện, file bị chặn hoàn toàn:
```text
[BLOCKED: AGENTS.md contained potential prompt injection (prompt_injection). Content not loaded.]
```

### Giới hạn dung lượng

| Giới hạn | Giá trị |
|---|---|
| Tối đa mỗi file | 20.000 ký tự (~7.000 token) |
| Tỷ lệ giữ đầu | 70% |
| Tỷ lệ giữ cuối | 20% |

### Mẹo viết AGENTS.md hiệu quả

1. Giữ ngắn gọn — agent đọc nó mỗi lượt.
2. Dùng heading `##` để chia phần: kiến trúc, quy ước, lưu ý quan trọng.
3. Cho ví dụ code cụ thể.
4. Nói rõ điều KHÔNG nên làm.
5. Liệt kê đường dẫn và port quan trọng.
6. Cập nhật khi project thay đổi — ngữ cảnh cũ tệ hơn không có ngữ cảnh.

---

## 5. Personality

### SOUL.md — định hình tính cách & giọng văn mặc định

**Vị trí:** `~/.hermes/SOUL.md` (hoặc `$HERMES_HOME/SOUL.md` nếu dùng home tuỳ chỉnh).

- Hermes tự tạo SOUL.md mặc định nếu chưa có.
- Chỉ nạp từ `HERMES_HOME` — **không** dò trong thư mục làm việc.
- Nếu file trống, không có gì được thêm vào prompt.

Ví dụ:
```markdown
# Soul

Bạn là một Senior DevOps Engineer.
Phản hồi ngắn gọn, trực tiếp và ưu tiên lệnh có thể chạy ngay.
Luôn cảnh báo trước khi thực hiện thay đổi có nguy cơ gây downtime.
```

### Personalities có sẵn (đổi nhanh trong phiên chat)

```text
/personality pirate
/personality kawaii
/personality concise
```

Có sẵn: `helpful`, `concise`, `technical`, `creative`, `teacher`, `kawaii`, `catgirl`, `pirate`, `shakespeare`, `surfer`, `noir`, `uwu`, `philosopher`, `hype`.

### Tự định nghĩa personality riêng

```yaml
# ~/.hermes/config.yaml
personalities:
  helpful: "You are a helpful, friendly AI assistant."
  kawaii: "You are a kawaii assistant! Use cute expressions..."
  pirate: "Arrr! Ye be talkin' to Captain Hermes..."
```

### So sánh SOUL.md vs AGENTS.md vs Personality

| | Phạm vi | Mục đích |
|---|---|---|
| **SOUL.md** | Toàn cục (mọi project) | Tính cách & giọng văn cố định của agent |
| **AGENTS.md** | Theo project | Quy tắc, kiến trúc, quy ước kỹ thuật |
| **/personality** | Theo session, đổi nhanh | Đổi giọng văn tạm thời (vui, nghiêm túc...) |
| **Skill** | Theo nhiệm vụ | Quy trình nhiều bước, tái sử dụng |

---

*Nguồn: hermes-agent.nousresearch.com/docs (User Guide → Features: Tools, Skills, Memory, Context Files, Personality) — Nous Research, MIT License.*
