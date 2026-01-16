---
layout: post
title: "The Bitter Lesson for Agentic AI: Why Simple Scales and Clever Breaks"
date: 2026-01-03 15:00:00 +0800
categories: [blog]
comments: true
tags: [Agentic AI, Agent Engineering, MCP, Prompt Engineer, Context ngineer]
---

> **📖 Language / 语言选择**
>
> [🇬🇧 English Version](#english-version) | [🇨🇳 中文版本](#中文版本)

---

## English Version

When I started building my first agentic AI system, I thought I understood Richard Sutton's "The Bitter Lesson". I'd read it. I'd nodded along. I'd even cited it in discussions about the triumph of scale over hand-crafted features.

But I didn't *really* understand it until my carefully engineered agent crashed in production for the third time that week.

## What Sutton Actually Said

Let me recap Sutton's 2019 essay for those unfamiliar. His core argument:

> "The biggest lesson that can be read from 70 years of AI research is that general methods that leverage computation are ultimately the most effective, and by a large margin."

He provided compelling examples:
- **Computer chess**: Deep Blue's brute-force search crushed human chess knowledge
- **Computer Go**: AlphaGo's self-play learning demolished hand-crafted features
- **Speech recognition**: Statistical methods beat phonetic expertise
- **Computer vision**: Deep learning overtook carefully designed features

The pattern was consistent and humbling: **researchers kept trying to inject human knowledge into systems, and kept being outperformed by general methods that scaled with computation.**

The lesson was "bitter" because it meant abandoning intellectually satisfying work—elegant representations, domain expertise, clever heuristics—in favor of what seemed almost... dumb. Just throw more compute at it. Just let it learn.

## My Misunderstanding

Here's what I thought Sutton meant when I started building agentic AI:

*"Use big models, don't overthink it, scale solves everything."*

So I approached my first agent with confidence:
- Start with GPT-4 (big model ✓)
- Let it figure things out (general methods ✓)
- Don't over-engineer (avoid human knowledge ✓)

Then reality hit.

## The First Bitter Lesson: Simplicity ≠ Simplistic

My initial agent architecture was actually quite sophisticated:
- Complex state machine for conversation flow
- Elaborate prompt templates with 15+ examples
- Sophisticated retry logic with exponential backoff
- Carefully designed tool selection heuristics
- Multi-stage validation pipelines

It was elegant. It was clever. It was broken.

The agent would:
- Get stuck in states my state machine didn't anticipate
- Ignore my carefully crafted examples and hallucinate tools
- Retry the same failing action because my logic didn't account for certain error types
- Choose the wrong tool because my heuristics encoded my assumptions, not reality

**What actually worked:**

```python
# Instead of complex state machines
system_prompt = """You are an assistant that helps users accomplish tasks.
You have access to tools. Think step by step about what to do next."""

# Instead of 15 examples
# Just 2-3 clear, diverse examples

# Instead of sophisticated retry logic  
max_retries = 3  # Simple counter
if "error" in result:
    retry_with_error_context(result)

# Instead of tool selection heuristics
# Let the model choose based on clear tool descriptions
```

This felt too simple. Too naive. But it **worked reliably** while my clever version kept breaking.

**Sutton's lesson applying:** Don't encode your assumptions about how the agent *should* behave. Give it clear tools and let the model's general reasoning figure it out.

## The Second Bitter Lesson: Emergence Over Engineering

I built a research agent that needed to:
1. Search for information
2. Read multiple sources
3. Synthesize findings
4. Fact-check claims
5. Generate a report

My engineered approach:
```python
def research_workflow(query):
    # Step 1: Search strategy planning
    search_plan = plan_search_strategy(query)
    
    # Step 2: Execute searches
    sources = execute_search_plan(search_plan)
    
    # Step 3: Relevance filtering
    relevant = filter_by_relevance(sources, query)
    
    # Step 4: Synthesis pipeline
    synthesized = synthesis_pipeline(relevant)
    
    # Step 5: Fact-check validation
    validated = fact_check(synthesized)
    
    # Step 6: Report generation
    return generate_report(validated)
```

Each step had sub-functions, error handling, validation logic. It was a beautiful architecture diagram.

**What actually worked:**

```python
def research_workflow(query):
    result = agent.run(
        task=f"Research this topic and provide a comprehensive report: {query}",
        tools=[search, read_url, python_repl]
    )
    return result
```

That's it. The agent would:
- Search multiple times naturally as it discovered knowledge gaps
- Read sources it found relevant
- Re-search when it needed fact-checking
- Synthesize organically through its reasoning process

The sophisticated workflow I engineered? It was me encoding my human process. The agent's natural reasoning flow was often better—sometimes it fact-checked *before* full synthesis, sometimes it searched in different orders based on what it learned.

**Sutton's lesson applying:** Let general reasoning emerge. Don't hard-code your human algorithm.

## The Third Bitter Lesson: More Context Beats Clever Compression

I obsessed over prompt efficiency. Every token mattered. I built:
- Dynamic example selection (pick the 3 most relevant from a bank of 50)
- Compressed tool descriptions (abbreviate, remove redundancy)
- Conversation summarization (condense history every 5 turns)

This felt smart. Efficient. LLMs have context limits, right?

**Problems:**
- Dynamic examples sometimes picked edge cases that confused the model
- Compressed descriptions lost critical details (what does "proc" mean again?)
- Summarization occasionally dropped context that became crucial later

**What actually worked:**

Just include the full context:
- All tool descriptions (even if verbose)
- Full conversation history (until hitting actual limits)
- Complete examples (don't abbreviate)

When I did hit context limits, the solution wasn't clever compression—it was **better task decomposition** (split into subtasks) or **offloading to retrieval** (RAG for long-term memory).

**Sutton's lesson applying:** With large context windows, don't prematurely optimize. Let the model process full information. Scaling context is more effective than compressing cleverly.

## The Fourth Bitter Lesson: Iteration > Perfect First-Try

My engineering instinct: **make it work perfectly the first time.**

So I built:
- Elaborate input validation
- Pre-flight checks before tool calls
- Predictive error prevention
- Confidence scoring for outputs

**Reality:** The agent still made mistakes. But now it couldn't recover because my validation prevented certain valid paths, and my pre-checks added latency.

**What actually worked:**

```python
# Simple loop
for attempt in range(max_attempts):
    result = agent.step()
    
    if is_successful(result):
        return result
    
    # Give the agent its error and let it try again
    context.add_error(result.error)
```

The agent got *really good* at:
- Recognizing its mistakes
- Adjusting its approach
- Learning from error messages
- Trying alternative strategies

My perfect-first-try engineering? It prevented the agent from developing this resilience.

**Sutton's lesson applying:** Agents learn through trial and error. Don't prevent failures—enable recovery. Iteration beats prediction.

## The Fifth Bitter Lesson: Data Quality > Prompt Wizardry

I spent weeks perfecting prompts:
- A/B testing phrasing
- Optimizing instruction order
- Fine-tuning tone and style
- Crafting the perfect system message

Meanwhile, my tool descriptions were inconsistent, examples had errors, and documentation was unclear.

**The breakthrough:** I spent two days cleaning up:
- Tool descriptions (clear, consistent format)
- Examples (accurate, diverse, debugged)
- Error messages (informative, actionable)

My mediocre prompts with clean data **vastly outperformed** my optimized prompts with messy data.

**Sutton's lesson applying:** Clean, abundant data beats clever prompting. Every hour spent on prompt engineering should probably be two hours on data quality.

## What This Means for Building Agentic AI

Sutton's bitter lesson wasn't "don't think, just scale." It was: **our human intuitions about how to build intelligent systems are often wrong. General methods that leverage learning and computation work better than encoding our clever ideas.**

For agentic AI, this translates to:

### 1. **Trust the Model's Reasoning**
Don't encode your algorithm. Provide clear tools and objectives. Let reasoning emerge.

### 2. **Simplify Your Architecture**
Every complex component should justify its existence. Start simple. Add complexity only when simple fails.

### 3. **Enable Iteration Over Prevention**
Build for recovery, not perfection. Let agents learn from mistakes.

### 4. **Scale Context Before Compressing**
Use full information. Compress only when necessary.

### 5. **Invest in Data Quality**
Clean tools, clear examples, good documentation > clever prompts.

## The Meta-Lesson

The bitter lesson applies to building agentic AI itself. I wanted to be clever. I wanted my engineering expertise to matter. I wanted to build something sophisticated.

But the agents that worked best were the ones where I got out of the way.

This doesn't mean "don't engineer." It means: **engineer the environment, not the behavior.** Build great tools. Provide clear information. Create good feedback loops. Then let the model's general reasoning do its thing.

It's bitter because it's humbling. But it's also liberating.

You don't have to predict every edge case. You don't have to encode every heuristic. You don't have to be clever.

You just have to build a good environment and trust the scale.

---

*What's your experience? Have you found yourself over-engineering agentic systems? I'd love to hear your bitter lessons.*y hard-coded decision is a bet that you know better than the model. You probably don't.

### 3. **Invest in Foundations, Not Clever Tricks**
- Clear tool interfaces > sophisticated orchestration
- Good examples > perfect prompt engineering
- Simple retry logic > predictive error prevention
- Full context > clever compression

### 4. **Enable Learning Loops**
- Let agents iterate and fail
- Provide informative errors
- Allow course correction
- Don't prevent recovery

### 5. **Scale What Works**
- More examples beats few-shot cleverness
- Bigger context beats summarization hacks
- More attempts beats perfect first try

## The Hardest Part

The bitter part of these lessons isn't that they're technically difficult. It's that they're *emotionally* difficult.

As engineers, we want to solve problems with our intelligence. We want to:
- Design elegant architectures
- Craft clever solutions
- Encode our expertise
- Optimize and perfect

But building effective agentic AI often means:
- Deleting code we're proud of
- Choosing simple over clever
- Admitting the model knows better
- Letting emergence replace engineering

This requires humility. It requires trusting that general methods—clear interfaces, good data, iterative learning—will outperform our specific cleverness.

## Looking Forward

I still make these mistakes. I still catch myself:
- Over-engineering workflows
- Optimizing prompts instead of fixing data
- Preventing failures instead of enabling recovery
- Encoding assumptions instead of letting patterns emerge

But now I recognize them faster. And when I do, I remember Sutton's lesson: **70 years of AI research showed that betting against general methods was always wrong.**

Why would agentic AI be different?

## Your Bitter Lessons

These are my lessons from my systems. Yours will be different—different domains, different constraints, different failures.

But I'd bet they'll follow the same pattern: your clever ideas didn't work as well as you hoped, and the simple, general approach you resisted worked better than you expected.

That's the bitter lesson.

---

**Next in this series:** "Building Reliable Agentic AI Systems: Lessons from Production"

*What bitter lessons have you learned building agentic AI? I'd love to hear about the clever things you deleted and the simple things that worked. Share your thoughts in the comments below.*


---

## 中文版本

当我开始构建第一个代理式 AI 系统时，我以为自己理解了 Richard Sutton 的《痛苦的教训》。我读过它，点头认同，甚至在讨论中引用过它来说明规模胜过手工特征的胜利。

但直到我精心设计的代理在生产环境中第三次崩溃时，我才*真正*理解它。

## Sutton 实际说了什么

让我为不熟悉的人回顾一下 Sutton 2019 年的文章。他的核心论点：

> "从 70 年的 AI 研究中可以得出的最大教训是，利用计算的通用方法最终是最有效的，而且优势很大。"

他提供了令人信服的例子：
- **计算机国际象棋**：深蓝的暴力搜索击败了人类的国际象棋知识
- **计算机围棋**：AlphaGo 的自我对弈学习摧毁了手工特征
- **语音识别**：统计方法击败了语音专业知识
- **计算机视觉**：深度学习超越了精心设计的特征

模式是一致且令人谦卑的：**研究人员不断尝试将人类知识注入系统，却不断被随计算扩展的通用方法超越。**

这个教训是"痛苦的"，因为它意味着放弃智力上令人满意的工作——优雅的表示、领域专业知识、巧妙的启发式——转而采用看似几乎...愚蠢的方法。只是投入更多计算。只是让它学习。

## 我的误解

当我开始构建代理式 AI 时，我以为 Sutton 的意思是：

*"使用大模型，不要想太多，规模解决一切。"*

所以我满怀信心地开始构建第一个代理：
- 从 GPT-4 开始（大模型 ✓）
- 让它自己搞清楚（通用方法 ✓）
- 不要过度工程化（避免人类知识 ✓）

然后现实来了。

## 第一个痛苦的教训：简单 ≠ 简单化

我最初的代理架构实际上相当复杂：
- 用于对话流程的复杂状态机
- 包含 15+ 个示例的精心设计的提示模板
- 带指数退避的复杂重试逻辑
- 精心设计的工具选择启发式
- 多阶段验证管道

它很优雅。很聪明。但坏了。

代理会：
- 卡在我的状态机没有预料到的状态中
- 忽略我精心制作的示例并幻想工具
- 重试相同的失败操作，因为我的逻辑没有考虑某些错误类型
- 选择错误的工具，因为我的启发式编码了我的假设，而不是现实

**实际有效的方法：**

```python
# 不用复杂的状态机
system_prompt = """你是一个帮助用户完成任务的助手。
你可以使用工具。一步一步思考接下来该做什么。"""

# 不用 15 个示例
# 只需 2-3 个清晰、多样的示例

# 不用复杂的重试逻辑
max_retries = 3  # 简单计数器
if "error" in result:
    retry_with_error_context(result)

# 不用工具选择启发式
# 让模型根据清晰的工具描述来选择
```

这感觉太简单了。太天真了。但它**可靠地工作**，而我的聪明版本却不断出错。

**Sutton 的教训应用：**不要编码你对代理*应该*如何行为的假设。给它清晰的工具，让模型的通用推理来解决。

## 第二个痛苦的教训：涌现胜过工程

我构建了一个研究代理，需要：
1. 搜索信息
2. 阅读多个来源
3. 综合发现
4. 事实核查声明
5. 生成报告

我的工程化方法：
```python
def research_workflow(query):
    # 步骤 1：搜索策略规划
    search_plan = plan_search_strategy(query)
    
    # 步骤 2：执行搜索
    sources = execute_search_plan(search_plan)
    
    # 步骤 3：相关性过滤
    relevant = filter_by_relevance(sources, query)
    
    # 步骤 4：综合管道
    synthesized = synthesis_pipeline(relevant)
    
    # 步骤 5：事实核查验证
    validated = fact_check(synthesized)
    
    # 步骤 6：报告生成
    return generate_report(validated)
```

每个步骤都有子函数、错误处理、验证逻辑。这是一个漂亮的架构图。

**实际有效的方法：**

```python
def research_workflow(query):
    result = agent.run(
        task=f"研究这个主题并提供全面的报告：{query}",
        tools=[search, read_url, python_repl]
    )
    return result
```

就这样。代理会：
- 在发现知识空白时自然地多次搜索
- 阅读它认为相关的来源
- 在需要事实核查时重新搜索
- 通过其推理过程有机地综合

我设计的复杂工作流？那是我在编码我的人类过程。代理的自然推理流程通常更好——有时它在完全综合*之前*进行事实核查，有时它根据学到的内容以不同的顺序搜索。

**Sutton 的教训应用：**让通用推理涌现。不要硬编码你的人类算法。

## 第三个痛苦的教训：更多上下文胜过巧妙压缩

我痴迷于提示效率。每个 token 都很重要。我构建了：
- 动态示例选择（从 50 个示例库中选择 3 个最相关的）
- 压缩的工具描述（缩写，删除冗余）
- 对话摘要（每 5 轮压缩一次历史）

这感觉很聪明。很高效。LLM 有上下文限制，对吧？

**问题：**
- 动态示例有时会选择混淆模型的边缘案例
- 压缩的描述丢失了关键细节（"proc"又是什么意思？）
- 摘要偶尔会丢弃后来变得关键的上下文

**实际有效的方法：**

只需包含完整的上下文：
- 所有工具描述（即使冗长）
- 完整的对话历史（直到达到实际限制）
- 完整的示例（不要缩写）

当我确实达到上下文限制时，解决方案不是巧妙的压缩——而是**更好的任务分解**（拆分为子任务）或**卸载到检索**（用于长期记忆的 RAG）。

**Sutton 的教训应用：**有了大的上下文窗口，不要过早优化。让模型处理完整信息。扩展上下文比巧妙压缩更有效。

## 第四个痛苦的教训：迭代 > 完美的第一次尝试

我的工程本能：**让它第一次就完美工作。**

所以我构建了：
- 详细的输入验证
- 工具调用前的预检查
- 预测性错误预防
- 输出的置信度评分

**现实：**代理仍然会犯错。但现在它无法恢复，因为我的验证阻止了某些有效路径，而我的预检查增加了延迟。

**实际有效的方法：**

```python
# 简单循环
for attempt in range(max_attempts):
    result = agent.step()
    
    if is_successful(result):
        return result
    
    # 给代理它的错误并让它再试一次
    context.add_error(result.error)
```

代理变得*非常擅长*：
- 识别自己的错误
- 调整方法
- 从错误消息中学习
- 尝试替代策略

我的完美第一次尝试工程？它阻止了代理发展这种韧性。

**Sutton 的教训应用：**代理通过试错学习。不要阻止失败——启用恢复。迭代胜过预测。

## 第五个痛苦的教训：数据质量 > 提示巫术

我花了几周时间完善提示：
- A/B 测试措辞
- 优化指令顺序
- 微调语气和风格
- 制作完美的系统消息

与此同时，我的工具描述不一致，示例有错误，文档不清楚。

**突破：**我花了两天时间清理：
- 工具描述（清晰、一致的格式）
- 示例（准确、多样、调试过的）
- 错误消息（信息丰富、可操作）

我的平庸提示配上干净的数据**远远超过**我的优化提示配上混乱的数据。

**Sutton 的教训应用：**干净、丰富的数据胜过巧妙的提示。每花一小时在提示工程上，可能应该花两小时在数据质量上。

## 这对构建代理式 AI 意味着什么

Sutton 的痛苦教训不是"不要思考，只要扩展"。而是：**我们关于如何构建智能系统的人类直觉往往是错误的。利用学习和计算的通用方法比编码我们的聪明想法更有效。**

对于代理式 AI，这转化为：

### 1. **信任模型的推理**
不要编码你的算法。提供清晰的工具和目标。让推理涌现。

### 2. **简化你的架构**
每个复杂组件都应该证明其存在的必要性。从简单开始。只有在简单失败时才添加复杂性。

### 3. **启用迭代而非预防**
为恢复而构建，而非完美。让代理从错误中学习。

### 4. **在压缩之前扩展上下文**
使用完整信息。只在必要时压缩。

### 5. **投资数据质量**
干净的工具、清晰的示例、良好的文档 > 巧妙的提示。

## 元教训

痛苦的教训也适用于构建代理式 AI 本身。我想变聪明。我想让我的工程专业知识发挥作用。我想构建复杂的东西。

但效果最好的代理是那些我不妨碍它们的代理。

这并不意味着"不要工程化"。它意味着：**工程化环境，而非行为。**构建优秀的工具。提供清晰的信息。创建良好的反馈循环。然后让模型的通用推理发挥作用。

这是痛苦的，因为它令人谦卑。但它也是解放的。

你不必预测每个边缘情况。你不必编码每个启发式。你不必聪明。

你只需要构建一个良好的环境并信任规模。

---

*你的经验是什么？你有没有发现自己过度工程化代理系统？我很想听听你的痛苦教训。在评论中分享你删除的聪明东西和有效的简单东西。*
