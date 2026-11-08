# AGENTS.md

本文件为 AI 编码助手（及新加入的同事）提供项目上下文。修改项目方向时请同步更新本文件和 `doc/` 下的对应文档。

## 项目一句话

**AI 客户陪练**：在企业微信里打开即用的销售/客服话术陪练工具——AI 扮演"难缠客户"与员工对练，练完由 AI 教练打分、点评并给出优秀话术示范，用于降低新人带教成本、提升一线沟通能力。

参赛背景：中瑞数科《"AI赋能·人人会用"比赛活动通知》（NO. C-HR-2026/11/01），初赛作品截止 **2026-11-25 24:00**。

## 文档索引

| 文档 | 内容 |
|---|---|
| [doc/README.md](doc/README.md) | 文档目录 |
| [doc/01-overview.md](doc/01-overview.md) | 项目背景、目标、比赛要求与评审维度、价值主张 |
| [doc/02-product.md](doc/02-product.md) | 用户角色、核心功能、用户流程、MVP 范围 |
| [doc/03-architecture.md](doc/03-architecture.md) | 技术架构、LangGraph 流程设计、企业微信接入、部署 |
| [doc/04-scenario-and-scoring.md](doc/04-scenario-and-scoring.md) | 场景配置规范（YAML）、客户画像、评分表 |
| [doc/05-data-and-security.md](doc/05-data-and-security.md) | 参考项目/客户信息接入规划、脱敏与保密规则 |
| [doc/06-milestones-and-delivery.md](doc/06-milestones-and-delivery.md) | 里程碑、比赛交付物、视频脚本、成本测算 |
| [doc/07-open-questions.md](doc/07-open-questions.md) | 待决事项清单 |

## 技术栈（规划）

- 后端：Python 3.11+、FastAPI、LangChain、LangGraph
- 模型：通过 OpenAI 兼容接口接入，具体厂商待定（候选：DeepSeek / 通义千问 / 豆包 / 公司私有部署），见 `doc/07-open-questions.md`
- 语音：ASR + TTS（候选：阿里云 Paraformer/CosyVoice、火山引擎豆包语音）
- 前端：企业微信自建应用内嵌 H5（移动端优先），OAuth 免登
- 配置：场景、客户画像、评分表均为 YAML，业务人员可自行维护

## 目录结构（规划，尚未创建的标注为 planned）

```
hackerzr2026/
├── AGENTS.md
├── README.md
├── doc/                  # 项目文档
├── backend/              # planned: FastAPI + LangGraph 服务
│   ├── app/
│   │   ├── api/          # HTTP 接口（会话、语音、报告、企业微信回调）
│   │   ├── graph/        # LangGraph 状态、节点、图定义
│   │   ├── prompts/      # 提示词模板
│   │   ├── services/     # 模型、ASR/TTS、企业微信 SDK 封装
│   │   └── core/         # 配置、日志
│   └── tests/
├── scenarios/            # planned: 场景 YAML 配置
├── frontend/             # planned: H5 对练页面
└── data/
    └── private/          # 本地敏感数据（已加入 .gitignore，禁止提交）
```

## 开发约定

- 场景/画像/评分表等**业务内容一律放 YAML 配置**，不要硬编码在 Python 里，保证可复制性（评审维度之一）。
- "客户"角色与"教练"评分角色**分离为不同节点/Agent**：客户 Agent 温度偏高求真实，教练 Agent 温度低 + 结构化输出求稳定。
- 客户 Agent 必须受隐藏状态（信任值、未解决异议）约束，避免"三句话被说服"。
- 模型调用统一经过一个封装层，便于切换厂商与统计 token 成本（成本数据是比赛汇报的重点）。
- 密钥只放 `.env`（已在 .gitignore），不得写入代码、文档或日志。
- 文档与注释使用中文；代码标识符使用英文。

## 数据安全红线（必须遵守）

- **禁止**将真实客户名称、联系方式、合同金额、通话录音等提交到仓库；真实数据只能放 `data/private/`。
- 进入提示词、知识库、演示视频的数据必须先脱敏，规则见 `doc/05-data-and-security.md`。
- 比赛通知明确要求：作品中不得包含涉密信息、客户数据或未经授权的第三方内容。

## 常用命令

尚未初始化代码工程。创建后在此补充：安装依赖、启动服务、运行测试、lint 等命令。
