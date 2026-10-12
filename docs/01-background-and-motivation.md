# 01 Background and motivation

## The baseline system

[novamart-multi-agent](https://github.com/Augamila/novamart-multi-agent) is a customer-service system for a fictional retailer. Five agents cooperate on each request:

| Agent | Model | Role |
|---|---|---|
| OrchestratorAgent | Claude Haiku 4.5, temperature 0 | Classifies the request, routes to specialists, manages shared state. Its prompt says it **never writes the customer reply**. |
| InventoryAgent | Claude Sonnet 4.5 | Reads the order and the customer's tier from DynamoDB |
| RefundAgent | Claude Sonnet 4.5 | Decides return eligibility and initiates the return |
| PolicyAgent | Claude Sonnet 4.5 | Multi-agent RAG: three retriever agents query the Returns, Shipping and Warranty Knowledge Bases (Bedrock KB on S3 Vectors) in parallel |
| CommunicationAgent | Claude Sonnet 4.5 | Writes the customer-facing reply from the facts the other agents recorded |

The agents share no conversation history. They coordinate through a WorkflowState record in DynamoDB, protected by optimistic locking. The system runs on Amazon Bedrock AgentCore Runtime behind a Bedrock Guardrail, with AgentCore Memory provisioned and every request traced in CloudWatch and X-Ray. The [system overview](system-overview.md) has the full architecture.

## What had already been verified

By the start of this work the baseline had passed a substantial set of checks:

- **A 120-point automated suite** (`tests/test_agent.py`) runs against live AWS resources. It covers agent construction and model assignment, parallel retrieval, guardrail policies and versioning, runtime status and environment, memory configuration, Knowledge Base status, embeddings and syncs, and logging and tracing. It scored 120/120.
- **Three core scenarios**, a return, a policy question and an arithmetic question, routed and answered correctly both locally and through the deployed runtime.
- **A guardrail validator** (`tools/validate_guardrail.py`) passes 13 inputs through the published guardrail version with the ApplyGuardrail API. All 12 scored cases behaved as expected, and one known gap (prompt injection) was documented.
- **An independent human review** of the code, configuration and evidence. The reviewer specifically praised the deterministic refund checks, the history reset between customers, the sequential tool executor and the deploy-time configuration checks.
- **Several defects had already been found and fixed during the build.** Among them was a race condition between concurrent tool calls, which X-Ray exposed and which `SequentialToolExecutor` plus race-tolerant state creation resolved.

On any normal acceptance standard, the system was done.

## Why that was not enough

Every check above shares a blind spot: it **verifies what was specified, by the method the specifier chose.**

- The suite checks *structure*: that agents exist, use the right models and are wired to the right resources. It does not read a single customer reply.
- The core scenarios check *happy paths*: three well-formed requests that the system was built to handle.
- The guardrail validator checks the guardrail *in isolation*. It shows the policies are configured correctly, not that the deployed system, with five models between the customer and the guardrail's output, behaves correctly end to end.
- The human review checks the *evidence it is given*.

None of them asks the question a model validator asks first: **what does the system actually do when it is used in a way nobody planned for, and what does the customer see?**

That question matters more for agentic systems than for traditional models. A credit model produces a number from a fixed set of inputs. An agent system produces free text, chooses its own sequence of actions, and relies on instructions written in natural language. In this system the most important behavioural rule, that only the CommunicationAgent speaks to the customer, was written as one sentence in the orchestrator's prompt:

> *CRITICAL: You must NEVER write the customer-facing response yourself.*

Nothing in the code enforced it. The reply path in the runtime was:

```python
text = _strip_xml_tags(state.get('communication_agent', '') or str(response))
```

If the CommunicationAgent wrote nothing, whatever the orchestrator said went to the customer. In the graded scenarios the orchestrator always followed its instruction, so the fallback never fired and nothing showed the gap.

## Objective

To answer three questions with evidence from the deployed system:

1. Does the guardrail behave the same end to end as it does in isolation?
2. Does the system hold its own behavioural rules under inputs it was not graded on?
3. Where it does not, can the rule be moved from instruction to control without breaking anything that already worked, and can that be shown in production?

[02 Method](02-method.md) describes how.
