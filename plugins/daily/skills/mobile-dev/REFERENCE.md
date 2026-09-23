# mobile-dev — 参考资料

## 启动环境

电脑端跑起 Remote Control server，手机才连得上。

```bash
rc
```

repo 内任何位置都可以，自动往上找 git root，session 名是 `rc-<repo 目录名>`。
启动后按**空白键**出 QR code，手机扫码连入。

| 动作 | 指令 |
|---|---|
| 启动 / 接回 | `rc` |
| 指定目录 | `rc <目录>` |
| 离开，server 继续跑 | `Ctrl-b` 放开再按 `d` |
| 看有哪些在跑 | `tmux ls` |
| 收掉 | `rcq`，或 `rcq <名字>` |

关掉 Claude 桌面 app、VS Code、终端机视窗都不影响 server。

### 省电

```bash
sudo pmset -a displaysleep 2
```

### 换机器时重建

```bash
brew install tmux
```

`~/.zshrc` 加：

```zsh
_rc_root() { git rev-parse --show-toplevel 2>/dev/null || print -r -- "$PWD" }

rc() {
  local dir name cmd
  if [[ -n "$1" ]]; then dir="${1:A}"; else dir="${$(_rc_root):A}"; fi
  [[ -d "$dir" ]] || { print -u2 "rc: 目录不存在: $dir"; return 1 }
  name="${dir:t}"
  cmd="caffeinate -is claude remote-control --name $name --spawn same-dir"
  if [[ -n "$TMUX" ]]; then
    tmux new -A -d -s "rc-$name" -c "$dir" "$cmd" && tmux switch-client -t "rc-$name"
  else
    tmux new -A -s "rc-$name" -c "$dir" "$cmd"
  fi
}

rcq() {
  local name="${${1:-${$(_rc_root):t}}#rc-}"
  if tmux kill-session -t "rc-$name" 2>/dev/null; then
    print "已收掉 rc-$name"
  else
    print -u2 "rcq: 没有 rc-$name。现有的: ${$(tmux ls 2>/dev/null | cut -d: -f1):-（无）}"
  fi
}
```

首次在一个新专案跑 `rc` 会问 spawn mode，选 `1`（same-dir）。

## 怎么知道这个 session 是不是 Remote Control 的

**查过了，没有现成的标志位。**下面是走过的死路，别再走一遍：

| 试过的 | 结果 |
|---|---|
| `claude remote-control --help` 找注入选项 | 只有 `--name` `--spawn` `--permission-mode` `--capacity` 等，**没有** `--append-system-prompt` |
| `~/.claude/bridge-spawn/*/append-system-prompt.txt` | 那是 **cloud session**（claude.ai/code 的云端容器）的注入，跟本机 Remote Control 是两回事 |
| transcript 的 `entrypoint` 字段 | 有值（`sdk-cli` / `claude-vscode` / `claude-desktop` / `cli`），但那是**写进日志的**元数据，跑的时候读不到 |
| `CLAUDE_REMOTE_CONTROL_SESSION_NAME_PREFIX` | 真实存在，但只管自动命名的前缀，**默认不设**，靠它判断会漏 |

**能用的只有进程链**，脚本在 SKILL.md。2026-09-23 实测，`rc` 经 tmux 起的 session 也查得到：

```
[0] claude ... --print --sdk-url ...                      ← spawn 出来的 session
[1] claude remote-control --name tumiki --spawn same-dir  ← server
```

### 如果哪天想要「必定触发」

纯 skill 侧做不到确定性 —— description 再 pushy 也只是提高概率。要 100%，得走这条：

1. `~/.zshrc` 的 `rc()` 里给 server 挂个环境变量（spawn 的子 session 会继承）：
   `cmd="caffeinate -is env CLAUDE_MOBILE=1 claude remote-control --name $name --spawn same-dir"`
2. `~/.claude/settings.json` 加 SessionStart hook，读到 `CLAUDE_MOBILE=1` 就往
   `hookSpecificOutput.additionalContext` 里写一句「本 session 用 mobile-dev」

**2026-09-23 评估后没做**，因为要动 `~/.zshrc` 和全局 settings 两个文件，换机器就得重配一次，
而 description + 进程自检已经够用。想改主意时按上面两步走。

## AskUserQuestion 的硬约束

SKILL.md 的点选规约全建在这个工具上。它的限制是**硬的**，设计选项前先对一遍，
不然调用会直接被拒：

| 项 | 上限 |
|---|---|
| 一次几个问题 | 1–4（手机上 2 个就够，4 个是压迫） |
| 每个问题几个选项 | 2–4（这就是回复只能切 2–4 段的原因） |
| `header` | **12 个字符**，会显示成小标签 |
| `label` | 1–5 个词 |
| `description` | 一两句，写「选了会怎样」 |
| 自由输入 | **系统自动加「Other」**，不要自己造 |
| 多选 | `multiSelect: true`，问「哪几段要展开」时必开 |

推荐的选项放第一个，label 末尾写「（推荐）」—— 手机上他多半直接点第一个。

调用长这样：

```json
{"questions": [{
  "question": "哪一段要展开？",
  "header": "深入",
  "multiSelect": true,
  "options": [
    {"label": "①现在的样子", "description": "把登录链路每一跳讲清楚"},
    {"label": "②卡在哪",     "description": "贴掉线时的日志和触发条件"}
  ]}]}
```

## Plane MCP 工具与调用成本

connector 名 `RAYVEN Plane OAuth`，endpoint `https://tasks.rayven.cloud/mcp`，workspace 固定为 `rayven`。
共 9 个工具：`plane_cycle` `plane_intake` `plane_label` `plane_me` `plane_member`
`plane_module` `plane_project` `plane_state` `plane_workitem`。

| 调用 | 返回大小 | 手机上能不能用 |
|---|---|---|
| `plane_workitem` `search` | 每条 6 个字段 | ✅ 开工首选 |
| `plane_workitem` `retrieve` | 含完整 `description_html`，常见 3000+ 字 | ⚠️ 必须摘要后再输出 |
| `plane_workitem` `list` | 上面那个 × per_page | ❌ 不要用它开场 |

### 票号 → UUID 的坑

`retrieve` / `update` 要的是 **UUID**，但人讲的是 `TUMIKI-53`。
票号对应的是 `sequence_id` 字段，**不是** `id`。唯一便宜的换算方式是 `search`：

```
plane_workitem action=search query="TUMIKI-53"
→ {"name":"[DEV-2511] 導入状況を新UIで出す",
   "id":"d72dd241-ffac-4ff9-bd11-9fa32a156177",
   "sequence_id":53, "project__identifier":"TUMIKI",
   "project_id":"2f37cc79-0f9c-4efd-bd3f-4df106fa78dd",
   "workspace__slug":"rayven"}
```

### 能力边界

- **没有 comment 工具。** 无法在票上留言回报进度。要回报只能 `plane_workitem update` 改
  `state` 或 `description_html` —— 而这两个都在红线内，必须先问用户。
- `delete` / `archive` 没有开放（这是好事，不用担心误删）。
- `plane_state` 需要 `project_id` 才能列状态。

## Plane project ↔ 本机 repo

**权威做法是实时查，不要信下面的表。** Plane 的 project `description` 里通常直接写了 GitHub URL：

```
plane_project action=retrieve project_id=<id>   → description 里抓 github.com/<owner>/<repo>
git remote get-url origin                        → 比对
```

下表只是 2026-09-21 实测的便利快照，Plane 开了新项目就会过期：

| Plane project | identifier | 本机 |
|---|---|---|
| Tumiki | `TUMIKI` | `~/work/tumiki` |
| イプラ様｜トライアングルエヒメ | `IPURA` | `~/work/ipla` |
| KOKUSHIRU×純真大学様 | `KOKUSHIRU` | 未 clone |
| PSC_Amazon自動化 | `PSCAMAZON` | 未 clone |
| RAYVEN 社内運用 | `OPS` | description 无 repo，要问 |
| SANYO / TAE / SALES / HIRE | | 非代码项目 |

两个要记住的例外：

- **`~/work/ipla` 的 remote 是 `triangle-ipla`** —— 目录名和 repo 名对不上，靠目录名猜必错。
- **`~/work/rayven-k3s-infra` 和 `~/work/scentmatic/*` 在 Plane 里没有对应 project。**
  这些 repo 上的工作不要硬套票号流程，直接问用户要做什么。

## tumiki 的 app 与端口

`~/work/tumiki` 是 pnpm + turbo monorepo。截图前先确认目标 app 在 `.claude/launch.json` 里。

| app | 端口 | dev 命令 |
|---|---|---|
| `internal-manager` | 3100 | `pnpm --filter @tumiki/internal-manager dev` |
| `lp` | 3200 | `pnpm --filter @tumiki/lp dev` |
| `tenant-console` | 3200 ⚠️ 与 lp 冲突 | `pnpm --filter @tumiki/tenant-console dev` |
| `manager` | 3000（默认） | `pnpm --filter @tumiki/manager dev` |
| `desktop-v2` | vite 默认 | `pnpm --filter @tumiki/desktop-v2 dev` |

`.claude/launch.json` 现成只有 `lp` 和 `lp-prod`。要截 `internal-manager` 的图，先补一段：

```json
{
  "name": "internal-manager",
  "runtimeExecutable": "pnpm",
  "runtimeArgs": ["--filter", "@tumiki/internal-manager", "dev"],
  "port": 3100
}
```

`tenant-console` 和 `lp` 都写 3200，两个不能同时起。要同时看就给其中一个改端口。

## Plane 授权为什么会过期

`https://tasks.rayven.cloud/mcp` 走的是 **Cloudflare Access Managed OAuth**，不是自建 OAuth server：

```
authorization_server:  https://rayven122.cloudflareaccess.com
grant_types_supported: ["authorization_code", "refresh_token"]
```

`refresh_token` 是支援的，所以**不是 refresh 轮替坏掉**。grant 绑在 Cloudflare Access 的
application session 上，session 一过期，refresh token 就再也换不到新的 access token，只能重跑互动式授权。

**永久修法（全在 dashboard，跟这个 skill 无关）：**

1. Zero Trust → Access → Applications → `tasks.rayven.cloud` → Session Duration 拉长（预设常是 24h）
2. 检查 Settings → Authentication 的 global session duration 没有更短
3. 检查该 application 的 policy 没有勾 "Require re-authentication every N"
4. policy 若要求 WARP / device posture，WARP 断线也会失效

注意 `rayven-k3s-infra` 的 `docs/cloudflare-access.md` 规定「例外を追加するときは、理由と認証主体を
この文書または対象 runbook に記録する」—— 拉长 session duration 属于放宽，照规范要留记录。

另外 Vexa 的 MCP（`mtg.rayven.cloud`）明确写着不用 Cloudflare Access Managed OAuth，Plane 这边却用了。
两套 MCP 认证架构不一致，是独立于本 skill 的技术债。

## 为什么不 curl Plane API 绕过去

`https://plane.rayven.cloud/api/v1` 挡在 Cloudflare Access 后面，非浏览器 client 直接 HTTP 000。
私网 NodePort（WARP 连线下的 `http://10.11.0.41:30386`）确实通，但那条路要另外配 PAT，
而且 `docs/cloudflare-access.md` 明确禁止对 `/api/*` 开 Bypass policy。
**授权失效时不要自己想办法绕，按 SKILL.md 的早退流程停下来问人。**
