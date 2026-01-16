---
layout: post
title: "Building Reliable Agentic AI Systems: Lessons from Production"
date: 2026-01-06 10:00:00 +0800
categories: [blog]
comments: true
tags: [Agentic AI, Production, Reliability]
---

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

