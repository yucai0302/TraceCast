# SPEC.md

TraceCast 技术规格。本文件定义数据模型、事件 Schema、可见性状态机、API 设计、播放器行为规则。实现细节（技术栈选型等）不在此文档范围内，见 CLAUDE.md 中"需要跟人确认"部分。

---

## 1. 数据模型

### User
| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid | |
| email | string | |
| display_name | string | |
| plan | enum(`free`, `paid`) | 默认 `free`，决定 `sandbox_quota`（见商业化配额一节），仅影响沙箱托管数量，不影响过程回放/弹幕的免费使用 |
| created_at | datetime | |

### Project
| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid | |
| owner_id | uuid | FK -> User |
| name | string | |
| default_visibility | enum(`private`, `public`) | 默认 `private`，所有者一次性开启后变为 `public` |
| opted_in_at | datetime, nullable | 首次开启公开的时间，用于审计 |
| created_at | datetime | |

> 注：`sandbox_quota` 不放在 Project 上，而是由 `User.plan` 在创建/开启 Sandbox 时校验（一个用户名下同时处于 `running`/`sleeping` 状态的 Sandbox 数量不能超过其 plan 对应的配额，`free` = 1，`paid` = 可配置多个）。

### Session
| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid | |
| project_id | uuid | FK -> Project |
| started_at | datetime | |
| ended_at | datetime, nullable | |
| status | enum(`in_progress`, `success`, `failed`, `abandoned`) | |
| visibility_override | enum(`hidden`, `public`), nullable | null 表示跟随 project 的 default_visibility |
| highlight_score | float, nullable | 自动分级得分，见第 4 节 |
| event_count | int | 冗余字段，方便列表展示 |

**实际可见性计算规则**（唯一实现位置：backend 的 `visibility` 模块）：
```
effective_visibility(session):
  if session.visibility_override is not None:
      return session.visibility_override
  return session.project.default_visibility
```

### Event
| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid | |
| session_id | uuid | FK -> Session |
| seq | int | 同一 session 内严格递增，用于排序 |
| timestamp | datetime | 采集端记录的真实发生时间 |
| duration_ms | int | 该事件持续时长（用于 thinking/loading 类事件的折叠展示） |
| type | enum | 见下方"事件类型枚举" |
| summary | string | 一句话摘要，用于时间轴节点展示（自动生成或规则提取） |
| payload | json | 类型相关的详细内容，已脱敏 |
| highlight_weight | float (0.0–1.0) | 单个事件的"精彩权重"，用于计算 session 级 highlight_score 和播放器默认展开策略 |

**事件类型枚举**：
- `plan` —— Agent 规划/决策下一步做什么（叙事转折点，默认高权重）
- `tool_call` —— 调用工具（读文件/跑命令/搜索等），默认折叠展示，需要 subtype 区分工具种类
- `tool_result` —— 工具返回结果
- `thinking` —— 纯推理/等待过程，无外部可观察动作，默认压缩为时间轴细线段
- `error` —— 报错，默认高权重、视觉高亮
- `retry` —— 重试，默认高权重
- `success` —— 任务完成，默认最高权重，播放器高潮点
- `user_input` —— 用户在过程中的输入/干预

### Danmu（弹幕）
| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid | |
| session_id | uuid | FK -> Session |
| anchor_event_id | uuid | FK -> Event，弹幕锚定的事件节点，**不是绝对时间戳** |
| offset_ms_within_event | int | 在该事件内部的相对偏移，用于同一事件内多条弹幕排序 |
| user_id | uuid | FK -> User |
| text | string | 长度限制（建议 ≤ 100 字符） |
| created_at | datetime | |

### RedactionRule（脱敏规则）
| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid | |
| project_id | uuid | FK -> Project |
| pattern | string | 正则或关键词 |
| rule_type | enum(`regex`, `keyword`) | |
| created_at | datetime | |

### Sandbox（Phase 2）
一个 Project 至多对应一个 Sandbox（一对一，不是一对多），代表项目**当前最新**可运行交付物的托管状态。

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid | |
| project_id | uuid | FK -> Project，唯一约束（一个项目最多一个沙箱） |
| status | enum(`building`, `running`, `sleeping`, `error`, `stopped`) | `sleeping` 表示空闲休眠，首次访问触发唤醒 |
| deliverable_type | enum(`web_app`, `script`, `cli_tool`, `other`, `none`) | `none` 表示该项目没有可运行交付物，不展示沙箱入口 |
| last_deployed_session_id | uuid, nullable | FK -> Session，最近一次触发部署的 session，仅用于追溯，不代表沙箱有多版本并存 |
| visibility_override | enum(`hidden`, `public`), nullable | null 表示跟随 `Project.default_visibility`，与 Session 的可见性判断逻辑相互独立（见第 2 节） |
| public_url | string, nullable | 沙箱运行后的访问地址 |
| last_active_at | datetime | 用于空闲休眠判断 |
| created_at / updated_at | datetime | |

**沙箱可见性判断规则**（与 Session 判断逻辑结构相同，但字段相互独立，不要合并）：
```
effective_sandbox_visibility(project):
  if project.sandbox is None or project.sandbox.deliverable_type == 'none':
      return not_applicable
  if project.sandbox.visibility_override is not None:
      return project.sandbox.visibility_override
  return project.default_visibility
```

---

## 2. 可见性状态机

```
Project.default_visibility:  private ──(所有者一次性操作)──> public
                                                （不可逆？见下方说明）

Session.visibility_override:  null ──(所有者手动操作)──> hidden
                               hidden ──(所有者手动操作)──> null 或 public
```

- Project 从 `private` 切到 `public` 是否允许再切回 `private`：**建议允许**，但切回 `private` 后，已经公开过的历史 session 不会自动清空已产生的弹幕数据；是否连带隐藏所有历史 session，需要产品侧二次确认（默认建议：切回 private 后，所有 session 的 effective_visibility 一律视为 private，不需要逐条改 override，因为 default 已经变了）。
- `hidden` 是"从公开列表移除"，**不等于删除数据**。硬删除是完全独立的操作，需要用户在项目管理面板里二次确认，且不可逆。
- Session 处于 `in_progress` 状态时，即使 effective_visibility 为 public，也建议默认不出现在"回放"列表里（避免半成品被当作完整内容消费），可以有单独的"进行中直播"入口（MVP 阶段可以不做，留到实时直播功能）。

---

## 3. Capture-Agent（本地采集端）行为规范

1. 监听/解析 Claude Code 会话输出，归一化为上述 Event Schema。**具体接入机制需开发前查阅 Claude Code 最新官方文档确认**（可能的方向包括 hooks 机制或结构化输出解析，以文档为准，不在本 SPEC 中假定实现细节）。
2. 本地脱敏：上传前，对 `payload` 字段应用项目配置的 `RedactionRule`，命中规则的内容替换为 `[REDACTED]`。
3. 上传方式：异步队列，不阻塞用户正常使用 Claude Code 的流程；断线重传；本地保留最近 N 天未成功上传的事件缓存。
4. 采集端本身**不做任何可见性判断** —— 上传即上传到 backend，backend 根据 Project/Session 的可见性设置决定是否对外暴露。采集端只负责"内容脱敏"，不负责"是否公开"这个决策。

---

## 4. 自动分级（Highlight Score）

Session 级 `highlight_score` 由其包含的 Event 的 `highlight_weight` 聚合计算，用于公开列表的默认排序。

初版建议公式（粗糙但可解释，后续可迭代）：

```
highlight_score =
    w1 * count(type == 'error')
  + w2 * count(type == 'retry')
  + w3 * (1 if status == 'success' else 0)
  + w4 * count(type == 'plan')
  - w5 * (thinking 事件总时长占比)
```

具体权重 w1–w5 不在本 SPEC 中锁定，MVP 阶段先用一组粗略默认值上线，收集数据后再调整。目标：报错→修复的转折、决策密度高的 session 排名靠前；纯长时间空转、无转折的 session 排名靠后。

---

## 5. 播放器行为规则（Web 前端核心）

- 时间轴是**叙事时间轴**，不是物理时间轴：`thinking`/`tool_call` 中的等待部分按 `highlight_weight` 压缩显示宽度，`error`/`plan`/`success` 类节点按权重放大显示宽度。
- 默认播放策略：低权重事件（如单纯的 thinking）默认快进跳过，但保留可点击展开的入口；高权重事件默认停留展示 summary，用户可点开看 `payload` 完整细节（含 diff、完整 prompt 等）。
- 弹幕渲染：拉取以 `anchor_event_id` 分组的弹幕列表，播放头进入某个 event 时展示对应弹幕，弹幕自身不随物理时间轴滚动，而是跟随事件切换出现/消失（类似字幕轨道，而不是传统视频弹幕的连续滚动 —— 需要在实现时决定具体交互形式，此处只锁定"锚定单位是 event"这一约束）。
- 用户可选择两种细节层级（对应 README 中提到的"围观者 vs 深入学习者"差异）：
  - **精简模式**：只展示 summary + 高权重节点，弹幕密度可视化在进度条上（类似视频热力进度条）
  - **完整模式**：展示所有事件节点，包括折叠的 tool_call 细节和 payload 原文

---

## 6. 沙箱行为规范（Phase 2）

- **粒度**：Project 级，不是 Session 级、也不是访客级。一个项目最多一个 Sandbox，代表其最新交付物状态。
- **触发部署**：某个 Session 成功结束且被判定产出可运行交付物时，触发一次部署，更新 `Sandbox.last_deployed_session_id` 并刷新运行内容。旧状态直接被替换，不做多版本并存（历史过程仍可通过 Session 回放查看，但沙箱本身只反映当下）。
- **共享实例，不隔离访客**：所有围观者访问的是同一个运行中的沙箱，彼此的操作互相可见。这是产品明确选择的方向，不需要为每个访客单独开实例或做会话状态隔离。
- **生命周期**：空闲一段时间后可自动休眠（`sleeping`），首次访问时唤醒（`building` → `running`），具体休眠时长阈值留待实现时根据实际成本调整，不在本 SPEC 锁定。
- **隔离要求**：沙箱运行的是 Agent 生成的代码，需要容器级隔离、CPU/内存资源限制、出站网络限制。这一层安全要求独立于第 3 节的"采集脱敏"，两者都不能省略。
- **可见性**：跟随 `Project.default_visibility`，但支持所有者单独 override（见第 1 节 Sandbox 表定义），用于"过程不公开、成果公开"这类场景。
- **配额**：创建/唤醒 Sandbox 前需校验所有者 `User.plan` 对应的配额（`free` = 同时最多 1 个处于 `running`/`sleeping` 状态的 Sandbox，`paid` = 更高配额，具体数值留待商业化阶段确定），超出配额应返回明确错误，不做静默拒绝或降级。

## 7. Web 前端权限边界

- 所有登录用户默认是"围观者"角色：可浏览公开项目/session、可回放、可发弹幕。**不能**对任何项目/session/沙箱的管理属性（可见性、配置、部署等）做操作。
- 项目所有者在自己的项目范围内，额外拥有：切换 `default_visibility`、对单条 session 设置 `visibility_override`、对 Sandbox 设置 `visibility_override`、配置 `RedactionRule`、硬删除数据。
- 权限边界必须在 API 层强制（对应角色的 API token 不应具备越权操作的能力），前端 UI 隐藏按钮只是体验层面的辅助，不是安全边界。
- **例外（Phase 2）**：围观者可以在 Sandbox 内直接与交付物本身交互（点击、输入等），这属于"体验产品"而非"管理 TraceCast 平台"，不受上述只读约束限制，两者是不同层面的权限判断，不要合并。

---

## 8. API 设计（草案，供 backend 开发参考，非最终锁定）

```
POST   /api/events/batch              # capture-agent 批量上传事件（需 project-scoped token）
GET    /api/projects/:id               # 项目详情（含 default_visibility）
PATCH  /api/projects/:id/visibility    # 所有者操作：切换项目默认可见性
GET    /api/projects/:id/sessions      # 项目下 session 列表（所有者视角，含 hidden）
GET    /api/public/sessions            # 公开 session 列表，按 highlight_score 排序（围观者视角）
GET    /api/public/sessions/:id        # session 详情 + 事件列表（仅 effective_visibility == public）
PATCH  /api/sessions/:id/visibility    # 所有者操作：设置/清除单条 session 的 override
GET    /api/sessions/:id/danmu         # 拉取弹幕列表（按 anchor_event_id 分组）
POST   /api/sessions/:id/danmu         # 发送弹幕（围观者权限即可，需登录）
POST   /api/projects/:id/redaction-rules   # 配置脱敏规则（所有者权限）

# 以下为 Phase 2，沙箱相关
GET    /api/projects/:id/sandbox           # 沙箱当前状态（所有者/围观者通用，权限内容有差异）
POST   /api/projects/:id/sandbox/deploy    # 触发部署最新交付物（系统内部调用，session 成功结束后自动触发，不开放给用户手动调）
PATCH  /api/projects/:id/sandbox/visibility  # 所有者操作：设置/清除沙箱的 visibility_override
POST   /api/projects/:id/sandbox/wake      # 围观者访问休眠沙箱时触发唤醒
```

---

## 9. MVP 明确不做的事项（避免范围蔓延）

- 实时直播模式（`in_progress` 状态的 session 实时推送）
- 多 Agent 工具源适配（Cursor / TRAE / 其他 CLI）
- 弹幕互动激励机制（点赞、热评、等级体系）
- 精彩片段自动剪辑/分享卡片生成
- 移动端适配（MVP 先做桌面 Web）
- **沙箱托管（第 6 节）及商业化配额**：整体属于 Phase 2，MVP 阶段不需要实现 `sandbox-runtime` 组件，但数据模型（`Sandbox` 表、`User.plan`）可以提前预留字段，避免后续加表迁移成本
