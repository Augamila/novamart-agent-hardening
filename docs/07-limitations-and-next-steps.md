# 07 Limitations and next steps

## What the fix does not do

| # | Limitation | Consequence | Remedy |
|---|---|---|---|
| L1 | **It controls what the customer sees, not what the model can do.** Identity is still a model-supplied tool argument (F5). | A more persuasive injection could make the orchestrator pass another customer's ID to a data tool. The reply filter would not catch data that the CommunicationAgent legitimately formats into a reply. | Bind identity on the server: put the authenticated ID in a `contextvars` value at the entry point, and have every data tool reject a mismatched argument. In production, use an [AgentCore inbound JWT authorizer](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html) and take the ID from token claims. |
| L2 | **No prompt-attack detection.** The injection passes the guardrail. | Attacks reach the agents. | Enable the guardrail's [prompt-attack filter](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-prompt-attack.html) on input. Keep L1 anyway: guardrails reduce risk, while authorization has to be enforced in code. |
| L3 | **The marker filter is a denylist.** | Process narration without a marker passes ("I'll compute the answer with exact arithmetic…" is still in the deployed math reply). A legitimate sentence containing a marker would be removed. Variants such as "policy agent" with a space are not matched. | Short term: review the WARNING logs and extend the list from what is observed. Long term: **structured output.** The CommunicationAgent returns a typed object (greeting, answer, next steps), and the reply is assembled from those fields, so there is nowhere for narration to go. |
| L4 | **Tier claims and arithmetic presentation are prompt-only (F3, R1).** | Their reliability is unmeasured beyond single trials, and the baseline showed run-to-run variation. | Pass the tier into the CommunicationAgent's context only when the InventoryAgent ran, and check any tier word in the reply against it. Compute arithmetic in a tool and have the model only present the result. |
| L5 | **No grounding check (F4).** | Plausible, invented policy details can reach customers. | Run a [contextual grounding check](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html) on policy answers against the retrieved passages. Or use an LLM judge, but only once its agreement with human labels has been measured. |
| L6 | **Single trials; one adversarial prompt.** | Results show capability, not rates. | See next step 1. |
| L7 | **The fallback is generic.** | A genuine customer whose request the orchestrator mishandled gets a clarification request instead of help. | Count `fallback` events. If the rate on non-adversarial traffic is material, add a Strands hook that forces the CommunicationAgent call when the orchestrator skips it, keeping the fixed message for flagged adversarial inputs. |
| L8 | **Seeded data contains a conflict.** `customer_tiers.txt` gives Standard customers a 1-year electronics warranty; `warranty_policy.txt` says 2 years. | Answers depend on retrieval ranking. | Correct the source documents, make one authoritative, and add an evaluation question that targets the conflict. |

The full limitations register for the system (identity, memory use, IAM scope, audit trail, cost) is in the [system overview, §9](system-overview.md#9-limitations-and-how-to-address-them).

## Next steps, in priority order

1. **Turn the end-to-end harness into an evaluation set with repeated trials.**
   - Extend the eight prompts to around 50: injection variants (indirect, multi-turn, encoded, role-play), out-of-scope requests, and mixed-intent requests.
   - Run each *k* = 5 times and report pass^k per category.
   - Score replies automatically: the `reply_source` value, the regex markers, a tier-claim check against the seeded customer tier, and arithmetic recomputed in code.
   - Run it on every prompt or code change. This turns this project's one-off validation into regression testing (implication 3).
2. **Server-side identity binding plus the prompt-attack filter (L1, L2).** These close F5, the remaining High finding. Measure the injection category's pass^k before and after.
3. **Structured output for the CommunicationAgent (L3, L4).** Removes the narration class of failure by construction and lets the tier claim be checked against data.
4. **Grounding check on policy answers (L5).** Closes F4. Measure how often it rejects answers on the core policy questions, to keep false rejections low.
5. **CloudWatch metric filters and an alarm** on the `fallback` and "Removed … sentence(s)" warnings, with a dashboard of `reply_source` over time. This is the ongoing-monitoring half of the control.

Steps 1, 2 and 4 together form the first worked example planned in [agentic-model-validation](https://github.com/Augamila/agentic-model-validation): a validation harness and report for a deployed agent system.
