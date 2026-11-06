# auto-fc-public

定时重建面板实例的**运行仓库**。

代码不在这里 —— 在私有仓库 `jyucoeng/fc-private`，运行时检出到 `fc-private/` 子目录再跑。
本仓库只放两样东西：

```
.github/workflows/fc-renew.yml     账号 ①（每天 UTC 13:33 = 北京 21:33）
.github/workflows/fc-renew-2.yml   账号 ②（同上）
.github/workflows/fc-renew-3.yml   账号 ③（同上）
fc/N/time.txt                      保活提交（防 60 天无活动被停用定时任务）
```

## 为什么拆成公开 + 私有

- 公开仓库的 Actions 分钟数不计费，定时任务也不会因为「仓库 60 天没活动」被自动停用；
- 私有仓库只存代码和 Secrets，不暴露运行记录、日志和账号信息。

## 延期 / 重建怎么切

环境变量 `FC_MODE`，三个来源优先级从高到低：

1. 手动 Run workflow 时填的 `mode`（下拉旁的输入框，留空即用下一条）
2. 仓库 Settings → Secrets and variables → Actions → **Variables** 里的 `FC_MODE`
3. 都没有就是 `delay`

| 值 | 行为 |
| --- | --- |
| `delay`（默认） | 只点面板上隐藏的「延长 72 小时」，不动机器、不做镜像 |
| `rebuild` | 老流程：备份镜像 → 退订 → 用镜像重开 → 端口挂回 |

配套的 `FC_EXTEND_KEEP_HOURS`（默认 48）：距离关机小于这么多小时才点下一次。

## 需要的 Secrets

Settings → Secrets and variables → Actions：

| Secret | 必填 | 说明 |
| --- | --- | --- |
| `PRIVATE_REPO_TOKEN` | ✅ | 能读 `fc-private` 的 GitHub token（`gh auth token` 拿到的 `gho_…` 就行） |
| `FC_USER` | ✅ | 用户名或邮箱 |
| `FC_PASS` | ✅ | 密码 |
| `FC_USER_2` / `FC_PASS_2` | 多账号时 | 第 2 个账号，见下面「多账号」 |
| `FC_TG_TOKEN` | 可选 | Telegram Bot Token，不配就不发报告 |
| `FC_TG_CHAT` | 可选 | Telegram 会话 id，多个用逗号分隔 |
| `HY2_PROXY_URL` | 可选 | 登录用的节点 URL（`hy2://` `vless://` `trojan://` `ss://` `tuic://` `anytls://`） |

Variables（可选）：

| Variable | 默认 | 说明 |
| --- | --- | --- |
| `FC_MODE` | `delay` | `delay` = 到期点「延长 72 小时」（不动实例）；`rebuild` = 备份镜像 + 退订重建 |
| `FC_INTERVAL_DAYS` | `29` | 重建周期（仅 `rebuild` 用） |
| `FC_DUE_DAYS` | `2` | 距到期几天动手（仅 `rebuild` 用） |

不配 `FC_MODE` 就是延期：每 3 天点一次「延长 72 小时」，实例和端口都不动。

## 跑法

- **定时**：每天 UTC 19:23（北京时间次日 03:23）。脚本自己判断「距到期不足 2 天」才动手，
  所以不是每天都重建，只是每天都来看一眼。
- **手动**：Actions → 🔄 FC 实例重建 ① → Run workflow，可以选阶段（auto/open/maintain）、
  强制忽略到期检查、或只体检（dry-run）。

跑完会在 Telegram 收到一条报告：

```
🎮 FC 实例重建报告

🕐 运行时间: 2026-10-25 01:07:45
📊 结果: ✅ 成功
🔧 类型: 实例重建
👤 账号: l***i                     ← 点一下跳到绑定的 TG
🏅 头衔: 同望者
🖥 实例: ct234
📦 规格: 容器2核2G
🔌 号段 12400-12419: 到期 2026-12-24 · 还能免费续 10 个月
💾 数据盘 5GB(/data): 到期 2026-12-23 · 还能免费续 10 个月

🌐 IP 信息: 185.156.53.*** (UA) [✅ 代理]
```

- **类型**：`新开实例` / `实例重建`
- **账号**：点进去是绑定的那个 TG（没绑定就是纯文本）
- **号段 / 数据盘**：到期时间 + 还剩几个月免费，免费额度用完就写月费
- 什么都没干的一轮（没到期）**不发**报告

## 多账号：一个账号一份 workflow

不加矩阵、不加脚本参数 —— **复制文件**就行，一个账号一份 workflow，各跑各的。

### 1. 加 Secret

第 2 个账号的凭据，名字加后缀 `_2`：

| Secret | 值 |
| --- | --- |
| `FC_USER_2` | 第 2 个账号的用户名（或邮箱） |
| `FC_PASS_2` | 第 2 个账号的密码 |

第 3 个就 `FC_USER_3` / `FC_PASS_3`，以此类推。
TG 和代理的 Secret（`FC_TG_TOKEN` / `FC_TG_CHAT` / `HY2_PROXY_URL`）**各账号共用**，不用复制。

### 2. 复制 workflow

```bash
cp .github/workflows/fc-renew.yml .github/workflows/fc-renew-2.yml
```

文件里所有要改的地方都标了 `①`～`⑤`，搜 `①` 逐个过一遍：

| 标号 | 位置 | 改成 |
| --- | --- | --- |
| ① | `name:` | `🔄 FC 实例重建 ②` |
| ② | `cron:` | 错开时间，见下表 |
| ③ | `FC_USER:` | `${{ secrets.FC_USER_2 }}` |
| ④ | `FC_PASS:` | `${{ secrets.FC_PASS_2 }}` |
| ⑤ | `TIME_DIR:` | `fc/2` |

`concurrency` **不用改** —— 用的是 `${{ github.workflow }}`，
复制出来的文件 `name` 不同，天然互斥，不会互相打断。

`cancel-in-progress: true`：同一个 workflow 来了新的一轮，会顶掉还在跑的那一轮。
所以手动 Run workflow 能立刻接管，也能用它当中止手段 —— 代价是正在重建的实例
会被扔在半路（可能停在「已退订、还没重开」），下一轮定时任务看到没有实例会自动补开。

### 3. 时间怎么错开

cron 是 UTC，+8 小时是北京时间：

| 账号 | `cron` | 北京时间 |
| --- | --- | --- |
| ① | `23 19 * * *` | 次日 03:23 |
| ② | `41 19 * * *` | 次日 03:41 |
| ③ | `59 19 * * *` | 次日 03:59 |

不用错开也能跑（互斥键是按文件分的，会并行），但错开对面板友好一点。

### 4. 验证

提交后 Actions 左侧会多出一个 `🔄 FC 实例重建 ②`，点进去 Run workflow，
勾 `dry_run` 跑一次，日志里应该看到：

```
入口自检：fc.py --help OK
FC_TG_TOKEN          已配置
HY2_PROXY_URL        已配置
账号：l***2
```

每个账号的 `fc/N/time.txt` 是分开的，一眼能看出哪个账号最后跑过。

## 只保活、不干活

不配 `FC_*` 凭据就行：「校验凭据」那步会打印
`没配凭据 → 本次只保活，不跑维护`，跳过安装代理和运行两个步骤，
但**照样提交 `fc/1/time.txt`**，整轮是成功的，不会每天红一次。

## 注意

- 本仓库是**唯一**跑定时任务的地方，私有仓库 `fc-private` 里不放任何 workflow。
- `fc-private` 的代码更新后不用动本仓库 —— 每次运行都是重新检出私有仓库的 `main`。
- 多账号就是复制 workflow，见上面「多账号：一个账号一份 workflow」——
  改 5 处（`name` / `cron` / `FC_USER` / `FC_PASS` / `TIME_DIR`），`concurrency` 不用动。

详细文档在私有仓库的 `README.md`。
