# 03 技术架构

## 1. 总体架构

```
┌──────────────── 企业微信 ────────────────┐
│ 工作台 → 自建应用 → H5 页面（OAuth 免登）   │
└───────────────┬──────────────────────────┘
                │ HTTPS（可信域名）
┌───────────────▼──────────────────────────┐
│ FastAPI 服务                              │
│  ├─ /auth      企业微信 OAuth 换取用户身份   │
│  ├─ /scenarios 场景列表与详情               │
│  ├─ /sessions  创建会话 / 发送消息 / 结束    │
│  ├─ /speech    ASR 语音转文字 / TTS 合成     │
│  └─ /reports   评分报告 / 历史记录           │
│                                           │
│  LangGraph 陪练图（会话状态 checkpoint）     │
│  模型封装层（OpenAI 兼容，统计 token/成本）   │
└───────┬───────────────┬──────────────┬────┘
        │               │              │
   大模型 API       ASR / TTS        存储
 (厂商待定)      (厂商待定)     SQLite→PostgreSQL
```

## 2. 交付形式选择

| 形式 | 结论 | 说明 |
|---|---|---|
| 自建应用 + H5 | **主入口** | 体验最好，可做语音、报告、排行榜 |
| 自建应用消息回调 | 不作为主入口 | 需 5 秒内响应，大模型须异步主动推送，且只能纯文字 |
| 企业微信智能机器人 | 复赛补充 | 用于"每日一练"推送等轻量场景，能力以官方文档为准 |

## 3. LangGraph 设计

### 3.1 状态（State）

```python
class PracticeState(TypedDict):
    session_id: str
    user_id: str
    scenario_id: str
    persona: dict               # 客户画像（来自场景 YAML）
    messages: list              # 对话历史
    trust: int                  # 客户信任值 0~100，初始值由场景决定
    open_objections: list[str]  # 尚未被化解的隐藏异议
    resolved_objections: list[str]
    turn: int
    hints_used: int
    inner_thoughts: list[dict]  # 每轮客户内心独白（报告用）
    status: Literal["ongoing", "deal", "hangup", "max_turns", "user_end"]
    report: dict | None
```

### 3.2 节点与流程

```
START → init_scenario
          ↓
      ┌─ wait_user_input（interrupt，等待员工发言）
      │     ↓
      │  evaluate_turn   评估员工本轮发言：信任值增减、是否化解异议（结构化输出，低温度）
      │     ↓
      │  customer_reply  客户 Agent 依据画像 + 当前信任值 + 未解决异议生成回复（较高温度）
      │     ↓
      └─ check_end ──未结束──┘
            │ 结束
            ↓
        coach_report     教练 Agent 按评分表生成报告（结构化输出，低温度）
            ↓
           END

旁路：hint 节点（员工点"求助"时调用，不推进轮次，hints_used += 1）
```

### 3.3 关键设计

- **先评估、后回复**：客户的态度由 `evaluate_turn` 更新后的信任值决定，避免客户 Agent 自行"被说服"。
- **结束条件**：信任值 ≥ 成交阈值且无未解决关键异议 → `deal`；信任值 ≤ 挂断阈值 → `hangup`；轮数达上限 → `max_turns`。
- **会话持久化**：使用 LangGraph checkpointer（初期 SQLite，后续 PostgreSQL），支持断线续练。
- **提示词管理**：提示词模板放 `backend/app/prompts/`，场景差异通过 YAML 注入，不复制提示词。
- **成本统计**：模型封装层记录每次调用的 token 数与估算费用，落库到会话，用于报告和比赛汇报。

## 4. 模型使用策略

| 用途 | 温度 | 输出 | 说明 |
|---|---|---|---|
| 客户回复 | 0.7~0.9 | 文本 | 求真实、有个性 |
| 轮次评估 | 0~0.2 | JSON（结构化） | 求稳定 |
| 教练报告 | 0~0.3 | JSON（结构化） | 求稳定、可复现 |
| 求助提示 | 0.3~0.5 | 短文本 | 简短可用 |

可按用途配置不同模型（例如客户回复用便宜快速的模型，教练报告用能力更强的模型）。

## 5. 语音方案

- **MVP**：按住说话 → 上传音频 → ASR 转文字 → 走 LangGraph → 客户回复 TTS → 前端播放。
- **进阶**（复赛）：实时语音模型，接近真实通话体验；需重新设计与 LangGraph 状态机的配合方式。

## 6. 企业微信接入前置条件（需 IT/管理员配合）

- [ ] 管理员创建自建应用，获得 CorpID、AgentID、Secret
- [ ] 配置可信域名（需备案且主体与企业相关）
- [ ] 配置企业可信 IP（服务器出口 IP）
- [ ] 公网 HTTPS 访问与证书
- [ ] 设置应用可见范围（试点人员 → 全员）

## 7. 部署

- 初期：单台服务器，Docker Compose（FastAPI + 数据库 + Nginx）。
- 配置通过环境变量 / `.env` 注入，密钥不入库。
