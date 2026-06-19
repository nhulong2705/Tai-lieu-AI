# Hermes Agent — Phần 1: Cơ bản, CLI/TUI, Cấu hình

> Biên soạn từ docs chính thức: hermes-agent.nousresearch.com/docs
> Phạm vi: Cài đặt, Quickstart, CLI, TUI, Configuration, Sessions, Docker, Security — tập trung vào phần dùng thực tế (bỏ Modal/Daytona/Singularity backend nâng cao, OpenRouter routing chuyên sâu).

---

## Mục lục

1. [Hermes Agent là gì](#1-hermes-agent-là-gì)
2. [Cài đặt](#2-cài-đặt)
3. [Chọn Provider & Model](#3-chọn-provider--model)
4. [Quickstart — chat đầu tiên](#4-quickstart--chat-đầu-tiên)
5. [CLI Interface](#5-cli-interface)
6. [TUI — giao diện hiện đại](#6-tui--giao-diện-hiện-đại)
7. [Cấu hình (config.yaml & .env)](#7-cấu-hình-configyaml--env)
8. [Terminal Backend (Local & Docker)](#8-terminal-backend-local--docker)
9. [Sessions — quản lý phiên làm việc](#9-sessions--quản-lý-phiên-làm-việc)
10. [Context Compression](#10-context-compression)
11. [Bảo mật & Khắc phục sự cố](#11-bảo-mật--khắc-phục-sự-cố)

---

## 1. Hermes Agent là gì

Hermes Agent là agent AI tự cải thiện (self-improving) của Nous Research — không phải coding-copilot gắn với 1 IDE, không phải chatbot wrapper quanh 1 API. Đặc điểm:

- **Vòng học tập khép kín** — tự tạo skill từ kinh nghiệm, tự cải thiện skill khi dùng, tự nhắc lưu lại kiến thức, tra cứu xuyên session (FTS5), mô hình hoá người dùng qua thời gian (Honcho).
- **Chạy mọi nơi** — không chỉ trên laptop. 6 backend terminal: local, Docker, SSH, Daytona, Singularity, Modal. Có thể chạy trên VPS $5/tháng, cluster GPU, hoặc serverless (ngủ khi không dùng, gần như miễn phí).
- **Sống ở nơi bạn sống** — CLI, Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Mattermost, Email, SMS, DingTalk, Feishu, WeCom, Weixin, QQ Bot, Home Assistant, Microsoft Teams, Google Chat... 20+ platform từ một gateway.
- **Tự động hoá theo lịch** — cron tích hợp, gửi kết quả tới bất kỳ platform nào.
- **Giao việc & chạy song song** — sinh subagent độc lập, Programmatic Tool Calling qua `execute_code` gộp nhiều bước thành 1 lần gọi LLM.
- **Hỗ trợ MCP** — kết nối bất kỳ MCP server nào để mở rộng tool.

---

## 2. Cài đặt

### Cài nhanh (Linux / macOS / WSL2 / Termux)
```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

### Windows (PowerShell, native)
```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

### Sau khi cài — reload shell
```bash
source ~/.bashrc   # hoặc: source ~/.zshrc
hermes             # Bắt đầu chat!
```

### Cài & mở Hermes Desktop (sau khi đã cài CLI)
```bash
hermes desktop
```

> Cài đặt tự động xử lý hết: Python (qua `uv`), Node.js v22, ripgrep, ffmpeg, clone repo, virtualenv, lệnh `hermes` toàn cục. Chỉ cần có `git`.

### Vị trí cài đặt

| Cách cài | Code ở đâu | Lệnh `hermes` | Thư mục dữ liệu |
|---|---|---|---|
| `pip install` | Python site-packages | `~/.local/bin/hermes` | `~/.hermes/` |
| Per-user (script cài) | `~/.hermes/hermes-agent/` | `~/.local/bin/hermes` (symlink) | `~/.hermes/` |
| Root-mode (`sudo`) | `/usr/local/lib/hermes-agent/` | `/usr/local/bin/hermes` | `/root/.hermes/` |

### Cài trên user không có sudo (VPS service account)
```bash
# 1. Admin có sudo cài thư viện hệ thống cho Chromium (1 lần)
sudo npx playwright install-deps chromium

# 2. User không có sudo chạy installer bình thường (tự bỏ qua bước cần root)
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# Hoặc bỏ qua hẳn phần browser automation nếu chạy headless:
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash -s -- --skip-browser
```

### Khắc phục lỗi cài đặt

| Lỗi | Cách sửa |
|---|---|
| `hermes: command not found` | `source ~/.bashrc` hoặc kiểm tra PATH |
| `API key not set` | `hermes model` để cấu hình provider |
| Thiếu config sau khi update | `hermes config check` rồi `hermes config migrate` |

```bash
hermes doctor   # Chẩn đoán toàn diện — luôn chạy đầu tiên khi có lỗi
```

---

## 3. Chọn Provider & Model

**Cách dễ nhất — Nous Portal** (1 subscription dùng 300+ model + Tool Gateway: web search, sinh ảnh, TTS, browser cloud):
```bash
hermes setup --portal
```

**Cách thủ công** — chọn provider/model tương tác:
```bash
hermes model
```

**Yêu cầu tối thiểu:** model phải có context ≥ **64,000 token**. Nếu chạy model local, đặt context ≥ 64K (`--ctx-size 65536` cho llama.cpp, `-c 65536` cho Ollama).

**Một số provider phổ biến:**

| Provider | Cách thiết lập |
|---|---|
| Nous Portal | OAuth qua `hermes model` |
| Anthropic (Claude) | OAuth (Max plan) hoặc API key |
| OpenRouter | Nhập API key |
| OpenAI Codex | OAuth qua ChatGPT |
| Google Gemini | `GOOGLE_API_KEY`/`GEMINI_API_KEY`, hoặc OAuth |
| DeepSeek | `DEEPSEEK_API_KEY` |
| Custom Endpoint | vLLM, SGLang, Ollama, hoặc bất kỳ API tương thích OpenAI |

**Cách lưu cấu hình đúng chỗ:**
```bash
hermes config set model anthropic/claude-opus-4.6
hermes config set terminal.backend docker
hermes config set OPENROUTER_API_KEY sk-or-...
```
> Secret/token → `~/.hermes/.env`. Cấu hình thường → `~/.hermes/config.yaml`. Dùng `hermes config set` để Hermes tự định tuyến đúng file.

---

## 4. Quickstart — chat đầu tiên

### Chạy thử
```bash
hermes            # CLI cổ điển
hermes --tui      # TUI hiện đại (khuyến nghị)
```

Câu hỏi mẫu để kiểm tra mọi thứ chạy ổn:
```text
Summarize this repo in 5 bullets and tell me what the main entrypoint is.
```
```text
Check my current directory and tell me what looks like the main project file.
```

**Dấu hiệu thành công:** banner hiện đúng model/provider, Hermes trả lời không lỗi, có thể dùng tool khi cần, hội thoại tiếp tục bình thường qua nhiều lượt.

### Kiểm tra resume session hoạt động
```bash
hermes --continue    # Tiếp tục session gần nhất
hermes -c            # Viết tắt
```

### Bộ công cụ khôi phục khi có lỗi (theo thứ tự)
```bash
hermes doctor
hermes model
hermes setup
hermes sessions list
hermes --continue
hermes gateway status
```

### Các lỗi thường gặp nhất

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| Hermes mở nhưng trả lời rỗng/lỗi | Sai provider/model | Chạy lại `hermes model`, kiểm tra auth |
| Custom endpoint trả về rác | Sai base URL/model name | Kiểm tra endpoint riêng trước |
| Gateway chạy nhưng không ai nhắn được | Bot token/allowlist chưa xong | `hermes gateway setup` lại, check `hermes gateway status` |
| `hermes --continue` không tìm thấy session | Đổi profile hoặc session chưa lưu | `hermes sessions list` |

---

## 5. CLI Interface

### Các cách chạy

```bash
hermes                                          # Phiên tương tác (mặc định)
hermes chat -q "Hello"                          # Chế độ 1 câu hỏi (non-interactive)
hermes chat --model "anthropic/claude-sonnet-4" # Model cụ thể
hermes chat --provider nous                     # Provider cụ thể
hermes chat --toolsets "web,terminal,skills"     # Toolset cụ thể
hermes -s hermes-agent-dev,github-auth          # Nạp skill khi mở phiên
hermes chat -s github-pr-workflow -q "open a draft PR"
hermes --continue                                # Tiếp tục session gần nhất (-c)
hermes --resume <session_id>                     # Tiếp tục session theo ID (-r)
hermes chat --verbose                            # Chế độ debug
hermes -w                                        # Worktree riêng — tương tác
hermes -w -q "Fix issue #123"                    # Worktree riêng — 1 câu hỏi
```

### Thanh trạng thái (Status Bar)

```text
⚕ claude-sonnet-4-20250514 │ 12.4K/200K │ [██████░░░░] 6% │ $0.06 │ 15m
```

| Phần tử | Ý nghĩa |
|---|---|
| Model name | Model hiện tại |
| Token count | Token context đã dùng / tối đa |
| Context bar | Thanh tiến trình, đổi màu theo mức |
| Cost | Chi phí ước tính của session |
| 🗜️ N | Số lần context đã bị tự động compress |
| ▶ N | Số background task đang chạy |
| ⚠ YOLO | Cảnh báo đang ở chế độ tự động phê duyệt |

**Màu thanh context:** Xanh lá <50% (còn nhiều chỗ) → Vàng 50-80% (đang đầy) → Cam 80-95% (gần giới hạn) → Đỏ ≥95% (nên `/compress`).

### Phím tắt

| Phím | Hành động |
|---|---|
| `Enter` | Gửi tin nhắn |
| `Alt+Enter` / `Ctrl+J` / `Shift+Enter` | Xuống dòng (multi-line) |
| `Alt+V` | Dán ảnh từ clipboard |
| `Ctrl+V` | Dán văn bản (+ thử dán ảnh) |
| `Ctrl+B` | Bật/tắt ghi âm voice mode |
| `Ctrl+G` hoặc `Ctrl+X Ctrl+E` | Mở buffer nhập trong `$EDITOR` |
| `Ctrl+C` | Ngắt agent (bấm 2 lần trong 2s để buộc thoát) |
| `Ctrl+D` | Thoát |
| `Ctrl+Z` | Đưa Hermes xuống nền (Unix), gõ `fg` để lấy lại |
| `Tab` | Chấp nhận gợi ý / autocomplete slash command |

> ⚠️ Trên Windows Terminal, `Alt+Enter` bị giữ để toggle fullscreen — dùng `Ctrl+Enter` hoặc `Ctrl+J` để xuống dòng.

### Slash command phổ biến

| Lệnh | Chức năng |
|---|---|
| `/help` | Hiện trợ giúp |
| `/model` | Xem/đổi model hiện tại |
| `/tools` | Liệt kê tool hiện có |
| `/skills browse` | Duyệt skill hub |
| `/background <prompt>` | Chạy prompt trong session nền riêng |
| `/skin` | Đổi giao diện CLI |
| `/voice on` | Bật voice mode (ghi âm bằng `Ctrl+B`) |
| `/reasoning high` | Tăng mức suy luận |
| `/title My Session` | Đặt tên session |
| `/status` | Xem thông tin session + recap |
| `/sessions` | Mở session picker tương tác |
| `/compress` | Tóm tắt ngữ cảnh để giảm token |
| `/usage` | Xem chi tiết token/chi phí |

### Quick Commands — lệnh shell không tốn token

```yaml
# ~/.hermes/config.yaml
quick_commands:
  status:
    type: exec
    command: systemctl status hermes-agent
  gpu:
    type: exec
    command: nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv,noheader
  restart:
    type: alias
    target: /gateway restart
```
Gõ `/status`, `/gpu`, `/restart` trong CLI hoặc bất kỳ platform nhắn tin. `exec` chạy lệnh thật, không gọi LLM — 0 token. `alias` chuyển hướng sang slash command khác.

### Nạp skill khi khởi động & dùng như slash command
```bash
hermes -s hermes-agent-dev,github-auth
```
```text
/gif-search funny cats
/github-pr-workflow create a PR for the auth refactor
/excalidraw            # chỉ tên skill → load và để agent hỏi cần gì
```

### Personalities (đổi giọng văn của agent)
```text
/personality pirate
/personality kawaii
/personality concise
```
Có sẵn: `helpful`, `concise`, `technical`, `creative`, `teacher`, `kawaii`, `catgirl`, `pirate`, `shakespeare`, `surfer`, `noir`, `uwu`, `philosopher`, `hype`.

Tự định nghĩa thêm trong `~/.hermes/config.yaml`:
```yaml
personalities:
  helpful: "You are a helpful, friendly AI assistant."
  pirate: "Arrr! Ye be talkin' to Captain Hermes..."
```

### Ngắt & điều hướng Agent giữa chừng

- Gõ tin nhắn mới + Enter khi agent đang chạy → ngắt và xử lý yêu cầu mới.
- `Ctrl+C` → ngắt thao tác hiện tại.
- Cấu hình hành vi khi bấm Enter lúc agent đang bận:
```yaml
display:
  busy_input_mode: "steer"   # "interrupt" (mặc định) | "queue" | "steer"
```
Hoặc trong phiên chat: `/busy queue`, `/busy steer`, `/busy interrupt`, `/busy status`.

### Background Sessions — chạy nền không gián đoạn

```text
/background Analyze the logs in /var/log and summarize any errors from today
```
Mỗi prompt `/background` tạo một **session agent hoàn toàn riêng biệt** trong thread daemon: không biết lịch sử session hiện tại, dùng cùng model/toolset/config, không chặn session chính. Có thể chạy nhiều task nền song song.

### Resume session (chi tiết)
```bash
hermes --continue                          # Session CLI gần nhất
hermes -c "my project"                     # Theo tên (lấy bản mới nhất nếu có nhiều phiên bản)
hermes --resume 20260225_143052_a1b2c3     # Theo ID cụ thể
hermes --resume "refactoring auth"         # Theo tiêu đề
```

---

## 6. TUI — giao diện hiện đại

TUI là giao diện hiện đại hơn, dùng cùng Python runtime, cùng session, cùng slash command với CLI cổ điển — chỉ là lớp hiển thị mượt hơn (mouse, modal overlay, không chặn input).

### Khởi chạy
```bash
hermes --tui                              # Mở TUI
hermes --tui -c                           # Tiếp tục session TUI gần nhất
hermes --tui -r 20260409_000000_aa11bb    # Tiếp tục theo ID
```

Đặt TUI làm mặc định:
```bash
export HERMES_TUI=1
hermes          # giờ sẽ dùng TUI
```

### Vì sao nên dùng TUI

- Vẽ banner ngay lập tức, không cảm giác đứng máy.
- Nhập liệu không chặn — gõ trước khi agent sẵn sàng, gửi ngay khi online.
- Overlay phong phú: model picker, session picker, approval đều hiện dạng modal.
- Kéo chuột để chọn văn bản như app bình thường.
- Không giật hình khi stream, không rác scrollback.

### Yêu cầu
- Node.js ≥ 20 (`hermes doctor` kiểm tra giúp).
- TTY (chạy non-interactive sẽ fallback về single-query mode).

### Một số slash command chỉ có ở TUI (hiện đẹp hơn)

| Lệnh | Hành vi trong TUI |
|---|---|
| `/sessions` | Modal session picker với preview |
| `/model` | Modal model picker theo provider, có gợi ý giá |
| `/agents` (hoặc `/tasks`) | Overlay quan sát subagent — cây trực quan, kill/pause |
| `/reload` | Đọc lại `.env` vào tiến trình TUI đang chạy (không cần restart) |
| `/mouse` | Tắt/mở mouse tracking |

---

## 7. Cấu hình (config.yaml & .env)

### Cấu trúc thư mục

```text
~/.hermes/
├── config.yaml     # Cấu hình (model, terminal, TTS, compression...)
├── .env             # API key và secret
├── auth.json        # OAuth credentials (Nous Portal...)
├── SOUL.md          # Identity chính của agent
├── memories/        # Trí nhớ lâu dài (MEMORY.md, USER.md)
├── skills/           # Skill do agent tự tạo
├── cron/             # Job lập lịch
├── sessions/         # Session từ gateway
└── logs/             # Log (secret tự redact)
```

### Quản lý cấu hình

```bash
hermes config              # Xem cấu hình hiện tại
hermes config edit         # Mở config.yaml trong editor
hermes config set KEY VAL  # Đặt giá trị cụ thể
hermes config check        # Kiểm tra thiếu option (sau khi update)
hermes config migrate      # Thêm option thiếu (tương tác)
```

### Thứ tự ưu tiên cấu hình (cao → thấp)

1. CLI arguments (ví dụ `--model`)
2. `~/.hermes/config.yaml`
3. `~/.hermes/.env` (bắt buộc cho secrets)
4. Default mặc định trong code

### Tham chiếu biến môi trường trong config.yaml
```yaml
auxiliary:
  vision:
    api_key: ${GOOGLE_API_KEY}
```

### Một vài khối cấu hình hay dùng

**Memory:**
```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  write_approval: false   # true = cần phê duyệt trước khi ghi nhớ
```

**Display (hiển thị tool, ngôn ngữ UI...):**
```yaml
display:
  tool_progress: all       # off | new | all | verbose
  skin: default
  personality: "kawaii"
  show_cost: false
  language: en              # en | zh | ja | de | es | fr | ... (không dịch nội dung agent trả lời)
  streaming: false          # stream từng token ra terminal
```

**Iteration budget (số bước tối đa mỗi lượt):**
```yaml
agent:
  max_turns: 90                # Số bước tối đa mỗi lượt hội thoại
  api_max_retries: 3           # Số lần retry trước khi chuyển fallback provider
  reasoning_effort: ""          # "" (medium) | none | minimal | low | medium | high | xhigh
  tool_use_enforcement: "auto"  # ép model phải gọi tool thật, không chỉ "nói sẽ làm"
```
Khi agent gần hết budget, hệ thống tự cảnh báo: ở 70% ("Start consolidating") và 90% ("Respond NOW").

**Privacy (ẩn PII trên gateway):**
```yaml
privacy:
  redact_pii: false   # true = ẩn số điện thoại, user ID... trước khi gửi cho LLM
```

**Quick commands** — xem mục CLI ở trên.

**Human delay (giả lập trả lời như người):**
```yaml
human_delay:
  mode: "off"          # off | natural | custom
  min_ms: 800
  max_ms: 2500
```

### Web Search Backend

```yaml
web:
  backend: firecrawl    # firecrawl | searxng | parallel | tavily | exa
```

| Backend | Env var | Search | Extract |
|---|---|---|---|
| Firecrawl (mặc định) | `FIRECRAWL_API_KEY` | ✔ | ✔ |
| SearXNG (free, self-host) | `SEARXNG_URL` | ✔ | — |
| Parallel | `PARALLEL_API_KEY` | ✔ | ✔ |
| Tavily | `TAVILY_API_KEY` | ✔ | ✔ |
| Exa | `EXA_API_KEY` | ✔ | ✔ |

### Auxiliary Models (model phụ cho vision, compression...)

Hermes dùng model "phụ" cho các việc nhỏ: phân tích ảnh, tóm tắt web, đặt tên session, compress context. Mặc định `provider: "auto"` → dùng luôn model chính của bạn.

```yaml
auxiliary:
  vision:
    provider: "auto"
    model: ""              # ví dụ "openai/gpt-4o", "google/gemini-2.5-flash"
  web_extract:
    provider: "auto"
    model: ""
  compression:
    model: ""               # "" = dùng model chat chính, hoặc đặt model rẻ/nhanh hơn
```

Cấu hình nhanh qua menu tương tác:
```bash
hermes model
# → chọn "Configure auxiliary models"
```

---

## 8. Terminal Backend (Local & Docker)

Hermes có 6 backend terminal — quyết định lệnh shell của agent thực thi ở đâu.

| Backend | Chạy ở đâu | Cô lập | Phù hợp |
|---|---|---|---|
| **local** | Máy của bạn | Không | Dev, dùng cá nhân |
| **docker** | 1 container Docker bền (share session) | Đầy đủ | Sandbox an toàn, CI/CD |
| **ssh** | Server remote | Ranh giới mạng | Dev remote, máy mạnh |
| modal/daytona/singularity | Cloud sandbox / HPC | Đầy đủ | Compute cloud tạm thời, HPC |

### Local (mặc định)
```yaml
terminal:
  backend: local
```
Không cần thiết lập gì thêm. Agent có quyền truy cập file giống tài khoản user của bạn — dùng `hermes tools` để tắt tool không muốn, hoặc đổi sang Docker để sandbox.

### Docker — sandbox an toàn

```yaml
terminal:
  backend: docker
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: false   # Mount thư mục launch vào /workspace
  docker_run_as_host_user: false         # true = file tạo ra thuộc về user host, không phải root
  docker_forward_env:                    # Forward biến môi trường từ host/.env vào container
    - "GITHUB_TOKEN"
  docker_volumes:                        # Mount thư mục host
    - "/home/user/projects:/workspace/projects"
    - "/home/user/data:/data:ro"
  container_cpu: 1
  container_memory: 5120        # MB
  container_persistent: true    # Giữ /workspace qua các session
```

**Đặc điểm quan trọng:** Hermes chạy **MỘT** container bền duy nhất, dùng chung qua mọi session/`/new`/subagent — không tạo container mới mỗi lần. Khi tắt Hermes, container vẫn chạy; lần mở sau tự gắn lại (qua label). Container chỉ bị xoá khi: đặt `docker_persist_across_processes: false`, idle quá `lifetime_seconds` (300s mặc định, chỉ khi không persist), hoặc bạn `docker rm -f` thủ công.

**Bảo mật mặc định:** `--cap-drop ALL` (chỉ giữ lại DAC_OVERRIDE, CHOWN, FOWNER), `--security-opt no-new-privileges`, `--pids-limit 256`, tmpfs giới hạn dung lượng cho `/tmp`.

**Yêu cầu:** Docker Desktop/Engine đang chạy. Hỗ trợ Podman: `HERMES_DOCKER_BINARY=podman`.

### SSH backend
```yaml
terminal:
  backend: ssh
  persistent_shell: true   # Giữ 1 bash session sống xuyên các lệnh (mặc định: true)
```
```bash
# Biến môi trường bắt buộc
TERMINAL_SSH_HOST=my-server.example.com
TERMINAL_SSH_USER=ubuntu
```

### Khắc phục lỗi backend

| Backend | Kiểm tra |
|---|---|
| Docker | `docker version` — nếu lỗi, fix Docker hoặc chuyển về `local` |
| SSH | Cần cả `TERMINAL_SSH_HOST` và `TERMINAL_SSH_USER` |
| Modal | Cần `MODAL_TOKEN_ID` hoặc `~/.modal.toml` |

Khi nghi ngờ, đặt `terminal.backend: local` để kiểm tra lệnh chạy được ở mức cơ bản trước.

---

## 9. Sessions — quản lý phiên làm việc

Mọi hội thoại (CLI, Telegram, Discord...) được lưu thành 1 session với lịch sử đầy đủ, lưu trong `~/.hermes/state.db` (SQLite + FTS5 full-text search).

### Resume session
```bash
hermes --continue                          # Session CLI gần nhất
hermes -c
hermes -c "my project"                     # Theo tên (lấy bản mới nhất nếu có nhiều phiên bản)
hermes --resume 20250305_091523_a1b2c3d4   # Theo ID cụ thể
hermes --resume "refactoring auth"         # Theo tiêu đề
```

### Đặt tên session
```text
/title my research project
```
```bash
hermes sessions rename 20250305_091523_a1b2c3d4 "refactoring auth module"
```

**Lưu ý:** tên phải duy nhất, tối đa 100 ký tự. Khi session bị compress, Hermes tự đặt tên nối tiếp: `"my project" → "my project #2"`.

### Cross-Platform Handoff — chuyển hội thoại sang platform khác

```text
# Trong phiên CLI
/handoff telegram
```
Hermes mở thread/topic mới trên platform đích, gắn cùng session ID, giữ nguyên toàn bộ lịch sử + tool call. Quay lại CLI bằng `/resume <title>` hoặc `hermes -r "<title>"`.

### Lệnh quản lý session

```bash
hermes sessions list                              # Liệt kê session gần đây
hermes sessions list --source telegram            # Lọc theo platform
hermes sessions export backup.jsonl               # Xuất toàn bộ session
hermes sessions export session.jsonl --session-id <id>   # Xuất 1 session
hermes sessions delete <session_id>                # Xoá 1 session
hermes sessions delete <session_id> --yes           # Xoá không cần xác nhận
hermes sessions rename <session_id> "tên mới"
hermes sessions prune                                # Xoá session cũ >90 ngày (đã kết thúc)
hermes sessions prune --older-than 30 --yes
hermes sessions stats                                # Thống kê tổng quan
```

### Tìm kiếm xuyên session

Agent có tool `session_search` tự động tra cứu lịch sử cũ bằng FTS5 khi bạn nhắc tới điều gì đó đã nói trước đây — không cần lặp lại thông tin.

### Cô lập session trong group chat

```yaml
group_sessions_per_user: true   # mặc định — mỗi người trong group có session riêng
```
Đặt `false` nếu muốn cả nhóm dùng chung 1 "bộ nhớ" — nhưng sẽ chia sẻ luôn token cost và trạng thái ngắt.

---

## 10. Context Compression

Khi hội thoại dài, Hermes tự tóm tắt phần giữa để không vượt giới hạn context.

```yaml
compression:
  enabled: true
  threshold: 0.50          # Compress khi đạt 50% giới hạn context
  protect_last_n: 20       # Giữ nguyên N message gần nhất, không compress
  protect_first_n: 3        # Giữ nguyên N message đầu (mục tiêu gốc luôn hiển thị)

auxiliary:
  compression:
    model: ""               # "" = dùng model chat chính, hoặc đặt model rẻ/nhanh hơn
```

> ⚠️ Model dùng để tóm tắt phải có context ≥ model chính — nếu nhỏ hơn, phần giữa hội thoại sẽ bị mất mà không có cảnh báo rõ.

Dùng `/compress` trong phiên chat để chủ động tóm tắt khi thấy thanh context chuyển màu vàng/đỏ.

---

## 11. Bảo mật & Khắc phục sự cố

### Cấu hình bảo mật cơ bản

```yaml
security:
  redact_secrets: true       # Tự ẩn API key/token trong tool output và log (mặc định: bật)
  tirith_enabled: true       # Quét lệnh terminal nguy hiểm trước khi chạy
  tirith_fail_open: true     # Cho chạy lệnh nếu scanner không khả dụng
```

### Nguyên tắc bảo mật khi triển khai thực tế

1. Không bật toàn bộ toolset nếu tác vụ chỉ cần quyền hạn giới hạn (`hermes tools`).
2. Không chia sẻ API key, token, nội dung `.env`.
3. Backup trước khi update: `hermes update --backup` hoặc `hermes backup`.
4. Dùng Docker backend để sandbox khi chạy trên VPS production.
5. Kiểm tra kỹ trước khi cho phép lệnh xoá file/thay đổi hệ thống.
6. Không cấp quyền bot messaging cho người lạ — dùng `unauthorized_dm_behavior: pair` (mặc định) để yêu cầu pairing code.
7. Không chạy Hermes production bằng tài khoản root nếu không cần thiết.

### Bộ lệnh chẩn đoán

```bash
hermes doctor          # Kiểm tra cấu hình & dependency
hermes status          # Trạng thái hiện tại
hermes logs            # Xem log
hermes config check    # Kiểm tra config thiếu sau update
```

---

*Nguồn: hermes-agent.nousresearch.com/docs (Getting Started, User Guide: CLI/TUI/Configuration/Sessions/Docker/Security) — Nous Research, MIT License.*
