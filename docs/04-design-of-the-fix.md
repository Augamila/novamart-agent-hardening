# 04 Design of the fix

## Design goal

Make "only the CommunicationAgent speaks to the customer, and never in internal terms" a property of the **code**, not just of the prompt. That has to happen without:
- hiding legitimate answers;
- swallowing the guardrail's own refusal messages;
- changing the agent architecture the rest of the system depends on.

The fix has three parts:
1. a single function that decides the final reply;
2. a deterministic filter for internal text;
3. two prompt changes for issues that code cannot easily check.

## 1. `final_customer_reply()`: one path to the customer

Before the fix, three entry points each picked the reply in their own way:
- `run_serve()` (the runtime) and `run_chat()` used `communication_agent or str(response)`;
- `run_test_scenarios()` printed `str(response)`.

All three now call one function:

```python
def final_customer_reply(response, state: dict | None) -> tuple[str, str]:
    state = state or {}
    raw = str(response).strip()
    comm = _strip_xml_tags(state.get('communication_agent', '') or '').strip()

    if comm:
        text, source = comm, 'communication_agent'
    elif (getattr(response, 'stop_reason', None) == 'guardrail_intervened'
          or raw in (GUARDRAIL_BLOCKED_INPUT_MESSAGE, GUARDRAIL_BLOCKED_OUTPUT_MESSAGE)):
        return raw, 'guardrail'
    else:
        logger.warning("Orchestrator produced a reply without the CommunicationAgent ...")
        return FALLBACK_REPLY, 'fallback'

    cleaned, removed = _remove_internal_sentences(text)
    if removed:
        logger.warning("Removed %d sentence(s) containing internal markers ...", removed, ...)
    if not cleaned:
        return FALLBACK_REPLY, 'fallback'
    return cleaned, source
```

The three branches follow how the system actually works:

**Branch 1, the CommunicationAgent wrote a reply.** This is the normal path. The CommunicationAgent writes its output to WorkflowState through `_run_worker()`, so the state record, not the orchestrator's final message, is the source of truth for what the customer should see. This also covers the case where the orchestrator adds commentary around the CommunicationAgent's text: the commentary is discarded.

**Branch 2, the guardrail intervened.** When the guardrail blocks the *input*, it stops the orchestrator's first model call, no worker runs, and the orchestrator's result *is* the guardrail's configured message. Branch 3 would replace that message with the generic fallback. That would be safe but wrong, because the guardrail's message is the designed response, and it tells the customer what the system *can* help with. Two independent signals detect this case:
- the Strands stop reason `guardrail_intervened`;
- an exact match against the configured messages.

To keep the exact-match test reliable, the messages are now module constants (`GUARDRAIL_BLOCKED_INPUT_MESSAGE`, `GUARDRAIL_BLOCKED_OUTPUT_MESSAGE`), and `create_guardrail()` configures the guardrail from those same constants. The text exists in one place.

**Branch 3, anything else.** The orchestrator produced text without the CommunicationAgent, which is exactly finding F1. The customer gets a fixed, neutral clarification:

> I'm sorry, I wasn't able to complete that request. I can help with your NovaMart orders, returns, refunds, shipping or warranty questions - could you tell me a little more about what you need?

It gives no explanation of *why*, no mention of rules, and no identifiers. An attacker learns nothing from it, and a confused genuine customer is pointed back to what the service does.

**Observability.** The function returns the reply and its source. The runtime adds `reply_source` to its response, and `invoke`, `test` and `chat` print it. Every `fallback` and every removed sentence is logged as a WARNING, and the logs go to CloudWatch. That makes the control **measurable**: a metric filter on these warnings gives a rate of prompt-rule violations over time, which is the ongoing-monitoring signal a validator would want.

## 2. The internal-marker filter

Even on the normal path, the CommunicationAgent can narrate its pipeline (finding F2). The filter removes any *sentence* that contains a marker which appears only when internal text leaks:

```python
_INTERNAL_MARKERS = re.compile(
    r"agent findings|\bsession id\b|\bcustomer id\b|route_to_\w+|initialize_session|"
    r"workflow ?state|workflow context|routing rules|specialist agent|prompt injection|"
    r"\b(?:orchestrator|inventory|policy|refund|communication)agent\b",
    re.IGNORECASE)
```

Design choices:
- **Sentence-level removal, not reply-level rejection.** In the observed failures the leak was one framing sentence around a correct answer. Removing the sentence keeps the answer, and if nothing is left, the fallback is used.
- **Line structure is preserved.** The text is split by line, then by sentence, so lists and paragraphs survive.
- **A denylist, deliberately narrow.** Every marker is a term a customer reply has no reason to contain. The list was built from what was actually observed (F1, F2) plus the system's own identifiers (agent and tool names).
- **Every removal is logged**, so the filter can't silently change replies without trace.

Known weaknesses, accepted and documented in [07](07-limitations-and-next-steps.md):
- It catches *markers*, not meaning. "I'll compute the answer with exact arithmetic…" is process narration with no marker, and it passes.
- A legitimate sentence could contain a marker. For example, "Your customer ID is CUST-001" would be removed, which is a false positive. In this system no agent is asked to tell customers their ID.
- "policy agent", written with a space, is not matched; only the class names are.

## 3. Prompt changes

Some behaviours are hard to enforce with a regex but easy to state. These went into the CommunicationAgent's prompt, and they are treated as **mitigations, not controls**:

| Change | Addresses | Baseline wording → new wording |
|---|---|---|
| No internal vocabulary in the instruction itself | F2 | "For calculation questions **(no agent findings)**…" → "For calculation questions (the context holds no order, policy or refund details)…" |
| No narration | F2 | added: "never describe what kind of question this is or how the reply was produced. Start directly with the answer." |
| Unrounded intermediate values | R1 | added in v2: "show the short working with every intermediate value unrounded (e.g. a discount of $14.995, not $14.99). Round currency to cents only in the final total (half up)." |
| No tier claims without data | F3 | added in v2: "Never state or imply the customer's tier (Standard or Premium) unless the InventoryAgent findings include it. For general policy questions, describe the terms for each tier without saying which one the customer is." |

**The rewrite repeated the original mistake.** The v2 calculation instruction still describes the situation in pipeline terms ("the context holds no order, policy or refund details"). On the hardened build, the model echoed it almost word for word, "a calculation question with no order, policy, or refund details in the workflow context". The filter caught it because of "workflow context". The lesson from F2 applies to the fix as well: any wording in a prompt can surface in the output.

## Alternatives considered

| Alternative | Why not chosen |
|---|---|
| **Strengthen the orchestrator prompt only** | This is what failed. A stronger instruction lowers the rate of the failure; it does not remove it. The hardened build itself shows the CommunicationAgent ignoring an explicit "never describe" instruction. |
| **Force a call to the CommunicationAgent when the orchestrator skips it** (a Strands hook that checks the tool sequence) | Better for the routing guarantee and recorded as limitation #8. It needs a second model call on the failure path and changes the agent loop. For an adversarial input, a fixed message is also *preferable* to letting another model compose a reply to the attack. Worth doing as well, not instead. |
| **Structured output** (the CommunicationAgent returns a typed `CustomerReply` object with only allowed fields) | The complete fix for narration and the right long-term design (limitation #7). It is a larger change to the agent and its prompt, and it needs its own regression testing. The filter is the minimal control that could be verified and deployed now. |
| **An LLM judge to screen replies** | It adds latency and cost and is itself probabilistic. A judge needs a measured agreement rate before it can be relied on. It is better suited to offline evaluation and sampling than to a synchronous gate. |
| **Output guardrail with a denied topic for "internal system details"** | Plausible, but topic classification is semantic and probabilistic, and it would replace the entire reply rather than one sentence. The filter is deterministic and explainable. |

## What did not change

The architecture, models, tools, routing rules, guardrail configuration, state handling and refund controls are unchanged. The fix sits entirely at the boundary between the system and the customer, which is where a control on *what the customer sees* belongs. That is why the 120-point suite, which checks the architecture, was unaffected.

## Other changes in the diff

These are not part of the fix, but they ship in the same build:
- **Expanded docstrings on the three retriever tools.** They state the Knowledge Base, the indexed documents, the topics covered and the empty-result behaviour. In Strands, a tool's docstring *is* the description the model sees, so this is a small change to model-visible text.
- **Two unreachable `NotImplementedError` guards** left from the scaffold are replaced. They now raise `RuntimeError`s that name the failing step and the next action.
- **Section headers and comments** no longer refer to course task numbers.
