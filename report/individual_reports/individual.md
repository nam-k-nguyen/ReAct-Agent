# Individual Report: Lab 3 - Chatbot vs ReAct Agent
- **Student Name**: [Your Name Here]
- **Student ID**: [Your ID Here]
- **Date**: 2026-04-06
---
## I. Technical Contribution (15 Points)

- **Modules Implemented**: `src/tools/tools.py`, `src/agent/agent.py` (partial)

- **Code Highlights**:
```python
# src/tools/tools.py — Tool registry and dispatcher

TOOLS = [
    {
        "name": "web_search",
        "description": "Search for real-time information such as weather, ticket prices, hotels.",
        "input": "string — plain text query (no quotes)"
    },
    {
        "name": "calculator",
        "description": "Evaluate a mathematical expression and return the result.",
        "input": "string — math expression e.g. 150000 * 2 + 50000"
    },
    {
        "name": "get_system_time",
        "description": "Return the current date and time.",
        "input": "none"
    },
    {
        "name": "wikipedia_search",
        "description": "Look up background information about a location, event, or concept.",
        "input": "string — name of place, event, or concept"
    }
]

def execute_tool(tool_name: str, tool_input: str) -> str:
    # Sanitize input — strip quotes to prevent phrase-search issues (Bug #3 fix)
    tool_input = tool_input.replace('"', '').strip()

    if tool_name == "web_search":
        result = brave_search(tool_input)
        if not result or result == "No results found.":
            result = duckduckgo_search(tool_input)  # fallback
        return result

    elif tool_name == "calculator":
        return str(eval(tool_input))  # safe eval wrapper in production

    elif tool_name == "get_system_time":
        from datetime import datetime
        return datetime.now().strftime("%A, %d/%m/%Y %H:%M")

    elif tool_name == "wikipedia_search":
        return wikipedia_lookup(tool_input)

    return "Tool not found."
```

- **Documentation**: The `TOOLS` list acts as a declarative registry — each entry contains the tool name, description, and expected input format. This list is injected directly into the system prompt so the LLM knows which tools are available and how to call them. The `execute_tool` dispatcher routes the parsed `Action` output from the ReAct loop to the correct tool function. Crucially, all queries are sanitized (quotes stripped) before being sent to any search API — this directly addresses Bug #3. The DuckDuckGo fallback in `web_search` ensures that a failed Brave Search call does not terminate the agent's reasoning chain with an empty observation.

---

## II. Debugging Case Study (10 Points)

- **Problem Description**: In Agent v1, several sessions ended with `AGENT_END | steps: 0`, meaning the agent produced a Final Answer without calling any tool. On queries like *"Thời tiết Đà Nẵng cuối tuần này thế nào?"*, the response appeared plausible but was fabricated entirely from the model's training data.

- **Log Source** (snippet from `logs/2026-04-06.log`):

[SESSION_START] query: "Thời tiết Đà Nẵng cuối tuần này thế nào?"
[LLM_CALL] prompt_tokens: 412 | latency: 1,034ms
[AGENT_END] steps: 0 | has_answer: true | tool_calls: []


No `TOOL_CALL` or `TOOL_RESULT` events appear between `LLM_CALL` and `AGENT_END`.

- **Diagnosis**: The root cause is a hallucinated ReAct trace. Because GPT-4o was trained on many ReAct-format examples, it learned to complete the entire `Thought → Action → Observation → Final Answer` pattern in a single generation pass. When given the ReAct prompt format, the model would generate a realistic-looking `Action:` line, then immediately continue with a fabricated `Observation:` line drawn from its parametric memory — never pausing for an actual tool call. The agent's parsing loop saw `Final Answer:` and exited with `steps: 0`. This is the most insidious class of failure because the output looks correct and well-structured; the error is invisible without telemetry.

- **Solution**: Pass `stop=["\nObservation:"]` into `llm.generate()`. This forces the LLM to halt its generation the moment it would write the `Observation:` line, giving the agent loop control to call the real tool and inject the actual result. After this fix, all sessions that previously showed `steps: 0` moved to `steps: 2` or more, with proper `TOOL_CALL` and `TOOL_RESULT` events logged between each step.
```python
# agent.py — applying the stop sequence
response = self.llm.generate(
    prompt=current_prompt,
    system_prompt=self.system_prompt,
    stop=["\nObservation:"]   # ← key fix for Bug #1
)
```

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

1. **Reasoning**: The `Thought` block gave the agent an explicit scratchpad to decompose a user request before acting. For example, when asked about weekend weather, the agent first reasoned *"I need today's date to determine which weekend dates to search"* — a step a plain chatbot skips entirely. This sequential decomposition meant each tool call was purposeful and targeted, rather than a single, monolithic guess. The chatbot, having no such reasoning step, either answers from stale training data or produces a generic response with no grounding in the current date or real conditions.

2. **Reliability**: The agent actually performed *worse* than the chatbot in two clear scenarios. First, for simple factual or historical questions (e.g., geography, culture of a destination), the chatbot answered correctly in ~1,200ms at ~$0.0035, while the agent spent ~7,100ms and ~$0.020 doing a web search that returned no additional value. Second, the agent was vulnerable to timeout failure: one session exceeded `max_steps=7` on a multi-part itinerary query and returned a default fallback answer — worse than what the chatbot would have generated with its general knowledge.

3. **Observation**: The observation after each tool call acted as dynamic grounding — it updated the agent's context with real-world data that then directly shaped the next `Thought`. In the successful Đà Nẵng trace, the `get_system_time` observation (`Sunday, April 06, 2026`) changed the next query from a vague date to a specific one (`11-12/04/2026`). Without this feedback loop, the second search would have been imprecise. This feedback mechanism is what fundamentally separates the agent from the chatbot: the environment can *correct* the agent mid-task, whereas the chatbot commits to a single generation with no opportunity for self-correction.

---

## IV. Future Improvements (5 Points)

- **Scalability**: Refactor the agent loop using an async framework such as LangGraph or Python's `asyncio`. Currently the agent runs single-threaded, blocking on each LLM call and tool execution. An async queue would allow multiple user sessions to run concurrently, and parallel tool calls (e.g., fetching weather and ticket prices simultaneously) could reduce average latency significantly.

- **Safety**: Introduce a lightweight Supervisor LLM layer that reviews each `Action` before it is executed. The supervisor would check whether the proposed tool call is relevant to the user's original query, block any calls that attempt to access sensitive endpoints, and flag sessions where the agent appears to be looping or drifting off-topic. This adds a guardrail beyond the current `max_steps` hard limit.

- **Performance**: For systems with a large number of available tools (20+), replace the full tool list in the system prompt with a vector-database-backed tool retrieval step. At each `Thought` stage, embed the current context and retrieve only the top-k most relevant tools to include in the prompt. This keeps prompt size bounded, reduces token cost per call, and makes the architecture horizontally scalable as new tools are added.

---

> [!NOTE]
> Submit this report by renaming it to `REPORT_[YOUR_NAME].md` and placing it in this folder.
