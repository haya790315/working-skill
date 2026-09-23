# COMMANDS — 命令、变量、GUI 步骤

**目标**：读者从上到下依序 copy-paste 就能跑完，全程只在**一个地方**动手改。

## 1. 变量区：一览表 + 定义块

放在「前提」之后、第一个手顺之前。两者成对出现，缺一不可。

**表格**说明每个变量是什么、从哪里取得；**定义块**是拿去 copy 的东西。

| 変数 | 意味 | 取得元 | 例 |
| --- | --- | --- | --- |
| `PROJECT_ID` | GCP プロジェクト ID | Console 右上のプロジェクト選択 | `scentmatic-dev` |
| `REGION` | デプロイ先リージョン | 固定 | `asia-northeast1` |
| `SERVICE` | Cloud Run サービス名 | 固定 | `handcream-api` |

> 新しいターミナルを開いたら、下のブロックを再実行する。

```bash
# ─── ここだけ書き換える ───
export PROJECT_ID="scentmatic-dev"     # ← 要変更
export REGION="asia-northeast1"        # ← 要変更
export SERVICE="handcream-api"         # ← 要変更

# ─── 以下は変更不要 ───
export IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/app/${SERVICE}:latest"
export SA="${SERVICE}-sa@${PROJECT_ID}.iam.gserviceaccount.com"
```

规则：

- **要改的和不用改的分两段**，中间用注释线隔开。读者只需要看第一段。
- 「从哪拿」写在**表格**里，定义块的行内注释只留 `← 要変更`，不让行变长。
- 派生值（URL、镜像名、ARN、服务账号）全部在下半部拼好，后面的手顺不再拼字符串。
- 变量名用 `UPPER_SNAKE_CASE`；容易和环境既有变量撞名的加前缀（`APP_REGION`）。

## 2. 手顺里的命令块

- **零硬编码**：凡是出现在变量表里的值，一律写成 `${VAR}`。
- **每块自成一体**：不依赖上一块的 `cd` 或临时状态，读者会跳着执行。需要切目录就写 `cd "${REPO_ROOT}"`。
- **长命令一行一个 flag**，用 `\` 续行。
- **每块后面写期待输出或成功判定**，让读者能自己判断对不对。

```bash
gcloud run deploy "${SERVICE}" \
  --project="${PROJECT_ID}" \
  --region="${REGION}" \
  --image="${IMAGE}"
```

**期待する出力**

```
Service [handcream-api] revision [handcream-api-00012-abc] has been deployed
```

### 对照

```bash
# ❌ 每块都要手改，改漏一处就失败，而且找不到该改哪里
gcloud run deploy handcream-api --region asia-northeast1 --project <your-project>
gcloud run services describe handcream-api --region asia-northeast1 --project <your-project>
```

```bash
# ✅ 值已在变量区定义，这两块直接 copy
gcloud run deploy "${SERVICE}" --region="${REGION}" --project="${PROJECT_ID}"
gcloud run services describe "${SERVICE}" --region="${REGION}" --project="${PROJECT_ID}"
```

## 3. GUI・管理画面の操作

不能 copy-paste 的步骤，要写到读者不看截图也知道点哪里。

- **编号步骤，一步一个动作。**
- 写**画面名 → 标签名 → 按钮上的实际文字**，用 `→` 串起来。
- 取得的值**立刻说明填回哪个变量**，让 GUI 步骤和后面的命令块接上。

```md
### 3. LIFF ID を取得する

1. LINE Developers コンソール → 対象チャネルを開く
2. 「LIFF」タブ → 対象アプリの行
3. 表示された `LIFF ID` をコピー

→ この値を変数表の `LIFF_ID` に入れる。
```

截图只在「按钮位置真的说不清楚」时才放，并跟随项目既有的放置规则（例：`docs/assets/`）。

## 4. 变量化不了的值

一次性取得的东西（控制台下载的金钥、临时 token），单独立一个块，并写清楚**从哪拿**：

```bash
# ← 要変更：GCP Console → IAM → サービスアカウント → 鍵 でダウンロードした JSON のパス
export SA_KEY_PATH="$HOME/Downloads/xxxxx.json"
```

凭证本身绝不写进文档。

## 5. 破坏性操作

`delete` / `destroy` / `apply` / `DROP` / 覆盖既有资源的命令，前面加一行说明会发生什么：

```md
> **注意**：本番の Cloud SQL インスタンスを削除する。復旧はバックアップからのみ。
```

## 6. 非 shell 的场合

同一套原则照搬：可变值提到最前面统一定义，正文只引用。

| 对象 | 变量放哪 |
| --- | --- |
| Terraform | `variables.tf` / `terraform.tfvars`，正文写 `var.project_id` |
| SQL | 开头 `\set` 或 CTE，正文引用 |
| YAML / Kubernetes | anchor 或 kustomize 的 `configMapGenerator` |
| `.env` | 一览表说明每个键的意味与取得元，块里只留键名与要改的行标注 |
