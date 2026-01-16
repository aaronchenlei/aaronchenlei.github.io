---
layout: post
title: "The Framework Trap: My Bitter Lesson in Agentic AI Development"
date: 2026-01-09 10:00:00 +0800
categories: [blog]
comments: true
tags: [Agentic AI, LangChain, LangGraph, Framework, Software Engineering]
---

> **📖 Language / 语言选择**
>
> [🇬🇧 English Version](#english-version) | [🇨🇳 中文版本](#中文版本)

---

## English Version

I spent three weeks integrating LangChain into my agentic AI system. I read the docs, watched tutorials, refactored my code to fit "the LangChain way." I felt productive. I felt like I was doing it right.

Then I spent two days ripping it all out and replacing it with 200 lines of Python.

The system got faster. The code got clearer. Debugging became trivial. And I finally understood another bitter lesson: **frameworks promise to simplify your work, but often they just shift complexity from your problem domain to framework mastery.**

## The Seduction of Frameworks

When I started building my telecom agent system, LangChain seemed perfect:

- **Promises**: "Build LLM applications in minutes"
- **Ecosystem**: Integrations with every LLM, vector DB, tool
- **Community**: Thousands of examples, active development
- **Best Practices**: Surely the framework embodies lessons from thousands of projects

I was sold. Who wants to reinvent the wheel?

## The First Warning Signs

Week one was exciting. I had a working prototype using LangChain's agent executor. But then I needed to add custom retry logic for our flaky telecom APIs.

**My thought process:**
1. Check LangChain docs for retry configuration
2. Find it's buried in some callback mechanism
3. Try to configure it, doesn't quite work for my case
4. Read source code to understand what's happening
5. Discover I need to subclass three different classes
6. Implement workaround that feels hacky

**Time spent:** 6 hours for what should have been a 10-line retry decorator.

I told myself: "I'm just learning the framework. It'll get easier."

It didn't.

## Framework Promises vs. Reality

Let me show you what I expected versus what I got:

| Framework Promise | What I Actually Got | What I Really Needed |
|------------------|---------------------|---------------------|
| Rapid development | Steep learning curve | LLM API call + tool execution |
| Best practices built-in | Opinionated abstractions | Simple retry logic |
| Rich ecosystem | Version incompatibilities | JSON parsing |
| Production-ready | Hidden complexity | Structured logging |
| Easy debugging | 5-layer stack traces | Direct control flow |

The gap between promise and reality kept growing.

## The Framework Tax

Here are the specific costs I paid for using a framework:

### 1. Abstraction Overhead

A simple tool call in my original code:

```python
def call_api(endpoint, params):
    response = requests.post(f"{BASE_URL}/{endpoint}", json=params)
    return response.json()
```

The LangChain version:

```python
from langchain.tools import BaseTool
from langchain.callbacks.manager import CallbackManagerForToolRun
from typing import Optional

class APITool(BaseTool):
    name = "api_call"
    description = "Calls the telecom API"
    
    def _run(
        self, 
        endpoint: str,
        params: dict,
        run_manager: Optional[CallbackManagerForToolRun] = None
    ) -> str:
        response = requests.post(f"{BASE_URL}/{endpoint}", json=params)
        return json.dumps(response.json())
    
    async def _arun(self, endpoint: str, params: dict):
        raise NotImplementedError("Async not supported")
```

Five times the code. And I still don't understand what CallbackManagerForToolRun does.

### 2. Hidden Complexity

When my agent failed, the error trace looked like this:

```
Traceback (most recent call last):
  File "langchain/agents/agent.py", line 1097, in _call
  File "langchain/chains/base.py", line 312, in __call__
  File "langchain/chains/llm.py", line 285, in _call
  File "langchain/llms/base.py", line 456, in generate
  File "langchain/llms/openai.py", line 198, in _generate
  ...
```

I couldn't tell if the problem was:
- My prompt
- My tool definition
- LangChain's agent logic
- The LLM's response
- Something in the callback chain

**Debugging time:** Hours, because I had to understand LangChain's internals.

### 3. Version Hell

Over three months:
- LangChain updated 12 times
- 3 breaking changes in agent executor API
- 2 breaking changes in tool interface
- 1 complete restructuring of callback system

Each update meant:
- Reading migration guides
- Refactoring code
- Testing everything again
- Hoping nothing else broke

### 4. Unnecessary Dependencies

To use LangChain's agent, I had to install:

```
langchain==0.1.0
langchain-core==0.1.10
langchain-community==0.0.13
pydantic==2.5.0
SQLAlchemy==2.0.23
tenacity==8.2.3
...15 more packages
```

I used maybe 10% of LangChain's features. But I got 100% of its dependencies, 100% of its security vulnerabilities, and 100% of its version conflicts.

### 5. Performance Overhead

Benchmarking revealed:

| Metric | LangChain Version | Simple Version | Difference |
|--------|------------------|----------------|------------|
| Cold start | 2.3s | 0.1s | 23x slower |
| Memory usage | 450MB | 80MB | 5.6x more |
| Avg latency | 340ms | 180ms | 1.9x slower |

The framework's generality came with real costs.

## The Breaking Point

The moment I decided to delete LangChain came during a production incident.

**The problem:** Agent was making duplicate API calls, costing us money and hitting rate limits.

**What I needed:** See exactly what the agent was doing, step by step.

**What I got:** A maze of callbacks, chains, and abstractions. I couldn't even find where the duplicate calls were happening without adding print statements to LangChain's source code.

I spent 4 hours debugging. Then I asked myself: **"What if I just wrote this myself?"**

## The Rewrite: Two Days to Clarity

I started from scratch. What did I actually need?

1. Call an LLM with messages
2. Parse tool calls from the response
3. Execute tools
4. Add results back to messages
5. Repeat until done

Here's what I built:

```python
from dataclasses import dataclass
from typing import List, Callable, Any
import openai

@dataclass
class Message:
    role: str
    content: str

@dataclass
class ToolCall:
    name: str
    arguments: dict

class Agent:
    def __init__(self, tools: dict[str, Callable], model: str = "gpt-4"):
        self.tools = tools
        self.model = model
        self.messages: List[Message] = []
    
    def run(self, task: str, max_steps: int = 10) -> str:
        self.messages = [Message(role="user", content=task)]
        
        for step in range(max_steps):
            # Call LLM
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[{"role": m.role, "content": m.content} 
                         for m in self.messages],
                tools=self._format_tools()
            )
            
            message = response.choices[0].message
            
            # If no tool call, we're done
            if not message.tool_calls:
                return message.content
            
            # Execute tool calls
            for tool_call in message.tool_calls:
                result = self._execute_tool(
                    tool_call.function.name,
                    json.loads(tool_call.function.arguments)
                )
                self.messages.append(
                    Message(role="tool", content=str(result))
                )
        
        return "Max steps reached"
    
    def _execute_tool(self, name: str, args: dict) -> Any:
        if name not in self.tools:
            return f"Error: Unknown tool {name}"
        
        try:
            return self.tools[name](**args)
        except Exception as e:
            return f"Error: {str(e)}"
    
    def _format_tools(self) -> List[dict]:
        return [
            {
                "type": "function",
                "function": {
                    "name": name,
                    "description": func.__doc__ or "",
                    "parameters": self._get_parameters(func)
                }
            }
            for name, func in self.tools.items()
        ]
    
    def _get_parameters(self, func: Callable) -> dict:
        # Simple parameter extraction from function signature
        # In production, use inspect module or pydantic
        return {"type": "object", "properties": {}}
```

That's it. The core agent logic in ~60 lines.

## The Results

After the rewrite:

| Metric | Before (LangChain) | After (Simple) | Change |
|--------|-------------------|----------------|--------|
| Lines of code | 2,847 | 312 | -89% |
| Dependencies | 18 | 3 | -83% |
| Cold start time | 2.3s | 0.1s | -96% |
| Memory usage | 450MB | 80MB | -82% |
| Avg debug time | 45min | 8min | -82% |
| Test coverage | 67% | 94% | +40% |

But the biggest win wasn't in the metrics. It was in **understanding**.

When something broke, I knew exactly where to look. When I needed a feature, I knew exactly where to add it. When someone asked "how does this work?", I could explain it in 5 minutes.

## What I Actually Needed

Looking back, my real requirements were simple:

**Core functionality:**
- LLM API calls (OpenAI SDK: 1 function)
- Tool execution (Python functions: 10 lines)
- Retry logic (tenacity library: 1 decorator)
- Logging (Python logging: built-in)

**That's it.** Everything else was framework overhead.

Here's my complete "framework" for production:

```python
from tenacity import retry, stop_after_attempt, wait_exponential
import logging

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def call_llm(messages, tools):
    logger.info(f"Calling LLM with {len(messages)} messages")
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=messages,
        tools=tools
    )
    logger.info(f"LLM response: {response.choices[0].message}")
    return response

def execute_tool(name, args, tools):
    logger.info(f"Executing tool: {name} with args: {args}")
    try:
        result = tools[name](**args)
        logger.info(f"Tool result: {result}")
        return result
    except Exception as e:
        logger.error(f"Tool execution failed: {e}", exc_info=True)
        return {"error": str(e)}
```

Add the Agent class from earlier, and you have a complete, production-ready agentic system in ~150 lines.

## When Should You Use a Framework?

I'm not saying frameworks are always bad. They have their place:

### Use a framework when:

✅ **Your needs align perfectly with the framework's design**
   - Example: You're building exactly what LangChain's tutorials show

✅ **You need the entire ecosystem**
   - Example: You want LangSmith monitoring, LangServe deployment, etc.

✅ **Your team already knows it**
   - Example: Everyone's trained on LangChain, switching costs are high

✅ **You're prototyping and speed matters more than understanding**
   - Example: Hackathon, quick demo, proof of concept

### Don't use a framework when:

❌ **You're just starting and don't know your requirements yet**
   - You'll spend more time fighting the framework than solving problems

❌ **Your use case is simple**
   - LLM + tools + retry logic doesn't need a framework

❌ **You need custom behavior**
   - Frameworks are opinionated; customization is painful

❌ **You're building for production and need full control**
   - Debugging, performance, security all require understanding

❌ **"Everyone else is using it" is your only reason**
   - Cargo culting leads to unnecessary complexity

## A Better Approach: Start Simple

Here's what I recommend:

### 1. Start with direct API calls

```python
# Just call the LLM directly
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}]
)
```

Don't abstract yet. Understand what you're doing.

### 2. Add only what you need, when you need it

Need retry logic? Add tenacity.
Need logging? Add structured logging.
Need tool calling? Add a simple tool registry.

Each addition should solve a real problem you've encountered.

### 3. Abstract only when you have duplication

If you're writing the same code three times, then abstract it. Not before.

### 4. Keep it readable

Your code should be explainable to a new team member in 10 minutes. If it takes longer, you've over-engineered.

### 5. Measure the cost of dependencies

Every dependency is a liability:
- Security vulnerabilities
- Version conflicts
- Learning curve
- Maintenance burden

Make sure the benefit outweighs the cost.

## My Simple Agentic AI Stack

For production telecom agents, here's my entire stack:

**LLM calls:**
```python
import openai  # Direct SDK
```

**Tool execution:**
```python
# Just Python functions with type hints
def query_subscriber(phone: str) -> dict:
    """Query subscriber information"""
    return db.query(phone)
```

**Retry logic:**
```python
from tenacity import retry, stop_after_attempt
```

**Logging:**
```python
import logging
import json

logger = logging.getLogger(__name__)

def log_agent_step(step, action, result):
    logger.info(json.dumps({
        "step": step,
        "action": action,
        "result": result,
        "timestamp": datetime.now().isoformat()
    }))
```

**Testing:**
```python
import pytest

def test_agent_handles_missing_subscriber():
    agent = Agent(tools={"query": mock_query})
    result = agent.run("Find subscriber 555-0000")
    assert "not found" in result.lower()
```

Total dependencies: 3 (openai, tenacity, pytest)
Total lines: ~500 including tests
Total understanding: 100%

## The Meta-Lesson

This is another bitter lesson in the spirit of Sutton's original essay.

I wanted to be smart. I wanted to use the "right" tools. I wanted to follow best practices. I wanted the framework to make me productive.

But reality taught me:

- **Simple beats clever**
- **Understanding beats abstraction**
- **Direct beats indirect**
- **Fewer dependencies beat rich ecosystems**
- **Your code beats someone else's framework**

The bitter part? Admitting that I wasted three weeks because I didn't trust myself to build something simple.

The liberating part? Realizing that most problems don't need frameworks—they need clear thinking and straightforward code.

## Questions to Ask Before Adopting a Framework

Next time you're tempted by a framework, ask yourself:

1. **Can I explain my core requirements in 3 sentences?**
   - If yes, you probably don't need a framework
   - If no, figure out your requirements first

2. **Can I build a minimal version in a day?**
   - If yes, do that first
   - If no, maybe you need the framework—or maybe your problem is too complex

3. **Do I understand what the framework does under the hood?**
   - If yes, you might not need it
   - If no, you'll struggle when things break

4. **Am I using >50% of the framework's features?**
   - If yes, it might be worth it
   - If no, you're paying for things you don't use

5. **Would I recommend this framework to a junior developer?**
   - If no, it's probably too complex
   - If yes, make sure it's for the right reasons

## The Bottom Line

Frameworks promise to make you productive. Sometimes they deliver. Often they just shift complexity from your problem domain to framework mastery.

For agentic AI systems, I've learned:

- **Start simple**: LLM + tools + retry logic
- **Add incrementally**: Only when you hit real problems
- **Understand everything**: If you can't explain it, you don't control it
- **Measure costs**: Dependencies, performance, complexity
- **Stay flexible**: Requirements change; frameworks don't

The best framework is often no framework at all—just clear code that solves your specific problem.

Your mileage may vary. Your requirements might genuinely need LangChain or LangGraph. But make sure you're choosing the framework because it solves your problems, not because it's popular or because you think you should.

## What's Your Experience?

Have you felt trapped by a framework? Did you stick with it or simplify? What's your approach to choosing tools for agentic AI?

I'd love to hear your stories—especially if you disagree with my approach. The best lessons come from diverse experiences.

---

**Next in this series:** "Tool Design Patterns: Making Your Agent Actually Useful"

*Share your framework experiences in the comments. Let's learn from each other's bitter lessons.*


---

## 中文版本

我花了三周时间把 LangChain 集成到我的代理式 AI 系统中。我读文档、看教程，重构代码以适应"LangChain 的方式"。我感觉很高效，觉得自己在做正确的事。

然后我用两天时间把它全部删掉，换成了 200 行 Python 代码。

系统变快了，代码变清晰了，调试变简单了。我终于明白了另一个痛苦的教训：**框架承诺简化你的工作，但往往只是把复杂度从问题域转移到了框架掌握上。**

## 框架的诱惑

当我开始构建电信代理系统时，LangChain 看起来很完美：

- **承诺**："几分钟内构建 LLM 应用"
- **生态系统**：与所有 LLM、向量数据库、工具的集成
- **社区**：数千个示例，活跃的开发
- **最佳实践**：框架肯定包含了数千个项目的经验教训

我被说服了。谁想重复造轮子呢？

## 第一个警告信号

第一周很兴奋。我用 LangChain 的 agent executor 做出了一个可工作的原型。但随后我需要为我们不稳定的电信 API 添加自定义重试逻辑。

**我的思考过程：**
1. 查看 LangChain 文档中的重试配置
2. 发现它埋在某个回调机制里
3. 尝试配置，但不太适合我的场景
4. 阅读源代码理解发生了什么
5. 发现我需要继承三个不同的类
6. 实现了一个感觉很 hack 的解决方案

**花费时间：**6 小时，而这本应该是一个 10 行的重试装饰器。

我告诉自己："我只是在学习框架，会越来越容易的。"

并没有。

## 框架承诺 vs 现实

让我展示一下我的期望与实际得到的：

| 框架承诺 | 实际得到的 | 真正需要的 |
|---------|-----------|----------|
| 快速开发 | 陡峭的学习曲线 | LLM API 调用 + 工具执行 |
| 内置最佳实践 | 固执的抽象 | 简单的重试逻辑 |
| 丰富的生态系统 | 版本不兼容 | JSON 解析 |
| 生产就绪 | 隐藏的复杂性 | 结构化日志 |
| 易于调试 | 5 层堆栈跟踪 | 直接的控制流 |

承诺与现实之间的差距越来越大。

## 框架税

以下是我为使用框架付出的具体代价：

### 1. 抽象开销

我原始代码中的一个简单工具调用：

```python
def call_api(endpoint, params):
    response = requests.post(f"{BASE_URL}/{endpoint}", json=params)
    return response.json()
```

LangChain 版本：

```python
from langchain.tools import BaseTool
from langchain.callbacks.manager import CallbackManagerForToolRun
from typing import Optional

class APITool(BaseTool):
    name = "api_call"
    description = "Calls the telecom API"
    
    def _run(
        self, 
        endpoint: str,
        params: dict,
        run_manager: Optional[CallbackManagerForToolRun] = None
    ) -> str:
        response = requests.post(f"{BASE_URL}/{endpoint}", json=params)
        return json.dumps(response.json())
    
    async def _arun(self, endpoint: str, params: dict):
        raise NotImplementedError("Async not supported")
```

代码量是原来的五倍。而且我仍然不明白 CallbackManagerForToolRun 是干什么的。

### 2. 隐藏的复杂性

当我的代理失败时，错误跟踪看起来是这样的：

```
Traceback (most recent call last):
  File "langchain/agents/agent.py", line 1097, in _call
  File "langchain/chains/base.py", line 312, in __call__
  File "langchain/chains/llm.py", line 285, in _call
  File "langchain/llms/base.py", line 456, in generate
  File "langchain/llms/openai.py", line 198, in _generate
  ...
```

我无法判断问题出在：
- 我的提示词
- 我的工具定义
- LangChain 的代理逻辑
- LLM 的响应
- 回调链中的某个环节

**调试时间：**数小时，因为我必须理解 LangChain 的内部机制。

### 3. 版本地狱

三个月内：
- LangChain 更新了 12 次
- agent executor API 有 3 次破坏性变更
- 工具接口有 2 次破坏性变更
- 回调系统完全重构 1 次

每次更新意味着：
- 阅读迁移指南
- 重构代码
- 重新测试所有功能
- 祈祷没有其他东西坏掉

### 4. 不必要的依赖

要使用 LangChain 的代理，我必须安装：

```
langchain==0.1.0
langchain-core==0.1.10
langchain-community==0.0.13
pydantic==2.5.0
SQLAlchemy==2.0.23
tenacity==8.2.3
...还有 15 个包
```

我可能只用了 LangChain 10% 的功能。但我得到了 100% 的依赖、100% 的安全漏洞和 100% 的版本冲突。

### 5. 性能开销

基准测试显示：

| 指标 | LangChain 版本 | 简单版本 | 差异 |
|-----|--------------|---------|------|
| 冷启动 | 2.3秒 | 0.1秒 | 慢 23 倍 |
| 内存使用 | 450MB | 80MB | 多 5.6 倍 |
| 平均延迟 | 340毫秒 | 180毫秒 | 慢 1.9 倍 |

框架的通用性带来了实实在在的代价。

## 临界点

我决定删除 LangChain 的那一刻发生在一次生产事故中。

**问题：**代理在进行重复的 API 调用，花费我们的钱并触及速率限制。

**我需要的：**准确看到代理在做什么，一步一步地。

**我得到的：**一个由回调、链和抽象组成的迷宫。我甚至无法找到重复调用发生在哪里，除非在 LangChain 的源代码中添加 print 语句。

我花了 4 小时调试。然后我问自己：**"如果我自己写会怎样?"**

## 重写：两天到清晰

我从头开始。我实际需要什么？

1. 用消息调用 LLM
2. 从响应中解析工具调用
3. 执行工具
4. 将结果添加回消息
5. 重复直到完成

这是我构建的：

```python
from dataclasses import dataclass
from typing import List, Callable, Any
import openai

@dataclass
class Message:
    role: str
    content: str

@dataclass
class ToolCall:
    name: str
    arguments: dict

class Agent:
    def __init__(self, tools: dict[str, Callable], model: str = "gpt-4"):
        self.tools = tools
        self.model = model
        self.messages: List[Message] = []
    
    def run(self, task: str, max_steps: int = 10) -> str:
        self.messages = [Message(role="user", content=task)]
        
        for step in range(max_steps):
            # 调用 LLM
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[{"role": m.role, "content": m.content} 
                         for m in self.messages],
                tools=self._format_tools()
            )
            
            message = response.choices[0].message
            
            # 如果没有工具调用，我们完成了
            if not message.tool_calls:
                return message.content
            
            # 执行工具调用
            for tool_call in message.tool_calls:
                result = self._execute_tool(
                    tool_call.function.name,
                    json.loads(tool_call.function.arguments)
                )
                self.messages.append(
                    Message(role="tool", content=str(result))
                )
        
        return "达到最大步数"
    
    def _execute_tool(self, name: str, args: dict) -> Any:
        if name not in self.tools:
            return f"错误：未知工具 {name}"
        
        try:
            return self.tools[name](**args)
        except Exception as e:
            return f"错误：{str(e)}"
    
    def _format_tools(self) -> List[dict]:
        return [
            {
                "type": "function",
                "function": {
                    "name": name,
                    "description": func.__doc__ or "",
                    "parameters": self._get_parameters(func)
                }
            }
            for name, func in self.tools.items()
        ]
    
    def _get_parameters(self, func: Callable) -> dict:
        # 从函数签名简单提取参数
        # 生产环境中使用 inspect 模块或 pydantic
        return {"type": "object", "properties": {}}
```

就是这样。核心代理逻辑约 60 行。

## 结果

重写后：

| 指标 | 之前 (LangChain) | 之后 (简单版) | 变化 |
|-----|----------------|-------------|------|
| 代码行数 | 2,847 | 312 | -89% |
| 依赖数量 | 18 | 3 | -83% |
| 冷启动时间 | 2.3秒 | 0.1秒 | -96% |
| 内存使用 | 450MB | 80MB | -82% |
| 平均调试时间 | 45分钟 | 8分钟 | -82% |
| 测试覆盖率 | 67% | 94% | +40% |

但最大的收获不在指标上，而在于**理解**。

当出现问题时，我确切知道该看哪里。当我需要一个功能时，我确切知道在哪里添加。当有人问"这是怎么工作的?"，我可以在 5 分钟内解释清楚。

## 我真正需要的

回顾过去，我的真实需求很简单：

**核心功能：**
- LLM API 调用（OpenAI SDK：1 个函数）
- 工具执行（Python 函数：10 行）
- 重试逻辑（tenacity 库：1 个装饰器）
- 日志记录（Python logging：内置）

**就这些。**其他一切都是框架开销。

这是我用于生产的完整"框架"：

```python
from tenacity import retry, stop_after_attempt, wait_exponential
import logging

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def call_llm(messages, tools):
    logger.info(f"使用 {len(messages)} 条消息调用 LLM")
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=messages,
        tools=tools
    )
    logger.info(f"LLM 响应：{response.choices[0].message}")
    return response

def execute_tool(name, args, tools):
    logger.info(f"执行工具：{name}，参数：{args}")
    try:
        result = tools[name](**args)
        logger.info(f"工具结果：{result}")
        return result
    except Exception as e:
        logger.error(f"工具执行失败：{e}", exc_info=True)
        return {"error": str(e)}
```

加上前面的 Agent 类，你就有了一个完整的、生产就绪的代理系统，约 150 行代码。

## 什么时候应该使用框架？

我不是说框架总是不好的。它们有自己的位置：

### 应该使用框架的情况：

✅ **你的需求与框架设计完美契合**
   - 例如：你正在构建的正是 LangChain 教程展示的内容

✅ **你需要整个生态系统**
   - 例如：你想要 LangSmith 监控、LangServe 部署等

✅ **你的团队已经熟悉它**
   - 例如：每个人都接受过 LangChain 培训，切换成本很高

✅ **你在做原型，速度比理解更重要**
   - 例如：黑客马拉松、快速演示、概念验证

### 不应该使用框架的情况：

❌ **你刚开始，还不知道自己的需求**
   - 你会花更多时间与框架斗争而不是解决问题

❌ **你的用例很简单**
   - LLM + 工具 + 重试逻辑不需要框架

❌ **你需要自定义行为**
   - 框架是固执的；定制很痛苦

❌ **你在构建生产系统并需要完全控制**
   - 调试、性能、安全都需要理解

❌ **"其他人都在用"是你唯一的理由**
   - 盲目跟风导致不必要的复杂性

## 更好的方法：从简单开始

这是我的建议：

### 1. 从直接 API 调用开始

```python
# 直接调用 LLM
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "你好"}]
)
```

先不要抽象。理解你在做什么。

### 2. 只在需要时添加需要的东西

需要重试逻辑？添加 tenacity。
需要日志？添加结构化日志。
需要工具调用？添加一个简单的工具注册表。

每个添加都应该解决你遇到的真实问题。

### 3. 只在有重复时才抽象

如果你写了三次相同的代码，那么抽象它。不要提前。

### 4. 保持可读性

你的代码应该能在 10 分钟内向新团队成员解释清楚。如果需要更长时间，你就过度工程化了。

### 5. 衡量依赖的成本

每个依赖都是一种负担：
- 安全漏洞
- 版本冲突
- 学习曲线
- 维护负担

确保收益大于成本。

## 我的简单代理式 AI 技术栈

对于生产环境的电信代理，这是我的完整技术栈：

**LLM 调用：**
```python
import openai  # 直接使用 SDK
```

**工具执行：**
```python
# 只是带类型提示的 Python 函数
def query_subscriber(phone: str) -> dict:
    """查询用户信息"""
    return db.query(phone)
```

**重试逻辑：**
```python
from tenacity import retry, stop_after_attempt
```

**日志记录：**
```python
import logging
import json

logger = logging.getLogger(__name__)

def log_agent_step(step, action, result):
    logger.info(json.dumps({
        "step": step,
        "action": action,
        "result": result,
        "timestamp": datetime.now().isoformat()
    }))
```

**测试：**
```python
import pytest

def test_agent_handles_missing_subscriber():
    agent = Agent(tools={"query": mock_query})
    result = agent.run("查找用户 555-0000")
    assert "未找到" in result.lower()
```

总依赖数：3（openai、tenacity、pytest）
总代码行数：约 500 行（包括测试）
总理解度：100%

## 元教训

这是 Sutton 原始文章精神的另一个痛苦教训。

我想变聪明。我想使用"正确"的工具。我想遵循最佳实践。我想让框架让我高效。

但现实教会了我：

- **简单胜过聪明**
- **理解胜过抽象**
- **直接胜过间接**
- **更少的依赖胜过丰富的生态系统**
- **你的代码胜过别人的框架**

痛苦的部分？承认我浪费了三周时间，因为我不相信自己能构建简单的东西。

解放的部分？意识到大多数问题不需要框架——它们需要清晰的思考和直接的代码。

## 采用框架前要问的问题

下次你被框架吸引时，问问自己：

1. **我能用 3 句话解释我的核心需求吗？**
   - 如果能，你可能不需要框架
   - 如果不能，先弄清楚你的需求

2. **我能在一天内构建一个最小版本吗？**
   - 如果能，先做那个
   - 如果不能，也许你需要框架——或者你的问题太复杂了

3. **我理解框架底层做了什么吗？**
   - 如果理解，你可能不需要它
   - 如果不理解，当出问题时你会很挣扎

4. **我使用了框架 >50% 的功能吗？**
   - 如果是，可能值得
   - 如果不是，你在为不用的东西付费

5. **我会向初级开发者推荐这个框架吗？**
   - 如果不会，它可能太复杂了
   - 如果会，确保是出于正确的原因

## 结论

框架承诺让你高效。有时它们确实做到了。但往往它们只是把复杂度从问题域转移到框架掌握上。

对于代理式 AI 系统，我学到了：

- **从简单开始**：LLM + 工具 + 重试逻辑
- **增量添加**：只在遇到真实问题时
- **理解一切**：如果你不能解释它，你就不能控制它
- **衡量成本**：依赖、性能、复杂性
- **保持灵活**：需求会变；框架不会

最好的框架往往根本不是框架——只是解决你特定问题的清晰代码。

你的情况可能不同。你的需求可能真的需要 LangChain 或 LangGraph。但要确保你选择框架是因为它解决了你的问题，而不是因为它流行或因为你认为你应该这样做。

## 你的经验是什么？

你有没有感觉被框架困住？你是坚持使用还是简化了？你选择代理式 AI 工具的方法是什么？

我很想听听你的故事——特别是如果你不同意我的方法。最好的教训来自多样化的经验。

---

**本系列下一篇：**"工具设计模式：让你的代理真正有用"

*在评论中分享你的框架经验。让我们从彼此的痛苦教训中学习。*
