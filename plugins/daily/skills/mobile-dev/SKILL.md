---
name: mobile-dev
description: >
  【手机远程开发】从手机通过 Remote Control 驱动电脑端 Claude 做开发时的行为契约。
  回复压到手机读得完的长度；问答类的长回复切成小节，结尾用 AskUserQuestion
  把「要深入哪一段」做成可点选项（手机上回滚去引用某一段再追问极其难操作）；
  开工先用 Plane 票号核对 repo；UI 改动强制截图（先切 mobile 视口）；
  不可逆动作一律先问；Plane 授权失效立刻早退不重试。
  Use when 这个 session 是 Remote Control 驱动的（`claude remote-control` spawn 出来的，
  用户人在手机端）——**哪怕用户没说自己在手机上也要用**：漏用的代价是把一屏 diff
  糊到对方脸上，误用的代价只是桌面端回复短了一点，两者不对称。
  也 use when 用户说自己在手机上 / 在外面 / 通勤中，只丢一个票号就要开工，
  要求看 UI 截图，或要你整理现况、给方案、做调查而他在小屏幕上读。
  触发词："/daily:mobile-dev"/"手机模式"/"我在手机上"/"从手机开发"/
  "remote control"/"rc"/"TUMIKI-<数字>"。
---

# mobile-dev — 手机远程开发

对方在**手机小屏幕上、通勤或走路中**读你的输出：看不了长 diff、读不了桌面宽度的截图、
来不及审查你做了什么，**而且打字很贵**。下面每条规则都从这两条推出来。

**本 skill 只读 Plane。** 票标题里的 `[DEV-xxxx]` 是 Linear 侧的镜像编号，只作对照，不去查 Linear
（手机端没有 Linear connector，写成要查它就是写一个跑不动的步骤）。

## 先确认：这个 session 是不是手机驱动的

`claude remote-control` 不会往 system prompt 里注入任何标记，所以你**看不到**现成的标志位。
想确认就自己查 —— 你的进程祖先链上有没有那台 server：

```bash
pat="remote""-control"                      # 必须拆开写，见下
p=$(ps -o ppid= -p $$ 2>/dev/null | tr -d ' ')   # 从父进程起，跳过自己
while [ "${p:-1}" -gt 1 ]; do
  case "$(ps -o command= -p $p 2>/dev/null)" in *"$pat"*) echo MOBILE; break;; esac
  p=$(ps -o ppid= -p $p 2>/dev/null | tr -d ' ')
done
```

**那两处别改回去。**`ps -o command=` 会读到你正在跑的这行脚本本身，字面量写
`*remote-control*` 就会自己匹配自己，从第 0 层直接返回 MOBILE —— 一个永远说是的
检测器比没有更糟，它会让你以为自己确认过了。拆写字符串加跳过自身，两个一起才绕得开。

命中长这样（`rc` 用 tmux 起也照样查得到，tmux 不会挡住这条链）：

```
[0] claude ... --print --sdk-url ...                     ← 你
[1] claude remote-control --name tumiki --spawn same-dir  ← server
```

**没命中也不等于安全。**从 claude.ai/code 网页端连进来的路径没验证过，进程树可能不长这样。
**拿不准就按手机模式走**：它在桌面端的副作用只是回复短一点、多几个可点选项；
反过来在手机上按桌面模式回复，对方要在 6 寸屏上滚三屏 diff —— 代价完全不对称。

## Step 1 — 开工三件事（每次必跑，不可跳过）

### 1. 解析票号

`plane_workitem` `action: "search"`，`query` 填票号原文（如 `TUMIKI-53`）。返回只有 6 个字段，极轻。

**绝不用 `list` 开场** —— 一次列 10 张票就是三万 token 的 `description_html`，手机上什么都读不到。
只有用户明确要看内容时才 `retrieve`，且必须摘成 3 条以内，**永不原样输出 `description_html`**。

### 2. 核对 repo（对不上就停）

- `plane_project` `action: "retrieve"` → 从 `description` 里找 `github.com/<owner>/<repo>`
- `git remote get-url origin` 比对
- **对不上 → 停下报告，等指示。不要自己 `change_directory`** —— 中途换 cwd 会让整个 session 的上下文错位
- Plane project 的 description 里没有 GitHub URL（如 `OPS`）→ 直接问用户，不要猜

### 3. 一行确认，等回复

```
TUMIKI-53「[DEV-2511] 導入状況を新UIで出す」
~/work/tumiki  分支 internal-manager-setup-tumiki-53 ✅
开始？
```

分支名含票号（`tumiki-53`）就标 ✅；对不上要明说，不要默默继续。

这一步的「等回复」也适用下面的点选规约 —— 与其让对方打「开始」，不如给
`开始` / `先看票的内容` / `换个分支` 三个选项。

## 输出规约

- **先给结论，3 行以内。**细节等对方问
- 不贴超过 20 行的 diff → 改成「改了 N 个文件：<清单>」
- 不贴完整 log → 只贴失败的那几行
- 表格最多 3 列
- 文件路径写成 markdown 链接，手机上可点

## 问答类回复：把「深入哪里」做成可点的

### 为什么要这么做

在手机上追问一份长回复，对方要：滚回去找那一段 → 长按选中 → 复制 → 粘贴 → 再打字说明想问什么。
五步，站在月台上根本做不了。结果就是他放弃追问，拿着一份只看懂一半的回复走人。

而点一下选项是零成本的。所以**把回复里每一段的入口，预先做成选项**，让他用点的代替打的。

Claude Code 默认告诉你 `AskUserQuestion` 只在「卡住、需要用户拍板」时才用。
**在这个 skill 里不适用那条。**这里用它不是因为你卡住了，是因为它是手机端唯一便宜的输入通道。
你不是在推卸决定，你是在给对方的下一个问题装一个不用打字的入口。

### 什么算问答类

标准是：**这份回复是让对方思考和决定的，还是汇报你已经做完的动作。**

| 要给点选入口 | 不用给 |
|---|---|
| 整理现况、盘点某块代码 | 「改完了，3 个文件」 |
| 给方案、比较几种做法 | 「测试过了，12 个全绿」 |
| 调查结果、排查原因 | 「push 上去了，PR #2091」 |
| 解释一段代码怎么跑的 | 报错、早退（照原流程直接说） |
| 回答「现在什么情况」 | 一两行就说完的事 |

执行类保持现在的简短汇报 —— 那种回复本来就没有「哪一段要深入」的问题，硬加选项只会变吵。

### 怎么做

**1. 回复切成 2-4 个带标签的小节。**

上限是 4，因为 `AskUserQuestion` 一个问题最多放 4 个选项，切太碎就装不进去。
这个限制跟手机阅读的舒适区正好吻合，不用跟它对抗。

小节标签要**短、具体、能单独看懂** —— 它等下要当选项用，对方看选项时已经不在看正文了。
「② 为什么慢」可以，「② 分析」不行。

**2. 结尾调一次 `AskUserQuestion`。**

```
问题：「哪一段要展开？」   header：「深入」   multiSelect: true
选项 = 各小节的标签
```

- **`multiSelect: true`** —— 让他一次勾两三段，省掉好几轮往返
- 每个选项的 `description` 写**选了会发生什么**，不是把小节内容再抄一遍。
  他刚读完正文，重复只是占屏幕
- **自由输入框是自动有的**（「Other」），不用自己造一个选项去模拟。
  但也别指望他用 —— 选项设计得好，他就不用打字了
- **留一格给「我这次没查的」。**选项不必每格都对应正文里的一段。
  你知道自己漏了什么 —— 没翻的目录、没验证的分支、口头带过的假设 ——
  把它也做成一个选项（`API 路由谁在挡 → 去翻 /api/* 各自的认证写法`）。
  他在手机上看不出你漏了什么，这一格就是你主动招供的地方；
  比他信了你的回复、三天后才撞上那个盲区要好

**3. 需要的话，再加第二个问题问方向。**

比如整理完现况，除了「哪段深入」再问一个「接下来做什么」：`开始改` / `再查一下` / `先不动`。
一次最多 4 个问题，但**2 个通常就够了** —— 手机上一次跳出 4 个问题也是一种压迫。

### 例子

回复正文（每节 3-5 行）：

```
① 现在的样子 —— 登录走 NextAuth，session 存 Redis sidecar
② 卡在哪 —— sidecar 跟 app 同 pod，重启就掉线
③ 两条路 —— 换 PG session store，或把 Redis 拆成独立 service
```

结尾：

```
问题：「哪一段要展开？」  header「深入」  multiSelect: true
  · ①现在的样子  → 把登录链路每一跳讲清楚
  · ②卡在哪      → 贴掉线时的日志和触发条件
  · ③两条路      → 两个方案各自的改动量和风险
问题：「接下来？」        header「下一步」
  · 先不动，我想想
  · 挑一条开始改
  · 再查点别的
```

### 别这样

| 反例 | 为什么坏 |
|---|---|
| 「都清楚了吗？」是/否 | 没信息量。答「否」之后他还是得打字说哪里不清楚 |
| 选项写成「选项 A / 选项 B」 | 脱离正文就看不懂了，等于逼他滚回去对照 |
| 回复只有 3 行还追问 | 短回复直接问就好，加选项是添乱 |
| 6 个小节硬塞进 4 个选项 | 先把回复合并成 4 段。装不下是回复太碎的信号 |
| 造一个「其他（请输入）」选项 | 系统自带「Other」，重复了还占掉一格 |

## UI 改动必须截图

**触发条件（命中任一就必截，不得自行判断"这次看不出差别"）：**

`*.tsx` `*.jsx` `*.vue` `*.svelte` `*.css` `*.scss` `**/components/**` `**/app/**` `**/pages/**` `**/ui/**`

1. 确认 `.claude/launch.json` 里有该 app 的配置 —— **没有就先补上**（端口见 [REFERENCE.md](REFERENCE.md)），不要因此放弃
2. `preview_start` `name=<app>`
3. **`resize_window` `preset="mobile"`** —— 必做。对方在手机上看，桌面宽度的图缩下去读不到字
4. `navigate` 到改动的那个画面
5. `computer` `action="screenshot"`
6. 一句话说明图里看到什么

**跑不起来要明说原因，不准静默跳过，不准说"应该没问题"。**

## 动作红线 —— 先问，等回复才做

| 自由做 | 先问 |
|---|---|
| 读文件、改 app 代码 | `git push` / `gh pr create` / `gh pr merge` |
| `test` / `lint` / `typecheck` | `plane_workitem update`（改 state 或 description） |
| 跑 dev server、截图 | 动 `infra/` `terraform/` `docker/` `.github/workflows/` |
| `git commit`（本地） | `kubectl` / `argocd` / `infisical` |
|  | `rm -rf` / `git reset --hard` / `git clean` |
|  | 装依赖（会动 lockfile） |

关卡放在**会流出去、不可逆**的动作上，不是放在改文件上 —— 对方在手机上来不及审 diff。

这里的「先问」同样用 `AskUserQuestion`，而且**要把后果写进选项的 description**
（`推上去后 CI 会跑 20 分钟` / `这会改掉票的状态，组里都看得到`）。
手机上他看不到你要推什么，选项里的那句话就是他唯一的判断依据。

## Plane 授权失效 → 强制早退

任一 `plane_*` 工具返回 401 / `invalid_token` / needs_auth：

1. **立刻停。**不重试，不换别的 plane 工具试，不 curl `tasks.rayven.cloud`（Cloudflare Access 会挡）
2. 一行报告：`Plane 授权过期，需要到 Connectors 重新登录 RAYVEN Plane OAuth`
3. 问：`要我先用你贴的票内容继续吗？`（给选项：`贴内容继续` / `我去重新授权` / `今天先算了`）
4. 等回复

根因与永久修法见 [REFERENCE.md](REFERENCE.md)。
