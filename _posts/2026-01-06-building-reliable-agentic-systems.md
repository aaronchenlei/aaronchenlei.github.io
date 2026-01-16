---
layout: post
title: "Building Reliable Agentic AI Systems: Lessons from Production"
date: 2026-01-06 10:00:00 +0800
categories: [blog]
comments: true
tags: [Agentic AI, Production, Reliability]
---

> **📖 Language / 语言选择**
>
> [🇬🇧 English Version](#english-version) | [🇨🇳 中文版本](#中文版本)

---

## English Version

After my [previous post](/posts/the-bitter-lesson-for-agentic-ai/) about applying Sutton's Bitter Lesson to agentic AI, several people asked: "Okay, but how do you actually build these systems for production?"

Fair question. Simplicity is great in theory. But production has requirements that demos don't: reliability, observability, cost control, and the ability to debug when things go wrong at 3 AM.

Here's what I learned shipping agentic systems that actually work.

## The Reliability Paradox

**The paradox:** Agents need freedom to reason, but production needs predictability.

My first production agent was a customer support assistant. In testing, it was brilliant—creative problem-solving, natural conversation, helpful suggestions.

In production? Chaos.

- Sometimes it would take 2 steps to solve a problem, sometimes 47
- Token costs varied wildly (one conversation cost $12)
- Response times ranged from 3 seconds to 4 minutes
- It occasionally went down rabbit holes that never converged

Users and finance teams were... unhappy.

### What Worked: Bounded Autonomy

```python
class AgentConfig:
    max_steps: int = 15          # Hard limit
    max_tokens_per_run: int = 50000
    timeout_seconds: int = 120
    
    # Soft limits (warnings, not failures)
    target_steps: int = 8
    target_tokens: int = 20000
    target_time_seconds: int = 30
```

The agent has freedom within bounds. When it hits soft limits, it gets a nudge:

```python
if steps > config.target_steps:
    context.add_message(
        "You've taken more steps than typical. "
        "Consider if you're close to a solution."
    )
```

**Results:**
- 95% of conversations stayed under 10 steps
- Costs became predictable
- Timeouts were rare but prevented runaway cases

**Key insight:** Agents don't need infinite freedom. They need enough freedom within reasonable constraints.

## Observability: The Unsexy Necessity

When an agent fails in production, "it didn't work" is not enough information.

I learned this the hard way when a customer reported: "Your agent gave me wrong information." Which agent run? What was the query? What tools did it use? What did it actually return?

No idea. I had logs, but they were useless.

### What Worked: Structured Tracing

Every agent run gets a trace:

```python
@dataclass
class AgentTrace:
    run_id: str
    user_query: str
    steps: List[StepTrace]
    final_output: str
    metadata: Dict
    
@dataclass  
class StepTrace:
    step_num: int
    reasoning: str
    tool_called: Optional[str]
    tool_input: Optional[Dict]
    tool_output: Optional[str]
    tokens_used: int
    duration_ms: int
```

Now when something fails, I can:
1. Pull up the exact trace
2. See the agent's reasoning at each step
3. Identify where it went wrong
4. Reproduce the issue

**Bonus:** These traces became training data for improving the system.

## The Tool Design Pattern That Changed Everything

Early on, my tools looked like this:

```python
def search_database(query: str) -> str:
    """Search the customer database"""
    results = db.search(query)
    return json.dumps(results)
```

Simple, right? Wrong.

**Problems:**
- Agent didn't know if search succeeded or failed
- No guidance on what to do with results
- No indication if results were empty vs. error
- Agent would retry the same failing query

### What Worked: Rich Tool Responses

```python
@dataclass
class ToolResult:
    success: bool
    data: Any
    message: str
    suggestions: List[str]
    
def search_database(query: str) -> ToolResult:
    try:
        results = db.search(query)
        
        if not results:
            return ToolResult(
                success=True,
                data=[],
                message="No results found",
                suggestions=[
                    "Try broader search terms",
                    "Check spelling",
                    "Use the list_all_customers tool to browse"
                ]
            )
        
        return ToolResult(
            success=True,
            data=results,
            message=f"Found {len(results)} results",
            suggestions=["Use get_customer_details for more info"]
        )
        
    except Exception as e:
        return ToolResult(
            success=False,
            data=None,
            message=f"Search failed: {str(e)}",
            suggestions=["Try again with simpler query"]
        )
```

**Impact:**
- Agent success rate went from 73% to 91%
- Fewer retries (agents knew what to do next)
- Better error recovery
- Easier debugging (clear failure messages)

**Key insight:** Tools are the agent's interface to reality. Make them informative.

## The Cost Control Nobody Talks About

Agentic systems can get expensive fast. My first month in production:
- Average cost per conversation: $0.47
- 95th percentile: $2.13
- One outlier: $31.50

That outlier? Agent got stuck in a loop, making the same API call 200+ times.

### What Worked: Multi-Layer Cost Control

**Layer 1: Caching**
```python
@lru_cache(maxsize=1000)
def get_customer_info(customer_id: str):
    # Expensive API call
    return api.get_customer(customer_id)
```

Simple, but reduced redundant tool calls by 40%.

**Layer 2: Deduplication**
```python
class AgentContext:
    def __init__(self):
        self.tool_calls_made = set()
    
    def call_tool(self, tool_name, args):
        call_signature = (tool_name, json.dumps(args, sort_keys=True))
        
        if call_signature in self.tool_calls_made:
            return cached_result(call_signature)
        
        self.tool_calls_made.add(call_signature)
        return execute_tool(tool_name, args)
```

Prevented the agent from calling the same tool with same args twice.

**Layer 3: Budget Awareness**
```python
if context.tokens_used > config.max_tokens_per_run * 0.8:
    context.add_message(
        "You're approaching token budget. "
        "Prioritize completing the task efficiently."
    )
```

**Results:**
- Average cost dropped to $0.19
- 95th percentile: $0.61
- No more runaway costs

## Error Recovery: The Difference Between Demos and Production

In demos, errors are rare. In production, they're constant:
- APIs timeout
- Rate limits hit
- Data is malformed
- External services go down

My first agent would just... give up. "Sorry, I encountered an error."

### What Worked: Graceful Degradation

```python
class ToolExecutor:
    def execute(self, tool_name, args, context):
        try:
            return self.tools[tool_name](**args)
        
        except RateLimitError:
            context.add_message(
                f"{tool_name} is rate-limited. "
                "Try alternative approaches or wait."
            )
            return ToolResult(
                success=False,
                message="Rate limited",
                suggestions=["Use cached data", "Try different tool"]
            )
        
        except TimeoutError:
            context.add_message(
                f"{tool_name} timed out. "
                "The service might be slow. Consider simpler queries."
            )
            return ToolResult(
                success=False,
                message="Timeout",
                suggestions=["Retry with smaller scope", "Use backup tool"]
            )
        
        except Exception as e:
            # Log for debugging
            logger.error(f"Tool {tool_name} failed", exc_info=True)
            
            # Give agent actionable info
            context.add_message(
                f"{tool_name} failed unexpectedly. "
                "Try accomplishing the goal differently."
            )
            return ToolResult(
                success=False,
                message=f"Error: {type(e).__name__}",
                suggestions=["Try alternative approach"]
            )
```

**Key insight:** Don't just catch errors—give the agent context to work around them.

## Testing Agentic Systems

Unit tests are easy. Integration tests are hard. But the real challenge? Testing emergent behavior.

How do you test "the agent should figure out the right approach"?

### What Worked: Evaluation Suites

```python
@dataclass
class TestCase:
    name: str
    user_query: str
    success_criteria: Callable[[AgentTrace], bool]
    max_acceptable_steps: int
    max_acceptable_cost: float

test_cases = [
    TestCase(
        name="simple_customer_lookup",
        user_query="What's the email for customer John Smith?",
        success_criteria=lambda trace: (
            "john.smith@example.com" in trace.final_output.lower()
        ),
        max_acceptable_steps=5,
        max_acceptable_cost=0.10
    ),
    TestCase(
        name="multi_step_research",
        user_query="Find all customers in California who haven't ordered in 6 months",
        success_criteria=lambda trace: (
            trace.steps[-1].tool_called == "search_database" and
            "california" in str(trace.steps[-1].tool_input).lower()
        ),
        max_acceptable_steps=10,
        max_acceptable_cost=0.30
    )
]
```

Run these on every change. Track:
- Success rate
- Average steps
- Average cost
- Regression detection

**Not perfect, but catches:**
- Performance regressions
- Cost increases
- Breaking changes
- Common failure modes

## The Human-in-the-Loop Pattern

Some decisions are too important for full autonomy. But interrupting the user for every decision kills the experience.

### What Worked: Confidence-Based Escalation

```python
class Agent:
    def should_escalate(self, action, context):
        # High-risk actions always escalate
        if action.tool_name in ["delete_customer", "refund_order"]:
            return True
        
        # Low confidence escalates
        if action.confidence < 0.7:
            return True
        
        # Novel situations escalate
        if not self.seen_similar_case(action, context):
            return True
        
        return False
    
    def execute_step(self, action, context):
        if self.should_escalate(action, context):
            approval = request_human_approval(action, context)
            if not approval.approved:
                return self.replan(approval.feedback)
        
        return self.execute_action(action)
```

**Results:**
- Critical errors dropped to near-zero
- User trust increased
- Escalation rate: ~5% (acceptable)

## What I'd Do Differently

If I started over today:

1. **Start with observability** - Build tracing from day one, not as an afterthought
2. **Design tools for agents** - Not just wrapped APIs, but agent-friendly interfaces
3. **Budget for evaluation** - Testing agentic systems takes time and money
4. **Plan for failure** - Graceful degradation isn't optional
5. **Iterate on real usage** - Synthetic tests only get you so far

## The Bottom Line

Building production agentic systems isn't just about picking the right model or writing clever prompts. It's about:

- **Constraints** that enable reliability
- **Observability** that enables debugging  
- **Tool design** that enables success
- **Cost control** that enables scale
- **Error handling** that enables resilience

The agents that work in production aren't the most autonomous or the most clever. They're the ones that fail gracefully, stay within bounds, and give you visibility when things go wrong.

Simplicity is still the goal. But production-ready simplicity requires thoughtful infrastructure.

---

*What's been your biggest challenge deploying agentic systems? Share your experiences and the patterns that worked for you in the comments below.*



---

## 中文版本

在我[上一篇文章](/posts/the-bitter-lesson-for-agentic-ai/)关于将 Sutton 的痛苦教训应用于代理式 AI 之后，有几个人问："好的，但你实际上如何为生产环境构建这些系统？"

这是个好问题。简单在理论上很好。但生产环境有演示没有的要求：可靠性、可观察性、成本控制，以及在凌晨 3 点出问题时能够调试的能力。

以下是我在交付真正有效的代理系统时学到的东西。

## 可靠性悖论

**悖论：**代理需要推理的自由，但生产需要可预测性。

我的第一个生产代理是客户支持助手。在测试中，它很出色——创造性的问题解决、自然的对话、有用的建议。

在生产中？混乱。

- 有时它用 2 步解决问题，有时用 47 步
- Token 成本变化很大（一次对话花费 12 美元）
- 响应时间从 3 秒到 4 分钟不等
- 它偶尔会陷入永不收敛的兔子洞

用户和财务团队...不高兴。

### 有效的方法：有界自主

```python
class AgentConfig:
    max_steps: int = 15          # 硬限制
    max_tokens_per_run: int = 50000
    timeout_seconds: int = 120
    
    # 软限制（警告，不是失败）
    target_steps: int = 8
    target_tokens: int = 20000
    target_time_seconds: int = 30
```

代理在边界内有自由。当它达到软限制时，它会得到一个提示：

```python
if steps > config.target_steps:
    context.add_message(
        "你已经采取了比通常更多的步骤。"
        "考虑一下你是否接近解决方案。"
    )
```

**结果：**
- 95% 的对话保持在 10 步以下
- 成本变得可预测
- 超时很少见，但防止了失控的情况

**关键见解：**代理不需要无限的自由。它们需要在合理约束内的足够自由。

## 可观察性：不性感的必需品

当代理在生产中失败时，"它不工作"不是足够的信息。

我在客户报告"你的代理给了我错误的信息"时艰难地学到了这一点。哪次代理运行？查询是什么？它使用了什么工具？它实际返回了什么？

不知道。我有日志，但它们没用。

### 有效的方法：结构化跟踪

每次代理运行都有一个跟踪：

```python
@dataclass
class AgentTrace:
    run_id: str
    user_query: str
    steps: List[StepTrace]
    final_output: str
    metadata: Dict
    
@dataclass  
class StepTrace:
    step_num: int
    reasoning: str
    tool_called: Optional[str]
    tool_input: Optional[Dict]
    tool_output: Optional[str]
    tokens_used: int
    duration_ms: int
```

现在当出现问题时，我可以：
1. 调出确切的跟踪
2. 查看代理在每一步的推理
3. 识别它在哪里出错
4. 重现问题

**额外收获：**这些跟踪成为改进系统的训练数据。

## 改变一切的工具设计模式

早期，我的工具看起来像这样：

```python
def search_database(query: str) -> str:
    """搜索客户数据库"""
    results = db.search(query)
    return json.dumps(results)
```

简单，对吧？错了。

**问题：**
- 代理不知道搜索是成功还是失败
- 没有关于如何处理结果的指导
- 没有指示结果是空的还是错误
- 代理会重试相同的失败查询

### 有效的方法：丰富的工具响应

```python
@dataclass
class ToolResult:
    success: bool
    data: Any
    message: str
    suggestions: List[str]
    
def search_database(query: str) -> ToolResult:
    try:
        results = db.search(query)
        
        if not results:
            return ToolResult(
                success=True,
                data=[],
                message="未找到结果",
                suggestions=[
                    "尝试更广泛的搜索词",
                    "检查拼写",
                    "使用 list_all_customers 工具浏览"
                ]
            )
        
        return ToolResult(
            success=True,
            data=results,
            message=f"找到 {len(results)} 个结果",
            suggestions=["使用 get_customer_details 获取更多信息"]
        )
        
    except Exception as e:
        return ToolResult(
            success=False,
            data=None,
            message=f"搜索失败：{str(e)}",
            suggestions=["用更简单的查询重试"]
        )
```

**影响：**
- 代理成功率从 73% 提高到 91%
- 更少的重试（代理知道接下来该做什么）
- 更好的错误恢复
- 更容易调试（清晰的失败消息）

**关键见解：**工具是代理与现实的接口。让它们提供信息。

## 没人谈论的成本控制

代理系统可能很快变得昂贵。我生产的第一个月：
- 每次对话的平均成本：0.47 美元
- 95 百分位：2.13 美元
- 一个异常值：31.50 美元

那个异常值？代理陷入循环，进行了 200 多次相同的 API 调用。

### 有效的方法：多层成本控制

**第 1 层：缓存**
```python
@lru_cache(maxsize=1000)
def get_customer_info(customer_id: str):
    # 昂贵的 API 调用
    return api.get_customer(customer_id)
```

简单，但减少了 40% 的冗余工具调用。

**第 2 层：去重**
```python
class AgentContext:
    def __init__(self):
        self.tool_calls_made = set()
    
    def call_tool(self, tool_name, args):
        call_signature = (tool_name, json.dumps(args, sort_keys=True))
        
        if call_signature in self.tool_calls_made:
            return cached_result(call_signature)
        
        self.tool_calls_made.add(call_signature)
        return execute_tool(tool_name, args)
```

防止代理用相同参数调用相同工具两次。

**第 3 层：预算意识**
```python
if context.tokens_used > config.max_tokens_per_run * 0.8:
    context.add_message(
        "你正在接近 token 预算。"
        "优先高效地完成任务。"
    )
```

**结果：**
- 平均成本降至 0.19 美元
- 95 百分位：0.61 美元
- 不再有失控的成本

## 错误恢复：演示与生产的区别

在演示中，错误很少见。在生产中，它们是常态：
- API 超时
- 达到速率限制
- 数据格式错误
- 外部服务宕机

我的第一个代理只会...放弃。"抱歉，我遇到了错误。"

### 有效的方法：优雅降级

```python
class ToolExecutor:
    def execute(self, tool_name, args, context):
        try:
            return self.tools[tool_name](**args)
        
        except RateLimitError:
            context.add_message(
                f"{tool_name} 受到速率限制。"
                "尝试替代方法或等待。"
            )
            return ToolResult(
                success=False,
                message="速率受限",
                suggestions=["使用缓存数据", "尝试不同的工具"]
            )
        
        except TimeoutError:
            context.add_message(
                f"{tool_name} 超时。"
                "服务可能很慢。考虑更简单的查询。"
            )
            return ToolResult(
                success=False,
                message="超时",
                suggestions=["用更小的范围重试", "使用备用工具"]
            )
        
        except Exception as e:
            # 记录以供调试
            logger.error(f"工具 {tool_name} 失败", exc_info=True)
            
            # 给代理可操作的信息
            context.add_message(
                f"{tool_name} 意外失败。"
                "尝试以不同方式完成目标。"
            )
            return ToolResult(
                success=False,
                message=f"错误：{type(e).__name__}",
                suggestions=["尝试替代方法"]
            )
```

**关键见解：**不要只是捕获错误——给代理上下文来解决它们。

## 测试代理系统

单元测试很容易。集成测试很难。但真正的挑战？测试涌现行为。

如何测试"代理应该找出正确的方法"？

### 有效的方法：评估套件

```python
@dataclass
class TestCase:
    name: str
    user_query: str
    success_criteria: Callable[[AgentTrace], bool]
    max_acceptable_steps: int
    max_acceptable_cost: float

test_cases = [
    TestCase(
        name="simple_customer_lookup",
        user_query="客户 John Smith 的电子邮件是什么？",
        success_criteria=lambda trace: (
            "john.smith@example.com" in trace.final_output.lower()
        ),
        max_acceptable_steps=5,
        max_acceptable_cost=0.10
    ),
    TestCase(
        name="multi_step_research",
        user_query="查找加利福尼亚州 6 个月内没有订购的所有客户",
        success_criteria=lambda trace: (
            trace.steps[-1].tool_called == "search_database" and
            "california" in str(trace.steps[-1].tool_input).lower()
        ),
        max_acceptable_steps=10,
        max_acceptable_cost=0.30
    )
]
```

在每次更改时运行这些。跟踪：
- 成功率
- 平均步数
- 平均成本
- 回归检测

**不完美，但能捕获：**
- 性能回归
- 成本增加
- 破坏性变更
- 常见失败模式

## 人在回路模式

有些决策对于完全自主来说太重要了。但为每个决策打断用户会破坏体验。

### 有效的方法：基于置信度的升级

```python
class Agent:
    def should_escalate(self, action, context):
        # 高风险操作总是升级
        if action.tool_name in ["delete_customer", "refund_order"]:
            return True
        
        # 低置信度升级
        if action.confidence < 0.7:
            return True
        
        # 新情况升级
        if not self.seen_similar_case(action, context):
            return True
        
        return False
    
    def execute_step(self, action, context):
        if self.should_escalate(action, context):
            approval = request_human_approval(action, context)
            if not approval.approved:
                return self.replan(approval.feedback)
        
        return self.execute_action(action)
```

**结果：**
- 关键错误降至接近零
- 用户信任增加
- 升级率：约 5%（可接受）

## 我会做得不同的事

如果今天重新开始：

1. **从可观察性开始** - 从第一天就构建跟踪，而不是事后补充
2. **为代理设计工具** - 不只是包装的 API，而是代理友好的接口
3. **为评估预算** - 测试代理系统需要时间和金钱
4. **为失败做计划** - 优雅降级不是可选的
5. **在真实使用中迭代** - 合成测试只能让你走这么远

## 结论

构建生产代理系统不仅仅是选择正确的模型或编写巧妙的提示。它关乎：

- **约束**使可靠性成为可能
- **可观察性**使调试成为可能
- **工具设计**使成功成为可能
- **成本控制**使规模成为可能
- **错误处理**使韧性成为可能

在生产中有效的代理不是最自主或最聪明的。它们是那些优雅失败、保持在边界内、并在出错时给你可见性的代理。

简单仍然是目标。但生产就绪的简单需要深思熟虑的基础设施。

---

*部署代理系统时你最大的挑战是什么？在下面的评论中分享你的经验和对你有效的模式。*
