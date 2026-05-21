# Cost Management for Agentic Architecture

## 1. LLM Selection and Model Routing

### Key challenges

**Overusing the most expensive model.**
Teams often default every task to the strongest model, even when many agent steps only require classification, extraction, summarization, routing, or lightweight reasoning.

**No clear model-performance benchmark.**
Without task-specific evaluations, teams cannot confidently answer: “Can a smaller model do this step well enough?” Microsoft Foundry’s observability guidance specifically recommends evaluating models for quality, safety, reliability, and task performance during model selection. ([Microsoft Learn][1])

**Model routing can become unpredictable.**
If the router is too dynamic, costs and latency can fluctuate. If the router is too simple, quality suffers because hard tasks go to weak models and easy tasks go to expensive ones.

**Quota and rate-limit constraints.**
Azure OpenAI quotas are scoped at the Azure subscription level, and throughput depends on request-per-minute and token-per-minute constraints. Poor routing can overload one deployment while other models sit idle. ([Microsoft Learn][2])

**No visibility into model-level cost.**
If traces do not capture model name, input tokens, output tokens, latency, retries, and tool calls, you cannot see which model, agent, user, tenant, or workflow is driving spend.

### Best practices

**Use a model portfolio, not one model.**
Separate models by role:

| Agent task               | Recommended model strategy                              |
| ------------------------ | ------------------------------------------------------- |
| Intent detection         | Small / fast model                                      |
| Query rewriting          | Small or mid-tier model                                 |
| RAG answer generation    | Mid-tier or strong model depending on risk              |
| Complex planning         | Strong reasoning model                                  |
| Tool selection           | Mid-tier model with strong function-calling reliability |
| Final user-facing answer | Stronger model when quality matters                     |
| Guardrail checks         | Small classifier or policy model where possible         |

**Create a routing policy.**
Use rules such as:

```text
If task = classification → small model
If task = extraction from structured text → small or mid model
If task = multi-step reasoning → reasoning model
If user tier = premium → higher quality model allowed
If latency budget < 2 seconds → fast model only
If cost budget exceeded → downgrade or ask clarification
If confidence < threshold → escalate to stronger model
```

**Use progressive escalation.**
Start with a cheaper model. Escalate only when needed:

```text
Small model → Medium model → Strong reasoning model → Human review
```

**Route by risk, not just complexity.**
A simple question in a regulated domain may require a stronger model or stricter validation than a complex but low-risk creative task.

**Measure model quality per task.**
Use task-specific evaluations: answer correctness, groundedness, tool-call accuracy, latency, refusal quality, safety, and cost-per-successful-task. Foundry includes built-in evaluators for quality, RAG groundedness, safety, agent tool-call accuracy, and task completion. ([Microsoft Learn][1])

**Track cost per business outcome.**
Do not only track cost per request. Track:

```text
Cost per resolved customer case
Cost per completed booking
Cost per generated report
Cost per successful agent task
Cost per escalation avoided
```

**Use provisioned throughput for predictable production workloads.**
For steady production traffic with known latency and throughput requirements, Azure provisioned throughput can provide allocated capacity, predictable performance, and reservation-based cost advantages. ([Microsoft Learn][3])

---

## 2. Observability, Tracing, and Diagnostics

### Key challenges

**Agent behavior is multi-step and non-deterministic.**
A single user request may involve planner calls, tool calls, retrieval, memory lookup, validation, retries, and final response generation. Without tracing, it is hard to know where quality, cost, or latency problems came from.

**Traditional monitoring is not enough.**
CPU, memory, and HTTP status codes do not explain why an agent made a bad decision, called the wrong tool, retrieved the wrong document, or burned 50,000 tokens.

**Token usage is often invisible.**
For agents, request count is a weak cost signal. Token usage, number of model calls, number of tool calls, retries, and context size are much better indicators.

**Tracing itself can create cost.**
Full traces, verbose logs, prompt capture, and long retention windows can increase Azure Monitor or Application Insights costs. Microsoft notes that Foundry monitoring itself has no extra charge, but tracing depends on Application Insights and Azure Monitor Logs pricing. ([Microsoft Azure][4])

**Security and privacy risks.**
Traces may contain prompts, user data, retrieved documents, tool arguments, API responses, or sensitive business context.

### Best practices

**Instrument every agent step as a trace span.**
At minimum, capture:

```text
User request ID
Conversation ID
Agent name
Task name
Model name
Provider
Prompt version
Input tokens
Output tokens
Total tokens
Latency
Tool name
Tool arguments metadata
Tool result status
Retrieval query
Retrieved document IDs
Retry count
Error type
Final outcome
Cost estimate
```

Microsoft Foundry tracing is built on OpenTelemetry and captures LLM calls, tool invocations, agent decisions, and service dependencies. It also integrates with Azure Monitor Application Insights. ([Microsoft Learn][1])

**Use OpenTelemetry GenAI semantic conventions.**
OpenTelemetry defines GenAI metrics for LLM calls, function calls, and other operations in larger AI workflows. This helps standardize model, provider, token, latency, and operation telemetry across frameworks and vendors. ([Microsoft Azure][5])

**Create dashboards by agent workflow.**
A useful production dashboard should show:

| Dashboard metric         | Why it matters                       |
| ------------------------ | ------------------------------------ |
| Cost by agent            | Identifies expensive agents          |
| Cost by tenant/user      | Detects abusive or high-volume usage |
| Cost by model            | Shows model spend distribution       |
| Tokens per request       | Detects context bloat                |
| Latency by step          | Shows bottlenecks                    |
| Tool-call failure rate   | Finds broken dependencies            |
| Retry rate               | Reveals instability and hidden cost  |
| Escalation rate          | Shows where cheaper models fail      |
| Quality/evaluation score | Connects spend to actual value       |
| Groundedness score       | Detects RAG failures                 |
| Safety violation rate    | Tracks policy risk                   |

**Sample traces instead of logging everything forever.**
Use full tracing in development, staging, and high-risk production workflows. In production, use adaptive sampling: keep all failed, high-cost, slow, or low-quality traces, and sample routine successful traces.

**Redact sensitive content before export.**
Use the telemetry pipeline to remove or hash secrets, PII, raw prompts, raw completions, and sensitive tool outputs before sending data to external observability tools.

**Correlate AI traces with app traces.**
A slow agent may actually be caused by search latency, database latency, network retries, or downstream API failures. Distributed tracing should connect the user request, app backend, retriever, tool calls, LLM calls, and response.

**Set alerts on cost and behavior anomalies.**
Useful alerts include:

```text
Daily cost exceeds budget
Cost per task exceeds threshold
Input tokens spike
Output tokens spike
Retry rate increases
Tool failures increase
Latency P95 exceeds target
Groundedness score drops
Unsafe response rate increases
Fallback-to-expensive-model rate increases
```

---

## 3. Cost, Latency, and Performance Optimization

### Key challenges

**Cost and latency are tightly connected.**
Long prompts, large retrieved contexts, unnecessary tool calls, retries, and verbose responses increase both latency and spend.

**Agents multiply LLM calls.**
A chatbot might use one model call. An agentic workflow may use 5, 10, or 30 calls across planner, worker, critic, memory, tools, and final response.

**Context windows create hidden waste.**
Larger context windows make it easy to pass too much history, too many documents, or repeated instructions.

**Retries can silently explode cost.**
Automatic retries are useful, but repeated model calls, tool calls, and retrieval calls can create major hidden spend.

**Throughput is not just quota.**
Azure OpenAI latency guidance notes that quota affects admission logic, but per-call latency variation means actual throughput may be lower than the theoretical quota. ([Microsoft Learn][6])

### Best practices

**Optimize prompt size.**
Remove repeated instructions, compress conversation history, summarize old context, and pass only the minimum data needed for the current step.

**Control retrieval size.**
Do not blindly pass top-20 documents into the model. Use:

```text
Top-k tuning
Semantic reranking
Document chunk filtering
Metadata filters
Context compression
Citation-aware answer generation
```

**Track cost per agent step.**
Break down spend like this:

```text
Planner call: $X
Retriever call: $Y
Tool calls: $Z
Validation call: $A
Final response call: $B
Total task cost: $T
```

This makes it clear whether the expensive part is planning, retrieval, tool execution, retries, or final generation.

**Set token budgets.**
Create hard budgets per workflow:

```text
Max input tokens per request
Max output tokens per response
Max total tokens per workflow
Max number of LLM calls
Max number of retries
Max number of retrieved chunks
Max number of tool calls
```

**Use caching aggressively.**
Cache stable system prompts, policy instructions, retrieved reference data, tool results, embeddings, search results, and deterministic intermediate outputs.

**Stream responses for perceived latency.**
Streaming does not always reduce total completion time, but it improves user experience because the user sees progress earlier.

**Parallelize independent work.**
If an agent needs flight, hotel, car rental, and restaurant options, run independent specialist agents or tools in parallel rather than serially.

**Avoid unnecessary agent loops.**
Many workflows do not need autonomous planning. Use deterministic orchestration for predictable steps and reserve agentic reasoning for ambiguous or complex decisions.

**Use smaller models for internal steps.**
A common pattern:

```text
Small model: classify, route, extract
Medium model: synthesize, call tools, rewrite queries
Large/reasoning model: plan, resolve conflicts, handle complex cases
```

**Use SLOs for cost and latency.**
Define service-level objectives such as:

```text
P95 latency under 5 seconds
Average cost per task under $0.05
P95 cost per task under $0.20
Tool failure rate under 1%
Retry rate under 3%
Groundedness score above threshold
Task completion above threshold
```

**Continuously evaluate cost vs. quality.**
A cheaper model is not better if it causes retries, bad answers, support tickets, or human escalation. Measure **cost per successful outcome**, not just cost per token.

---

## Recommended operating model

For production AI agents, we would manage these three areas together as one loop:

```text
1. Observe
   Capture traces, tokens, latency, tool calls, model choice, errors, and quality signals.

2. Diagnose
   Identify expensive workflows, slow steps, bad routes, failed tools, and poor prompts.

3. Optimize
   Reduce context, improve prompts, cache, route to smaller models, parallelize, and tune retrieval.

4. Evaluate
   Confirm that quality, groundedness, safety, and task completion did not regress.

5. Govern
   Enforce budgets, model policies, rate limits, retention rules, and escalation paths.
```

The key principle is this:

**You cannot control agent cost by looking only at cloud spend. You control cost by observing every reasoning step, every model call, every token, every tool call, every retry, and every failed outcome.**

[1]: https://learn.microsoft.com/en-us/azure/foundry/concepts/observability "Observability in Generative AI - Microsoft Foundry"
[2]: https://learn.microsoft.com/en-us/azure/foundry/openai/quotas-limits "Azure OpenAI in Microsoft Foundry Models Quotas and Limits"
[3]: https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/provisioned-throughput "What Is Provisioned Throughput for Foundry Models?"
[4]: https://azure.microsoft.com/en-us/pricing/details/foundryobservability "Observability in Foundry Control Plane - Pricing"
[5]: https://azure.microsoft.com/en-us/products/devops "Azure DevOps"
[6]: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/latency= "Performance and latency - Azure"
