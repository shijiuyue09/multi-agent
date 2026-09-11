# 多 Agent 个人管理工具 — 技术规格文档（SPEC）

- 版本：v0.1
- 对应 PRD：PRD.md v0.1
- 日期：2026-07
- 状态：待评审

---

## 1. 第一性原理：本系统的三条设计公理

Spec 的每个具体决策都从这三条推导，评审任何设计分歧时回到这里裁决。

### 公理一：LLM 只负责"理解"，Python 负责"事实"

LLM 是概率系统，擅长模糊的自然语言理解，不擅长精确计算；Python 是确定性系统，反之。因此：

- 所有**理解类**工作（意图识别、时间语义理解、账目分类）→ 交给 LLM
- 所有**事实类**工作（时间运算、金额求和、存储、检索）→ 交给纯 Python 工具
- 推论：工具函数不调用 LLM；Agent 不做心算，必须调工具。金额汇总这类事，宁可工具算错（可修、可测），不可让模型心算（不可控）

### 公理二：时间的唯一真相源是"注入的当前时刻"

"明天下午三点"的含义完全取决于"现在是几点"。LLM 不知道现在几点，训练数据里的"今天"是错的。因此：

- 每次 kickoff 前，系统将**当前本地时间**（含星期几）注入任务描述
- Agent 的职责被限定为：相对时间 → 绝对时间（ISO 格式字符串）的**转换**
- 工具的职责是：校验转换结果格式合法、落在合理范围
- 推论：系统里任何地方不允许出现 `datetime.now()` 之外的"当前时间"来源

### 公理三：协作过程必须可观测、可归因

多 Agent 系统的失败模式（分派错误、循环委派）比单 Agent 更难排查。因此：

- verbose 全程开启，分派链打印到终端
- 每条数据记录携带 `created_by` 字段（哪个 Agent 写入）
- 测试集（S1–S7）即验收标准，分派错误视同功能 bug，不视为"模型偶尔会这样"

---

## 2. 数据规格

### 2.1 存储总览

- 位置：`data/` 目录，每领域一个 JSON 文件
- 编码：UTF-8，`ensure_ascii=False`，`indent=2`（人类可直接阅读）
- 顶层结构均为**数组**，元素为记录对象
- 所有 ID：UUID4 前 8 位（可读性与唯一性的平衡，原型期够用）

### 2.2 `data/events.json` — 日程

```json
[
  {
    "id": "a1b2c3d4",
    "title": "和张三开产品评审会",
    "datetime": "2026-07-22 15:00",
    "note": "",
    "created_by": "schedule_agent",
    "created_at": "2026-07-21 10:30:00"
  }
]
```

约束：`datetime` 格式严格为 `YYYY-MM-DD HH:MM`；存储按 `datetime` 升序。

### 2.3 `data/todos.json` — 待办

```json
[
  {
    "id": "e5f6g7h8",
    "content": "给房东交房租",
    "deadline": "2026-08-01",
    "done": false,
    "created_by": "todo_agent",
    "created_at": "2026-07-21 10:31:00"
  }
]
```

约束：`deadline` 可为 `null`；`done` 布尔值，完成的待办不物理删除（保留可查）。

### 2.4 `data/expenses.json` — 账目

```json
[
  {
    "id": "i9j0k1l2",
    "amount": 35.0,
    "category": "餐饮",
    "note": "午饭",
    "date": "2026-07-21",
    "created_by": "expense_agent",
    "created_at": "2026-07-21 13:05:00"
  }
]
```

约束：`amount` 为正浮点数；`category` 枚举 ∈ {餐饮, 交通, 购物, 居住, 娱乐, 其他}；`date` 格式 `YYYY-MM-DD`。

### 2.5 `data/notes.json` — 笔记

```json
[
  {
    "id": "m3n4o5p6",
    "content": "CrewAI 的 Hierarchical 模式适合这个项目的分派逻辑",
    "tags": ["agent", "架构"],
    "created_by": "note_agent",
    "created_at": "2026-07-21 14:00:00"
  }
]
```

约束：`tags` 为字符串数组，可为空。

---

## 3. 模块规格

### 3.1 `storage.py` — 存储层

纯 Python，零 LLM 依赖，唯一允许触碰 `data/` 的模块。

```python
DATA_DIR = Path(__file__).parent / "data"

def load(name: str) -> list[dict]:
    """读取 data/{name}.json，文件不存在返回空列表。"""

def save(name: str, records: list[dict]) -> None:
    """整体覆写 data/{name}.json，自动创建目录。"""

def new_id() -> str:
    """生成 UUID4 前 8 位。"""

def now_str() -> str:
    """当前本地时间 'YYYY-MM-DD HH:MM:SS'，唯一真相源（公理二）。"""
```

设计说明：原型期每次操作整体覆写。N 条记录规模下性能无虞；换来的是零并发bug、零部分写入损坏。

### 3.2 `tools.py` — 工具层

每个工具用 CrewAI 的 `@tool` 装饰器。工具描述（docstring）是写给 LLM 看的契约，**必须写明参数格式约束**。

#### 日程工具（schedule）

```python
@tool("add_event")
def add_event(title: str, datetime_str: str, note: str = "") -> str:
    """添加日程。datetime_str 必须是已解析好的绝对时间，格式 YYYY-MM-DD HH:MM。
    返回确认信息，其中复述解析后的时间。"""
    # 校验：datetime.strptime 解析失败 → 返回错误说明（不抛异常）
    # 存储后按 datetime 排序

@tool("list_events")
def list_events(range_desc: str = "本周") -> str:
    """列出日程。range_desc 取值：今天/明天/本周/全部。
    返回格式化列表，无结果时明确说"没有日程"。"""

@tool("delete_event")
def delete_event(title_keyword: str) -> str:
    """按标题关键词模糊匹配删除。匹配到 0 条 → 说明；匹配到多条 → 列出候选
    并要求用户明确（对应 PRD 开放问题2：删除需二次确认）。"""
```

#### 待办工具（todo）

```python
@tool("add_todo")
def add_todo(content: str, deadline: str = "") -> str:
    """添加待办。deadline 格式 YYYY-MM-DD，无期限传空串。"""

@tool("list_todos")
def list_todos() -> str:
    """列出未完成待办，按创建时间排序。"""

@tool("complete_todo")
def complete_todo(keyword: str) -> str:
    """按内容关键词标记完成。多条匹配 → 列候选要求澄清。"""
```

#### 记账工具（expense）

```python
VALID_CATEGORIES = ["餐饮", "交通", "购物", "居住", "娱乐", "其他"]

@tool("add_expense")
def add_expense(amount: float, category: str, note: str = "", date: str = "") -> str:
    """记一笔支出。category 必须是：餐饮/交通/购物/居住/娱乐/其他 之一。
    date 格式 YYYY-MM-DD，空串 = 今天（由工具调 now_str() 填充，不信 LLM）。
    amount 必须为正数。"""

@tool("expense_summary")
def expense_summary(month: str = "", category: str = "") -> str:
    """汇总支出。month 格式 YYYY-MM，空串 = 当月；category 空串 = 全部。
    返回：总金额 + 按分类的分项金额。求和由 Python 完成（公理一）。"""
```

#### 笔记工具（note）

```python
@tool("add_note")
def add_note(content: str, tags: str = "") -> str:
    """保存笔记。tags 为逗号分隔字符串，如 'agent,架构'，可为空。"""

@tool("search_notes")
def search_notes(keyword: str) -> str:
    """按关键词在内容和标签中搜索，返回匹配的笔记列表。"""
```

#### 工具层统一约定

1. **返回字符串，不抛异常**——工具的返回值就是 Agent 的观察结果（observation），异常会打断 Crew 执行链；错误信息写成"发生了什么 + 该怎么办"的自然语言
2. 成功返回中包含**回显**（写入了什么），供 Agent 复述给用户确认
3. 每个工具内自调 `now_str()` 填默认日期——默认时间永远由工具填，不信 LLM 传参（公理二的工程落地）

### 3.3 `agents.py` — Agent 与 Crew 定义

#### LLM 配置

```python
from crewai import LLM

deepseek = LLM(model="deepseek/deepseek-chat", temperature=0.1)
# temperature 取低值：本系统是执行型而非创作型，确定性优先
# API key 从环境变量 DEEPSEEK_API_KEY 读取（litellm 原生约定）
```

#### 专职 Agent（4 个）

通用模板：`role` = 职位名，`goal` = 职责 + 输出要求，`backstory` = **边界声明**（明确"不做什么"，防止 Manager 误派时 Agent 越权硬答）。

```python
schedule_agent = Agent(
    role="日程管家",
    goal="管理用户的日程：添加、查询、删除。所有相对时间必须结合给定的当前时间转换为绝对时间后再调用工具",
    backstory="你只负责日程。遇到记账、待办、笔记类请求，明确说明不属于你的职责范围，不要代为处理。",
    tools=[add_event, list_events, delete_event],
    llm=deepseek,
)

todo_agent = Agent(...)    # 待办专员，tools=[add_todo, list_todos, complete_todo]
expense_agent = Agent(...) # 记账专员，tools=[add_expense, expense_summary]
note_agent = Agent(...)    # 笔记专员，tools=[add_note, search_notes]
```

`expense_agent` 的 goal 中额外写明：一句输入含多笔开销时，逐笔调用 `add_expense`。

#### Manager 与 Crew

```python
crew = Crew(
    agents=[schedule_agent, todo_agent, expense_agent, note_agent],
    tasks=[main_task],            # 见 3.4
    process=Process.hierarchical,
    manager_llm=deepseek,         # 自动创建的经理 Agent 用同一模型
    verbose=True,                 # 公理三：全程可观测
)
```

Hierarchical 模式下 CrewAI 自动创建经理 Agent，负责拆解与委派；其 system prompt 无法直接定制，因此**分派边界写在每个专职 Agent 的 backstory 里**（经理能看到下属的职责描述，据此委派）。

### 3.4 任务与输入流（`main.py`）

```
启动 → 检查 DEEPSEEK_API_KEY（缺失则提示并退出）
     → 打印欢迎语与示例
     → 循环：
         读入一行（quit/exit 退出）
         构造 Task：
           description = f"当前时间：{now_str()}（{星期}）\n用户请求：{user_input}"
           expected_output = "对用户请求的完整处理结果，用中文简洁回复，复述关键信息（如解析后的时间、金额）供用户确认"
         crew.kickoff()
         打印 result
```

设计说明（公理二的落地）：当前时间在**任务描述**里注入，而非依赖 Agent 自觉——这是每一次请求的固定前置。

---

## 4. 关键流程规格

### 4.1 复合指令流程（S7 灵魂场景）

```
输入："明天下午三点开会，顺便记一笔今天午饭 35"
  │
  ├─ main.py 注入当前时间（如 2026-07-21 周二 12:00）
  │
  ▼
Manager（deepseek）
  ├─ 拆解：子任务A = 日程(会议, 2026-07-22 15:00)
  │         子任务B = 记账(35元, 餐饮, 2026-07-21)
  │
  ├─ 委派 A → schedule_agent → add_event(...) → "已添加日程：会议，2026-07-22 15:00"
  ├─ 委派 B → expense_agent  → add_expense(35, "餐饮", "午饭", ...) → "已记账：餐饮 35.0 元"
  │
  ▼
汇总返回："已为你安排：① 明天（7月22日）15:00 会议；② 记账：午饭 35 元（餐饮）。"
```

### 4.2 澄清流程（FR-1.4）

输入缺少关键信息（如"帮我记个会"无时间）→ Agent 不调工具，直接回复反问；Manager 原样转达。**规格约束：宁反问，不编造。**

### 4.3 删除二次确认流程（PRD 开放问题2）

`delete_event` 单条精确匹配 → 直接删；模糊匹配多条 → 返回候选列表，不调删除；用户在下一轮指明。工具内部不维护"待确认状态"——确认语义由对话完成，工具保持无状态。

---

## 5. 测试规格

### 5.1 单元测试（脱离 LLM，M1 验收）

| 用例 | 验证点 |
| --- | --- |
| T1 add_expense 正常/负金额/非法分类 | 参数校验与错误文案 |
| T2 expense_summary 跨月、按分类过滤 | 求和正确性（公理一的核心回归） |
| T3 add_event 非法 datetime_str | 格式校验不抛异常 |
| T4 list_events "今天/本周"边界 | 时间范围过滤正确 |
| T5 add_note + search_notes 中文关键词 | 检索命中 |
| T6 所有写操作后重 load | 持久化往返一致 |

运行方式：`python -m pytest test_tools.py`（不依赖 API key）。

### 5.2 端到端测试（依赖 DeepSeek，M3 验收）

PRD S1–S7 七个场景逐条执行，人工核对：

| 场景 | 通过标准 |
| --- | --- |
| S1–S6 | 正确 Agent 被委派；数据正确落盘；回复含回显 |
| S7 | 两个 Agent 均被委派，顺序不限；回复同时含两项确认 |

分派准确率统计口径：20 条指令（S1–S7 变体），错误 ≤ 2 条（对应 PRD ≥ 90%）。

---

## 6. 依赖与配置

```
# requirements.txt
crewai
python-dotenv
pytest        # 单元测试
```

```
# .env（不入 git，提供 .env.example）
DEEPSEEK_API_KEY=sk-xxxxxxxx
```

已知风险备案（对应 PRD 第 7 节）：Windows 安装 CrewAI 若遇 `chroma-hnswlib` 编译错误，需装 Visual Studio Build Tools；备选方案是用 `pip install crewai --no-deps` 手工裁剪依赖，届时再处理。

---

## 7. 与 PRD 的追溯表

| PRD 条目 | Spec 落点 |
| --- | --- |
| FR-1 Manager | 3.3 Hierarchical Crew + 3.4 任务构造 |
| FR-2/3/4/5 四领域 | 3.2 工具层 + 3.3 专职 Agent |
| FR-6 持久化 | 3.1 storage.py + 第 2 节数据规格 |
| FR-7 CLI | 3.4 main.py |
| NFR-1 可验证 | 5.1 单元测试 |
| NFR-3 隐私 | .env 约定 + created_by 本地留痕 |
| S1–S7 场景 | 5.2 端到端验收 |
| 开放问题1/2/3 | 固定 6 分类（3.2）/ 4.3 确认流 / 不开 memory（3.3） |
