# 早期临床试验剂量递增安全决策平台

面向首次人体 / 剂量递增试验的安全决策系统。每个队列**冻结**给药方案、观察窗和受试者集合；
安全事件按严重度和可评估状态进入**版本化规则引擎**；只有**五道安全门**全部通过、
且**独立安全委员达到门槛**，才能递增（escalate）、扩展（expand）或停止（stop）。

零第三方依赖：Python 3.11 标准库（sqlite3 / http.server / hmac / hashlib）。

## 核心安全不变量

| 要求 | 实现机制 |
|---|---|
| 队列冻结给药方案/观察窗/受试者集合 | `freeze_cohort` 后剂量、窗不可改；受试者集合以 `revision` 编号，替补产生新修订 |
| 安全事件按严重度+可评估状态进入引擎 | AE 须经裁定（assessed + evaluable），规则门槛 `min_severity_dlt`（默认 ≥ grade3）才计 DLT |
| 观察窗完整 + 缺失数据处理完毕 + 委员门槛 | 五道安全门：窗口完整 / 缺失数据清零 / 事件全裁定 / 可评估集合完整 / ≥2 名在职独立委员 |
| **迟到 AE、替补、规则修订不得拼接快照** | 每次决策锚定单快照 `content_hash`；实质性事件（AE/裁定/替补/给药）使旧快照 `superseded`；规则集不可变、快照绑定 `ruleset_version`，修订永不重算历史快照 |
| **两名委员并发投票不得拼接** | 投票携带 `expected_version`（乐观并发）+ 服务级锁 + DB 层计数 CAS；后到者得到 409，必须刷新视图重试 |
| **断网草稿不得写入陈旧决定、不得重复成票** | 草稿存委员本机（动作+快照指纹+规则版本）；恢复后服务端先核对快照仍有效（迟到事件/替补/规则换版 → 记 `expired` 并 409 阻止写入，冲突原因入队列），有效才按实时投票同一套校验一次性落库；`draft_key` 幂等，重复提交只回放首次结果 |
| **紧急暂停立即阻断后续给药、保留已给药事实** | 暂停事务提交即生效（支持队列级/全试验级）；阻断的给药尝试以 `held_back=1` 留痕；`administrations` 表只追加，已给药行永不修改/删除 |
| 决策可证明、可审计 | 哈希链事件日志（seq/prev_hash/hash）+ 快照内容哈希 + 委员 HMAC 签名；决策证明含 11 项独立校验 |

规则约束：递增/扩展类决定不得超越引擎建议（`engine_disagreement`）；保守的 `stop` 永远允许；
暂停生效期间只接受 `stop` / `continue` 投票。

## 运行

```bash
# 启动（自带 Web 安全工作台 + REST API）
python3 -m app.server --db data/platform.db --port 8080
# 打开 http://127.0.0.1:8080 ，点「载入演示数据」

# 命令行一键演示全链路（含并发投票与紧急暂停）
python3 -m app.cli_demo

# 测试
python3 -m unittest tests.test_platform -v
```

## 决策流程

```
建试验 → 建队列(draft) → 入组 planned_n 人 → 冻结(方案/窗/集合)
  → 登记给药(只允许冻结剂量) → AE 上报/裁定 → 缺失数据登记/处理
  → [观察窗结束] → 构建快照（跑规则引擎+五门，计算 content_hash）
  → ≥2 名独立委员 HMAC 签名投票（expected_version 乐观并发）
  → 签发决定（校验：同一快照哈希/门槛/引擎/暂停/签名/哈希链）
  → escalate/expand: 关闭本队列并派生子队列草案；stop: 队列停止
```

## REST API 摘要

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/trials` `/api/cohorts` | 建试验/队列 |
| POST | `/api/cohorts/{id}/enroll` `/freeze` | 入组 / 冻结 |
| POST | `/api/cohorts/{id}/administrations` | 登记给药（暂停时返回 409 且留痕） |
| POST | `/api/cohorts/{id}/aes` `/api/aes/{id}/adjudicate` | AE 上报（首快照后自动标 late）/ 裁定 |
| POST | `/api/cohorts/{id}/pending` `/api/pending/{id}/resolve` | 缺失数据登记/处理（可标 imputed） |
| POST | `/api/subjects/{id}/replace` | 受试者替补（revision+1，旧快照失效） |
| POST | `/api/rules` | 发布规则集新版本（不可变，旧快照不受影响） |
| POST | `/api/cohorts/{id}/holds` `/api/holds/{id}/release` | 紧急暂停 / 解除 |
| POST | `/api/cohorts/{id}/snapshots` | 构建快照（引擎+五门+content_hash） |
| POST | `/api/snapshots/{id}/votes` | 委员投票（member_id/action/signature/expected_version） |
| POST | `/api/drafts` | 断网草稿一次性提交（draft_key 幂等；快照被推进则 409 `draft_expired` 并记冲突原因） |
| GET | `/api/drafts?cohort_id=` | 草稿队列：submitted/expired 终态 + conflict_code/conflict_reason |
| POST | `/api/snapshots/{id}/decisions` | 签发决定并生成决策证明 |
| GET | `/api/decisions/{id}` `/verify` | 决定与证明 / 11 项独立校验 |
| GET | `/api/state` `/api/journal` | 全局状态 / 哈希链事件日志 |
| POST | `/api/clock` | 推进模拟时钟（确定性演示/测试） |
| POST | `/api/sign` | 演示用：委员 HMAC 本地签名端点 |

## 断网投票草稿

委员网络中断时，在快照详情点「存为断网草稿」：决定动作 + 快照指纹（content_hash）+
规则版本以 `draft_key` 为幂等键存入本机 localStorage（断网不丢，工作台有「模拟断网」开关可演练）。
恢复后点「提交草稿」：服务端在锁内**先核对快照仍有效**——被迟到事件/替补标记 `superseded`、
出现新实质事件、指纹不符、规则集换版，任一成立即把草稿记为 `expired`（冲突原因留痕）并 409 阻止写入；
仍有效则复用实时投票的全部校验（五门/暂停/签名/版本 CAS），投票与草稿在同一事务落库。
同一 `draft_key` 重复提交只回放首次结果（已提交给回原 vote_id，已过期给回原冲突原因），绝不产生第二张票。
工作台草稿队列展示 **待提交 / 已提交 / 过期** 三态；`GET /api/drafts` 返回冲突原因；
`GET /api/state` 的 `vote_drafts` 字段供多终端合并。草稿事件（`vote_draft_submitted` /
`vote_draft_expired`）进入哈希链，实时投票与决策证明校验不受影响。

## 决策证明（11 项校验）

`GET /api/decisions/{id}/verify` 独立重算：证明哈希未改、快照哈希锚定、快照载荷可重算、
快照未失效、五门全过、**规则引擎按绑定版本重算结论一致**、全部投票 HMAC 签名有效、
**每张票锚定同一快照哈希**、法定人数满足（≥2 票且 ≥2 独立）、事件哈希链完整、
动作未超越引擎。任一项失败即 `verified=false`。

## 代码结构

- `app/store.py` — SQLite schema、哈希链追加 `append_event`、`verify_chain`
- `app/services.py` — 领域服务：冻结/给药/暂停/AE/缺失/替补/规则/快照/投票/断网草稿/决定/证明
- `app/rules.py` — 版本化规则规格与引擎（严重度→DLT→bands→建议）
- `app/crypto.py` — 规范 JSON / SHA-256 / HMAC
- `app/clock.py` — 可注入时钟（观察窗判定唯一时间来源）
- `app/server.py` + `app/web/index.html` — REST API 与安全工作台
- `tests/test_platform.py` — 31 个不变量测试
