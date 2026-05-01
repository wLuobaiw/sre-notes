# 相关网页

[**skills编写指南 - agentskills.io**](https://agentskills.io/home)



# Skills 是什么？



## AI 的成长

1. **大模型（大脑）**

   此时 AI 什么都知道，但是**什么都做不了**

2. **对话能力（嘴巴）**

   此时 AI 可以和用户对话，但是**只能指挥**，不能实际上手帮用户做

3. **Tools（双手）**

   加入了工具的调用，现在可以动手操作，但是不同 AI 的调用不同，每种 AI 都要定制一个工具（m*n爆炸）

4. **MCP（USB）**

   提供给大模型一个统一的调用接口（AI 界的 USB），大大提高 Tools 的接入速度（m+n即可全互联），工具生态爆炸式增长

5. **Skills（肌肉记忆、技巧）**

   “能做” ≠ “做的好”

   **以弹吉他举例**：给 AI 一把吉他 = Tool，教会 AI 弹吉他 = Skill



## Skills 的发展

1. **Prompt（提示词）复用**

   大家把好用的 Prompt 存成文件，下次直接使用 —— 这是 Skills 最早的雏形

2. **可复用指令**

   如 OpenAI 的 `GPTs`，Cursor 的 `.cursorrules`，给 AI 预设身份和规则更方便，但是还是纯文本

3. **从文本到文件夹（关键转折）**

   Anthropic 在 Claude Code 中引入 Skills —— 从“一段文本”变成“一个文件夹”

   入口文件 + 脚本 + 模板 + 参考文档 = **完整能力包**

4. **生态扩展**

   OpenClaw 等开源项目打破平台壁垒，其他 Agent 借鉴和支持类似 Skills 机制



---

---



# Skills 的定义



## **定义：Skills = 模块化的能力扩展单元**

每个 Skills 打包了三样东西：

- **元数据**（Metadata）

  ```markdown
  name: 名字
  description: 功能描述
  ```

  告诉 Aengt 这个 Skill 叫什么，用来做什么

- **指令**（Instructions）

  Skill 的核心，写在 **`SKILL.md`** 中，告诉 AI 怎么做

- **资源**（Resources）

  脚本 `scripts/`，模板 `templates/`，参考文档 `refs/`



## **典型 Skill 文件夹结构**

```markdown
my-skills/
   ├── SKILL.md		# 入口
   ├── scripts/		# 可执行脚本
   |	└── ...
   ├── templates/	# 模板文件
   |	└── ...
   ├── references/	# 参考文档
   |	└── ...
   └── ...
```



## 五步工作流

**发现 --> 匹配 --> 加载 --> 执行 --> Hooks（可选）**

**核心设计哲学**：按需加载、渐进式披露、模型自主决策



### 发现（Discovery）

**Agent 启动做的第一件事**：

- 扫描 Skills 目录
  - `.codebuddy/skills/`（项目级）
  - `~/.codebuddy/skills/`（用户级）
- 找到每个 `SKILL.md`



> **关键：**
>
> - **不读正文**，只提取 name + description
>
>   读取20个 Skill ≈ 几百 Token，对上下文窗口**几乎无影响**
>
> - **设计哲学：**
>
>   最小化启动成本 —— 不读正文，不加载资源，只提取少量信息简历索引



### 匹配（Matching）

在用户提问后，Agent 拿着技能清单做语义匹配，将用户请求与每个 Skill 的 description 对比，判断哪个 Skill 最适合处理当前任务



> **此处不是关键词匹配，而是模型推理**
>
> 可以理解为：description 是简历，Agent 是面试官



**好的 description**：明确场景 + 触发短语 + 能力边界



### 加载（Loading）

1. 读取 `SKILL.md` 完整正文

2. 按需加载资源文件

   如：正文说 “需要 API 详情” 时，去读 `references/api.md`

3. Agent 自主决定读哪些文件



> **三大好处**：
>
> - **节省 Token**：几万行内容，单次可能只读 2-3 个文件
> - **上下文清晰**：不被无关信息干扰
> - **支持大型 Skill**：资源可以很丰富，不用担心溢出



### 执行（Execution）

**纯指令型**：
按指令直接生成内容
如：代码审查，逐项检查并输出报告

**脚本辅助型**：
调用scripts/获取数据，再分析输出
如：性能分析，先跑脚本收集数据

**模板驱动型：**
读取模板，填充内容，结构化输出
如：周报生成，按模板格式填写

**混合型**：
以上模式的组合
大多数好的 Skill 都是混合型，质量最高



### Hooks —— 特定时机自动触发

**Hooks = 主动介入**
普通Skill：用户提问→被动触发
Hooks：Agent 执行到特定时机 → 自动触发 Skill 逻辑

**示例：安全守卫Skill**
注册 PreToolUse Hook
Agent 每次调用工具前自动检查
检测到危险操作 → 阻止执行

**四种 Hook 类型**：

- **PreToolUse**

  调用工具之前：安全检查、阻止危险操作

- **PostToolUse**

  调用工具之后；日志记录、结果校验

- **OnError**

  执行出错时；自动重试、错误上报

- **OnStart**

  Skill 被激活时；初始化检查、环境验证



> **Hooks 让 Skills 从 “被动的知识包” 升级为 “主动的守护者”** —— 不需要用户触发，自动运行



---

---



# Skills 的三条核心原则



## 简洁至上

上下文窗口是公共资源每个 token都有成本



**上下文窗口是公共资源**
你的Skill和以下内容共享同一个窗口：

- System Prompt（系统提示）
- 对话历史
- 其他 Skills 的元数据
- 用户的实际请求



**Token 成本分阶段**

- 启动时

  只加载 metadata，成本极低

- 触发后（关键）

  `SKILL.md` 完整读入

  每个 token 都在争抢空间

- 按需加载

  额外资源只在需要时读取



> **你的 Skill多占1个 token = 其他内容少1个token的空间**
>
> **灵魂拷问：**这段话值它占的token吗？



## 匹配自由度

任务脆弱性决定指令精确度自由度需要匹配任务特性



**关键决策：给AIAgent多大的自由空间？**

- 给太多自由：AI可能走错方向，尤其在脆弱操作上
- 给太少自由：AI被束缚，无法发挥推理能力
- 关键是匹配：根据任务的脆弱性和可变性，给出匹配的精确度



**高自由**（文本指引/启发式）

- 适用：多种方案都对
- 风格：给方向，信任 AI
- 例子：Code Review

**中自由**（伪代码／带参数的脚本）

- 适用：有推荐模式
- 风格：给模板，允许调整
- 例子：报告生成

**低自由**（精确脚本/不可修改）

- 适用：操作脆弱易出错
- 风格：精确命令，禁止修改
- 例子：数据库迁移



> 脆弱性：做错后果越严重 → 自由度越低
> 可变性：每次最佳做法越不同 → 自由度越高
>
> **灵魂拷问：**这是窄桥还是旷野？



## 多模型测试

效果取决于底层模型不同模型需要不同策略



**为什么要多模型测试？**

Skills是模型的扩展，效果取决于底层模型

- 为Opus写的简洁指令
  Haiku可能理解不了

- 为Haiku写的详细指令
  Opus可能被干扰



> **对国内开发者更重要**
>
> ​	模型更多样（Claude+国产模型）
> ​	不同模型对Skill理解差异更大
>
> **灵魂拷问：**在所有目标模型上都测过了吗？



---

---



# Skills 结构规范



## YAML Frontmatter

这是 Skill 的 YAML 头部标识，用于 Agent 启动时的发现阶段

**两个必填字段**

- **name** 一 Skill 的名字
- **description** 一 Skill 的描述

**Agent启动时只扫描这两个字段**



**字段约束**

**name 的限制**

- 最多64字符
- **命名规范**：只能小写字母+数字+连字符
- 不能含XML标签
- 不能用保留词

**description 的限制**

- 不能为空
- 最多1024字符
- 不能含XML 标签



## Description：三大编写规则

**Description = 发现的唯一依据**

Al Agent 需要从 100+个 Skills 中选对一个，它只能靠description来判断

### 规则一：必须用第三人称

"Processes Excel files..."**（可以）**

“I can help you..."**（不行）**

"You can use this to..."**（不行）**



### 规则二：What + When
**What**：核心能力是什么

**When**：什么时候该触发

Description 承担**双重职责**



### 规则三：具体关键词
想想用户会怎么描述需求
把这些词都放进description

例：对于一个 excel 分析 Skill

应该覆盖`Excel`、`spreadsheets`、tabular` data`、`xlsx`



## 渐进式披露

**核心思想**

- 不要把所有内容塞进 `SKILL.md`

- `SKILL.md` < 500行，超出就拆

- 概览+导航，具体内容放独立文件

- Al Agent 只在需要时才读取

**用多少加载多少，类似函数式编程的思想**

### 两条关键规则

`SKILL.md` 是导航地图，不是百科全书。它指路，不堆料。

#### 规则一：避免深层嵌套

Al Agent遇到嵌套引用(A→B→C）时，可能只用 `head -100` 预览，信息不完整

- （**不可取**）`SKILL.md` → `advanced.md` → `details.md` → 信息

- （**正确**）`SKILL.md` 直接链接所有文件，**保持一层深度**

#### 规则二：长文件加目录

超过 100 行 → 顶部加 ToC

```markdown
# API Reference
## Contents
- Authentication
- Core methods
- Advanced features
```

Al Agent看到目录后可以决定读全文还是跳到特定章节