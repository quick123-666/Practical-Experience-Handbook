# Practical Experience Handbook

> **实战经验手册** — 30 篇来自 AI / RAG / Agent / Pipeline 真实生产场景的实战教程合集。
>
> **不是教科书，是踩坑日志 + 可复用代码 + 数字基线**。
> 每篇教程都有完整的：背景 → 失败案例 → 修复依据 → 端到端验证 → 5 个踩坑 → Checklist。

---

## 一、仓库简介

本仓库收录了 Mavis / 玄针理梁 在 2026-09 期间围绕 AI 工程基础设施、RAG 知识库、Agent 编排、桌面控制 Pipeline 这 4 个主题沉淀下来的实战经验。每篇教程都来自一个真实跑通（或真实跑失败后修复）的工程任务，**不是 demo**。

**核心主题**：

| 主题 | 教程数 | 关键场景 |
|---|---|---|
| **Skill / Agent 体系** | 3 | Skill 体系实战、Agent 架构设计、AI Agent 工程师速成 |
| **RAG 知识库** | 7 | 多跳 RAG、召回偏差修复、幻觉防御、KB 导入运营、增量更新、闭环实战 |
| **Codex / Harness 工程实践** | 3 | Codex 三层模型、Harness 工程、Memory v2 hot rule |
| **OCR / 代码评审** | 2 | 阿里 Token Plan + OCR、Mavis 接管 code review |
| **桌面控制 Pipeline** | 2 | Laya 贪吃蛇思维 + computer_control_pipeline 多 backend + 几何过滤 |
| **AI 基础与学习路径** | 2 | AI 基础概念、AI 工程基础设施 |
| **视频 / 语音自动化** | 2 | 视频转 md 教程、语音对话循环 + TTS 自动播放 |
| **大模型架构与面试** | 3 | 分布式系统设计、推理优化、面试突围、幻觉治理 |
| **实战报告 / 综合** | 3 | 企业级 RAG 报告、生产级多跳 RAG、Agent Bootstrap |

---

## 二、教程目录

### Skill / Agent 体系

- **[`01_Skill体系实战教程.md`](01_Skill体系实战教程.md)** — 30+ Skill 的目录、命名约定、trigger 设计、调用链
- **[`02_Agent架构与实战教程.md`](02_Agent架构与实战教程.md)** — 多 Agent 编排 + Compact & Resume + SkillClaw 5 阶段蒸馏
- **[`Agent_Bootstrap_教程.md`](Agent_Bootstrap_教程.md)** — 一句话 idea → 生产级 AI Agent 工程
- **[`AI Agent工程师速成教程_从Demo到生产级数字员工_图文教程.md`](AI_Agent工程师速成教程_从Demo到生产级数字员工_图文教程.md)** — 从 demo 到上线的 7 阶段
- **[`AI_Agent架构设计实战教程.md`](AI_Agent架构设计实战教程.md)** — 44KB 的 Agent 架构权威指南

### RAG 知识库

- **[`04_RAG与知识库架构教程.md`](04_RAG与知识库架构教程.md)** — 20KB RAG 系统设计权威指南
- **[`07_RAG召回偏差5层修复实战教程.md`](07_RAG召回偏差5层修复实战教程.md)** — 召回偏差的 5 层递进修复
- **[`08_RAG幻觉防御与可观测性实战教程.md`](08_RAG幻觉防御与可观测性实战教程.md)** — v1.4 幻觉防御层
- **[`09_RAG防以为导入与KB运营实战教程.md`](09_RAG防以为导入与KB运营实战教程.md)** — KB 导入标准流程 + recall 验证 + 踩坑
- **[`2026-09-20_RAG增量更新实战教程.md`](2026-09-20_RAG增量更新实战教程.md)** — 增量 vs 全量 rebuild trade-off
- **[`2026-09-20_RAG实战闭环与TTS切换总览.md`](2026-09-20_RAG实战闭环与TTS切换总览.md)** — 闭环总览
- **[`2026-09-20_RAG技术栈完整报告_top3召回验证.md`](2026-09-20_RAG技术栈完整报告_top3召回验证.md)** — 技术栈选型报告
- **[`RAG系统设计实战教程.md`](RAG系统设计实战教程.md)** — 52KB RAG 系统权威指南
- **[`生产级多跳RAG系统实战教程.md`](生产级多跳RAG系统实战教程.md)** — 50KB 生产级权威
- **[`实战报告_如何做企业级RAG_2026-09-16.md`](实战报告_如何做企业级RAG_2026-09-16.md)** — 实战报告

### Codex / Harness 工程实践

- **[`03_Codex与Harness工程实践教程.md`](03_Codex与Harness工程实践教程.md)** — Codex 三层模型 + Harness 工程
- **[`14_Memory_v2_hot_rule自约束失效与三层防御实战教程.md`](14_Memory_v2_hot_rule自约束失效与三层防御实战教程.md)** — 48KB Memory v2 hot rule + 三层防御
- **[`15_LLM硬约束与一致性实战教程.md`](15_LLM硬约束与一致性实战教程.md)** — 27KB LLM 硬约束

### OCR / 代码评审

- **[`12_阿里Token_Plan_OCR代码评审实战教程.md`](12_阿里Token_Plan_OCR代码评审实战教程.md)** — 阿里 Token Plan + OCR code review
- **[`13_RAG_OCR_代码评审集成实战教程.md`](13_RAG_OCR_代码评审集成实战教程.md)** — RAG + OCR code review 集成

### 桌面控制 Pipeline

- **[`16_Laya贪吃蛇思维控制电脑实战教程.md`](16_Laya贪吃蛇思维控制电脑实战教程.md)** — Laya + snake pipeline + PAI DSW 实例创建 0→1
- **[`17_computer_control_pipeline多Backend+几何过滤实战教程.md`](17_computer_control_pipeline多Backend+几何过滤实战教程.md)** — 4 backend (laya/jev/agentjev/auto) + 几何过滤 + bilibili 真实跑通 BV1xwWn6FEiH

### AI 基础与学习路径

- **[`05_AI基础概念与学习路径教程.md`](05_AI基础概念与学习路径教程.md)** — AI 基础概念 + 学习路径
- **[`06_AI工程基础设施教程.md`](06_AI工程基础设施教程.md)** — AI 工程基础设施

### 视频 / 语音自动化

- **[`10_视频转md教程_v10关键截图版实战.md`](10_视频转md教程_v10关键截图版实战.md)** — 视频 → markdown 教程 + 关键截图
- **[`11_语音对话循环_TTS自动化播放实战教程.md`](11_语音对话循环_TTS自动化播放实战教程.md)** — 语音对话循环 + edge-tts 自动播放

### 大模型架构与面试

- **[`分布式系统设计全栈实战教程.md`](分布式系统设计全栈实战教程.md)** — 54KB 分布式系统权威
- **[`大模型推理优化实战教程.md`](大模型推理优化实战教程.md)** — 推理优化权威
- **[`大模型幻觉治理实战教程.md`](大模型幻觉治理实战教程.md)** — 幻觉治理权威
- **[`大模型面试突围实战教程.md`](大模型面试突围实战教程.md)** — 38KB 面试突围

---

## 三、用法

### 1. 直接读（按主题）

每个教程独立可读，按需求查目录：

```bash
# 找 RAG 召回相关
ls | grep -i "召回\|RAG"

# 找桌面控制相关
ls | grep -i "Pipeline\|Laya\|贪吃蛇"

# 找 LLM 防御 / Memory 相关
ls | grep -i "Memory\|硬约束\|幻觉"
```

### 2. GitHub 在线浏览

每篇教程都是 Markdown 格式，可直接在 GitHub 网页查看：
[https://github.com/quick123-666/Practical-Experience-Handbook](https://github.com/quick123-666/Practical-Experience-Handbook)

### 3. 用 RAG 检索（如果有 local RAG）

教程都是结构化 Markdown（标题层级清晰、有表格、有代码块），适合作为 RAG KB 直接索引。

```bash
# 推荐索引命令（如果有本地 RAG 工程）
python scripts/import_doc.py *.md --kb practical_handbook --embedder bge-m3
```

---

## 四、教程通用结构

每篇教程都按以下 13 章节固定结构写（方便跨教程对照）：

1. **实战目标 + 落地日期 + 适用** — 一句话讲清楚这页能干什么
2. **缘起** — 为什么做这个，做之前 / 失败案例
3. **整体架构** — mermaid 流程图
4. **核心组件清单** — 表格列出所有依赖 + 路径
5. **安装与准备** — 一步一步命令
6. **核心实现** — 完整可 copy 代码
7. **端到端 demo / 测试验证** — 真实跑通截图 + 数字基线
8. **关键数字基线** — latency / conf / hit rate 表
9. **N 个核心踩坑** — 实操必看
10. **为什么 X 比 Y 好** — 决策依据 + 实验对比
11. **复现 checklist** — 一步一步验证
12. **与现有方案的对比** — trade-off 表
13. **参考文件路径 / 一句话总结**

---

## 五、注意事项

1. **无敏感信息**：所有 API key 都是 `sk-xxxxxxxxxxxxxxxx` 占位符，不会有真 key 泄漏
2. **可执行**：所有命令都在 Windows + PowerShell 5.1 + Python 3.11 + CN-ISP 环境实测过
3. **LF vs CRLF**：教程统一 LF 换行，Git 在 Windows 上会自动转 CRLF（warning 不影响）
4. **中英混排**：教程主体中文，专有名词保留英文（Laya / RAG / OCR / KB 等）

---

## 六、版本与作者

- **作者**: Mavis Agent（AI 编排助手） + 玄针理梁（用户主导）
- **建立日期**: 2026-09-24
- **教程总数**: 30 篇
- **总字数**: ~1.5 MB（17,189 行 commit）
- **License**: 跟随用户偏好（默认 MIT）

---

## 七、一句话总结

**30 篇实战教程合集 — 不是教科书，是踩坑日志 + 可复用代码 + 数字基线，每篇都从真实失败案例出发，给出修复依据 + 端到端验证 + 复现 checklist，覆盖 AI 工程基础设施 / RAG 知识库 / Agent 编排 / 桌面控制 Pipeline 4 个主题。**