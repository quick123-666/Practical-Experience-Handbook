# 18_ipython-kernel与AgentJev数据决策体系实战教程

> 实战日期：2026-09-26 ｜ 环境：Windows 11 ｜ 主题：给 agent 团队建立"有状态 Python 执行环境 + 本地决策引擎"的完整数据决策体系

---

## 0. 一句话总结

用 **jupyter-live-kernel（NousResearch/hermes-agent, MIT）** 方案给 agent 团队部署了一个**有状态的 IPython 内核**（变量跨调用持久、127.0.0.1:8888 常驻、计划任务自启），再把本地 **AgentJev-0.6B 决策模型**（127.0.0.1:8149 常驻）接入为"判断中枢"——最终沉淀出 **8 个数据决策点**、两个可复用工具（`tag_data.py` 三模式 + `pipeline_checks.py` 四函数），ipython-kernel 里 import 即用。

```
ipython 数据资产（CSV/JSON/审计日志/内核状态）
        │ 加工
        ▼
AgentJev 决策中枢（choice 选择 / boolean 判真 / score 分级，零解码 token）
        │ 输出
        ▼
标签落盘 / 审计追加 / 系统动作（清理·重启·报告）
```

---

## 1. 背景与目标

- **问题**：agent 团队每次执行代码都是"从零跑一遍的无状态脚本"，无法迭代探索、无法跨步骤保留变量。
- **方案调研**：GitHub 上 Anthropic 官方 skills（105k★）无 jupyter 相关项；datalayer/jupyter-ai-agents（163★）过重；最终选定 **NousResearch/hermes-agent 的 `jupyter-live-kernel` skill（170k★, MIT）**——"有状态 Python REPL，变量跨执行持久"思路最清晰。
- **第二阶段目标**：把本地 AgentJev 决策模型变成"数据判断中枢"，覆盖打标、分类、体检、校验、健康、清理等场景。

---

## 2. 环境事实（本机固定）

| 项 | 值 |
|---|---|
| 系统 Python | `C:\Users\Administrator\AppData\Local\Programs\Python\Python311\python.exe`（pip 26.2.1） |
| 内核 Python | `C:\Users\Administrator\AppData\Roaming\uv\tools\jupyterlab\Scripts\python.exe`（装包必须用这个） |
| JupyterLab | 4.6.3（uv tool install，内含 ipython 9.17.1 / ipykernel 7.3.0） |
| 操作脚本 | `C:\Users\Administrator\.agent-skills\hamelnb\skills\jupyter-live-kernel\scripts\jupyter_live_kernel.py` |
| 常驻服务 | 计划任务 `AgentJupyterServer`（登录自启 → `http://127.0.0.1:8888`） |
| 内核工作区 | `C:\Users\Administrator\notebooks\scratch.ipynb` |
| AgentJev | 计划任务 `AgentJevServer` → `http://127.0.0.1:8149`（model=AgentJev-0.6B） |
| AgentJev 源码 | `C:\Users\Administrator\Desktop\agent-jev`（GitHub: malevrigns/agent-jev） |
| 审计日志 | `C:\Users\Administrator\.agentjev\logs\decisions.jsonl`（每次调用自动记录） |

---

## 3. 第一部分：IPython 常驻内核部署

### 3.1 安装与克隆

```powershell
uv tool install jupyterlab            # 装到独立工具环境
git clone https://github.com/hamelsmu/hamelnb C:\Users\Administrator\.agent-skills\hamelnb
```

> 注意：uv tool install 写 stderr 报 exit 1 是 PowerShell 误报，实际装成功。裸 `python` 指向沙箱运行时 3.14，执行命令必须显式用 Python311 全路径。

### 3.2 常驻服务（计划任务，禁止 .bat）

```powershell
schtasks /Create /TN "AgentJupyterServer" /SC ONLOGON /RL LIMITED `
  /TR "C:\Users\Administrator\.local\bin\jupyter-lab.exe --no-browser" `
  /F
```

配置 `C:\Users\Administrator\.jupyter\jupyter_server_config.py`：

```python
c.ServerApp.ip = "127.0.0.1"
c.ServerApp.port = 8888
c.ServerApp.token = ""
c.ServerApp.disable_check_xsrf = True
c.ServerApp.notebook_dir = r"C:\Users\Administrator\notebooks"
c.ServerApp.open_browser = False
```

> 安全边界：仅绑 127.0.0.1 且无 token = "本机任何进程可访问"——只允许本地 agent 使用，不要对外开放。

### 3.3 有状态验证（全链路）

```powershell
uv run $S servers --compact                          # 发现服务
# 建 scratch.ipynb + Jupyter REST 创建内核会话
uv run $S execute --path scratch.ipynb --code "x = 41" --compact
uv run $S execute --path scratch.ipynb --code "x + 1" --compact   # → 42（变量跨调用持久）
uv run $S variables --path scratch.ipynb list --compact          # 活变量
uv run $S restart-run-all --path scratch.ipynb --save-outputs --compact  # 整本从头验证
```

多行代码：PowerShell 用反引号 `n 换行（here-string 传参会失效，已验证）。

### 3.4 封装为 skill

自建 `ipython-kernel` skill（.user_skills 下，中文/PowerShell 适配版），挂入 `service-starter\scripts\registry.json` 新增 ipythonkernel 子系统。

---

## 4. 第二部分：AgentJev 决策引擎

### 4.1 它是什么（README 一句话）

> AgentJev-0.6B **不写句子**。给它一段状态和已写好的问题，一次前向返回每个选项的概率分布，**解码 token = 0**。适合在 agent 循环里做 gate（门控）/ route（路由）/ score（评分）。

### 4.2 三种原语（核心）

| 原语 | 问什么 | 传什么 | 拿回什么 |
|---|---|---|---|
| **Boolean** | 一个命题 | 可选 true/false 准则 | `value`、为真 `probability`、两侧质量 |
| **Choice** | 给定选项选哪个 | 2–255 选项，每项描述（list/map） | `value`、`top_probability`、`margin`、完整分布 |
| **Score** | 落在有序量尺哪里 | 2–10 级描述，从低到高 | `level`（argmax 下标，**0-based**）、`score`=Σi·Pᵢ |

### 4.3 HTTP 接口

```json
POST http://127.0.0.1:8149/api/evaluate
{
  "requests": [{
    "id": "0",
    "state": "用户投诉: 支付失败三天了",
    "questions": [{"id": "intent", "type": "choice", "question": "...", "options": {"billing": "账单/退款", "bug_report": "..."}}]
  }]
}
```

- 上限：32 段状态 / **128 题** / 1024 候选路径 / 请求体 ≤1MB；上下文 **2048 tokens 超长拒绝**（不静默截断）
- 返回 `results[].answers[]`；choice **没有 `probability` 字段**（只有 top_probability/margin）——曾因误用字段 KeyError
- 失败重试 1 次、120s 超时较稳

---

## 5. 第三部分：8 个数据决策点（核心资产）

| # | 决策点 | 原语 | 工具 | 实测结果 |
|---|---|---|---|---|
| #1 | 主题打标 | Choice | `tag_data.py --mode tag` | 10 行测试 8/10 准确 |
| #2 | 审计场景分类 | Choice | `tag_data.py --mode audit` | 374 条，规则对照一致率 95.1% |
| #3 | 审计异常检测 | Boolean | `tag_data.py --mode check` | 453 条判异常 46（10.2%） |
| #4 | 审计风险分级 | Score | `tag_data.py --mode check` | L2+L3 共 21 条需关注 |
| #5 | 计算结果校验 | Boolean | `pipeline_checks.check_result` | prob 0.96 判准 |
| #6 | cell 产出质量 | Score | `pipeline_checks.assess_cell` | 空 cell 规则兜底为 0 |
| #7 | 内核健康检查 | Boolean | `pipeline_checks.check_kernel` | 判健康 prob 0.80 |
| #8 | 清理优先级 | Score | `pipeline_checks.classify_cleanup` | %TEMP% 残留→建议清理 |

### 5.1 tag_data.py（三模式）

```powershell
# tag：主题打标（自动归纳候选 → AgentJev 选择）
python tag_data.py --input 数据.csv [--column 文本列] [--out 输出.csv]

# audit：审计日志场景分类（规则层按问题 id 映射 + 模型层语义判断，双轨对照）
python tag_data.py --mode audit --input decisions.jsonl [--batch 10]

# check：审计体检（Boolean 异常 + Score 风险）
python tag_data.py --mode check --input decisions.jsonl [--batch 10]
```

**tag 模式核心算法**（自动归纳标签，无需手写规则）：
1. jieba 分词 + 词性过滤（白名单 `{n,nr,ns,nt,nz,nrt,ng,an,vn,v,l}`，滤形容词）取真词
2. 全局主题词 = 至少出现在 2 行的词（min_df=2）+ DF≤0.6 过滤通用词
3. 每行候选 = 命中全局主题词 + 至多 1 行内词（共≤3）+ `other`
4. 分批 POST /api/evaluate（choice），输出 `tag/tag_desc/tag_prob/tag_margin`

**audit 模式双轨**：`scene_rule`（按问题 id 精确映射，如 team_route→任务路由）vs `scene_ml`（AgentJev 语义判断）；`match` 列对照。未归类的 210 条中模型给 206 条判了明确场景——这是语义判断的增益。

**check 模式**：每条同时问 Boolean（是否异常）+ Score（风险 0-3），叠加规则指标（ok/answers/wall_ms）输出体检表。

### 5.2 pipeline_checks.py（四函数，内核内 import 即用）

```python
import sys
sys.path.insert(0, r'C:\Users\Administrator\DoubaoWork\chats\2026-09-26\new-chat-1')
import pipeline_checks as pc

r = pc.check_result(json.dumps({"total": 100, "rows": 4}), expectation="total 应为 100")
c = pc.assess_cell(code_text, output_text)            # level 0-3；空 cell 直接返回 0
k = pc.check_kernel('jupyter server pid 992, kernel idle')
u = pc.classify_cleanup('路径: %TEMP%/_t1.txt 12KB 5天前 临时残留')
```

---

## 6. 实测数据

### 6.1 decisions.jsonl 画像（截至 2026-09-26）

- 421KB / 360+ 条，9/24 起持续增长（每次调用自动审计）
- 问题类型：choice 234 / boolean 118 / score 76
- 调用场景：通用分类 166、记忆团队质量评估 73+73、任务路由 64
- state 形态：记忆团队指令 133、其他文本 158、JSON 对象 54、空 14

### 6.2 audit 全量（374 条）

场景分布：任务路由 223 / 运行质量评估 69 / 内容分类 36 / 健康检查 31 / 其他 10 / 数据标记 5。
规则对照：164 条可判定，156 一致（**95.1%**）；8 条冲突全部集中在异常/截断 state（`用户任务: ???`、乱码）——模型在内容不可判时保守判"其他"，行为合理。

### 6.3 check 全量（453 条）

异常 46 条（10.2%，多为早期测试/安全事件/发票等"值得关注"记录）；风险 L0 227 / L1 205 / L2 2 / L3 19。L3 中 4-6 条为常规记忆团队调用误报（wall 3-8s 偏慢触发"高危"）。**使用口径：`ml_abnormal=True` 或 `ml_risk_level≥2` 为关注信号，L3 常规调用人工复核**。

### 6.4 内核内实测（scratch.ipynb 会话）

```
KERNEL #5 -> True 0.968    # 结果校验
KERNEL #6 -> 2 1.965       # cell 质量
KERNEL #7 -> True 0.630    # 内核健康
KERNEL #8 -> 3 1.711       # 临时残留→立即清理
```

---

## 7. 踩坑经验（本会话全部 gotcha，按类别归档）

### 7.1 环境/执行类

1. **裸 `python` 是沙箱 3.14**——本机真实环境必须用 Python311 全路径；装包目标是内核 Python（uv tools 环境），不是系统 Python。
2. **uv tool install 报 exit 1 是误报**——stderr 输出导致 PowerShell 判定失败，实际装成功。
3. **PowerShell 输出经常被吞**——可靠方案：`... 2>&1 | Out-File $log -Encoding utf8` 后 Read，或 `2>&1 | Out-String`。
4. **多行代码传参**：反引号 `n 可靠；here-string 不可靠；Start-Process ArgumentList 会拆碎含空格参数——代码写文件后 exec 最稳。
5. **15 秒前台等待自动转后台任务**——用 TaskOutput -block 读；超时不代表失败，任务在后台继续。

### 7.2 AgentJev 接口类

6. **choice 返回没有 `probability` 字段**——只有 `top_probability`/`margin`，用错字段 KeyError。
7. **score 的 `level` 是 0-based 下标**——对应 legend 数组位置，统计时从 0 起。
8. **boolean 返回 Python 布尔值（True/False）不是字符串**——`== "true"` 永远 False，要用 `is True or str().lower()=="true"`。
9. **上下文 2048 tokens 超长直接拒绝**——长 state 截断到 200 字防超时（extract_state_text max_len=200）。
10. **HTTP 超时要捕获 TimeoutError/OSError**——socket 超时不是 URLError，不捕获会崩全流程；单批失败兜底 except Exception 标记 error 不中断。
11. **批量上限**：128 题 / 1024 候选路径——大批数据按 batch 20（tag）/10（audit/check）分批。

### 7.3 工具开发类

12. **元组取值错位 bug 出现两次**——(ri, value, prob) 元组取 ml[0] 拿到的是索引不是值；组装输出后必须用 CSV 回读验证列值。
13. **候选噪音治理 5 轮迭代**：2-3 字 n-gram 产生大量噪音（这个/就到/个价）→ 装 jieba 分词 → 词性过滤误伤真词（快递/退款被标 v 滤掉，改白名单含动词）→ 行内词主导改全局主题词命中 → min_df=2 + DF≤0.6 过滤。最终 10 行测试 8/10 准确。
14. **0.6B 对空输入会瞎猜**——空 cell 被判"有价值"，加规则兜底（空输入直接返回 level=0，不劳烦模型）。
15. **Edit 工具"File has not been read yet"偶发**——完整 Read 后 Write 全量重写最可靠。

### 7.4 语义/质量边界类

16. **审计日志 ≠ 待标记语料**——decisions.jsonl 的 state 是上游指令文本，主题标签不匹配（新闻文档被标 freight、评估文本被标 团队/回答）；正确做法是 audit 模式按"调用场景"打标。
17. **0.6B 置信度正常区间 0.3-0.8**——粗粒度判断可靠，L1/L2 边界、复杂语义需人工复核；需要更精细可换 headroom/MiniMax 引擎。
18. **`margin` 是集合内相对偏好**——不是动作独立成功概率；要后者对每个动作单独问 Boolean。

---

## 8. 使用建议与边界

- **分工**：ipython-kernel 负责"加工数据"（pandas/清洗/转换），AgentJev 负责"对数据做判断"（选/判/打分），tag_data/pipeline_checks 是封装好的客户端。
- **审计闭环**：每次判断自动写 decisions.jsonl，可追溯；定期跑 `--mode audit` + `--mode check` 做体检。
- **内核集成**：ipython-kernel skill 文档已含「数据标记」「流水线判断」两节，agent 团队照文档 import 即用。
- **已知边界**：0.6B 小模型粗粒度；数据量 >50 行主题统计更稳；置信度 <0.4 建议人工复核；L3 常规调用有误报。

---

## 9. 相关产物清单

| 产物 | 位置 |
|---|---|
| tag_data.py（三模式工具） | `C:\Users\Administrator\DoubaoWork\chats\2026-09-26\new-chat-1\` |
| pipeline_checks.py（四函数） | 同上 |
| decisions_audit.csv / decisions_check.csv | 同上（全量审计结果） |
| ipython-kernel skill | `.user_skills\ipython-kernel\SKILL.md` |
| hamelnb 仓库 | `C:\Users\Administrator\.agent-skills\hamelnb`（MIT） |
| AgentJev 源码 | `C:\Users\Administrator\Desktop\agent-jev`（Apache-2.0） |
| 服务配置 | `C:\Users\Administrator\.jupyter\jupyter_server_config.py` |

---
