# 06 Implications

This is a small system with seeded data, a single adversarial prompt and single trials. The implications below are stated at the strength that evidence supports. They are not claims about agent systems in general. They are patterns this project shows concretely, and that are consistent with what supervisors and security bodies now say about agentic AI.

## 1. For agent design: a rule in a prompt is a control *objective*, not a control

The baseline had one important behavioural rule ("only the CommunicationAgent speaks to the customer") and enforced it in one place, a sentence in the orchestrator's prompt. It held in every graded scenario and failed on the first adversarial input. The hardened build then showed the same pattern one level down. The CommunicationAgent, explicitly told not to describe the question or how the reply was produced, did exactly that on the next run.

The contrast with the rest of the system is the lesson. The controls that already lived in code all held in production:
- `initiate_refund` re-checks the policy and writes conditionally;
- identifiers are validated;
- the tool executor is sequential.

The rules that lived only in prompts did not hold reliably.

**Design rule that follows:** for every behavioural requirement, ask *where it is enforced*.
- If the answer is "the prompt says so", the prompt is a mitigation that lowers frequency. It is not a control.
- Requirements whose failure reaches the customer, data or money need a deterministic check at the boundary where that harm would occur. Here that was the reply path.
- Where the check can't be made deterministic (tier claims, grounding), its failure should at least be measured.

## 2. For testing: test what the acceptance criteria don't grade

Four independent checks passed the baseline: a 120-point automated suite, the core scenarios, the guardrail validator and a human review. Every finding in this project came from outside their scope:

| Finding | Why each check missed it |
|---|---|
| F1 orchestrator speaks | The suite checks structure; the core scenarios never skip the CommunicationAgent; the guardrail let the input through by design |
| F2 narration | No check reads reply text for internal vocabulary |
| F3 tier claim | It is in the reviewed log; a reader checking "explained the premium policy?" marks it correct |
| F4 invented benefit | Plausible-sounding, and grounding is not checked |

Acceptance tests are written to show that a specification was built. Validation asks what the system does outside the specification. For agents the outside is much larger than for traditional models: open-ended input, chosen action sequences and free-text output. Concretely:
- adversarial and out-of-scope inputs belong in every evaluation set;
- someone has to **read the outputs**;
- evaluation must cover the deployed system, not only its parts. The guardrail was right in isolation *and* end to end, but the end-to-end view is where F1 appeared.

## 3. For change management: a prompt edit is a model change

The first fix reworded one instruction to stop F2. It also changed how the model presented arithmetic (R1). Rewording a prompt changed behaviour the edit was not about.

Prompt text is model logic, and it should go through the same change controls as code:
- **version it** (it is in the source file here, so the diff shows every change);
- **run the regression set on every change**, including scenarios the change is not aimed at;
- **compare outputs with the previous version, not just with expectations.** R1 was found by comparing v1 with the baseline, not by any failing test.

## 4. For evaluation: one pass is not evidence

Two runs of the *unchanged* baseline disagreed. One showed consistent arithmetic ($14.995); the other said "you save $15.00" next to a total of $134.96. The hardened build's results are also single trials. The 8/8 end-to-end result shows that the hardened system *can* behave correctly on those inputs, and that the deterministic parts *will*. It does not show how often the probabilistic parts do.

The deterministic control changes what a single trial can establish:
- `final_customer_reply()` sits in code, so "the orchestrator's text never reaches the customer" holds on **every** trial, not just this one;
- the tier rule and the arithmetic rule are prompt-only, so they need repeated trials (pass^k) before their reliability can be stated.

This is the practical reason to move rules into code wherever possible: it makes them *testable* at a known strength.

## 5. For operations: make exception paths observable

Before the fix, F1 and F2 were invisible. They produced fluent replies, no errors, and green traces: every edge on the service map was Ok 100%. Tracing showed the system *ran* correctly, not that it *behaved* correctly. The hardened build makes each exception a countable event:
- `reply_source` on every response;
- a WARNING for each fallback and each sentence removed.

A CloudWatch metric filter on those lines gives a monitoring signal ("prompt-rule violations per thousand conversations") with thresholds and alerts. That is ongoing monitoring of the model's behaviour, not just of the infrastructure.

---

## Implications for model risk and governance

These patterns map onto how agentic AI is starting to be governed. My companion project, [agentic-model-validation](https://github.com/Augamila/agentic-model-validation), sets out an Agentic Validation Framework (AVF) with eight validation dimensions. This exercise is a small, worked instance of several of them:

| AVF dimension | What this project shows |
|---|---|
| **D2 Task outcomes** | Correct answers alone (the core scenarios passed) do not mean correct behaviour |
| **D3 Trajectory and tool use** | F1 is a *trajectory* failure: the orchestrator skipped a required step. Only the path, not the final answer, reveals it. |
| **D4 Groundedness** | F3 and F4: fluent, plausible statements unsupported by retrieved or looked-up facts |
| **D5 Robustness and security** | F5: the injection was stopped by model behaviour, not a control ([OWASP ASI01 Agent Goal Hijack](https://genai.owasp.org/)). F1 turned a refusal into an information leak. |
| **D6 Consistency** | Two baseline runs disagreed; single-trial results are labelled as such |
| **D8 Containment** | `final_customer_reply()` is a containment control: whatever the agents do, what leaves the system is bounded |

**Effective challenge.** The baseline had been reviewed and accepted. The findings came from re-testing it with a different question: not "does it meet the specification?" but "what does it do that the specification did not anticipate?". That is the purpose of independent challenge in model risk management. US supervisory guidance on model risk ([SR 26-2](https://www.federalreserve.gov/supervisionreg/srletters/SR2602.htm), which superseded SR 11-7 in April 2026) places generative and agentic AI outside its formal model definition, but not outside risk-management expectations. Other regimes take a broader view: OSFI's Guideline E-23 covers AI and machine-learning models, and the MAS guidelines on AI risk management name AI agents explicitly. Whatever the label, this exercise is the kind of evidence those expectations ask for: findings with severity, root cause, remediation, and verification after the change.

**Controls testing.** An internal auditor would describe F1 as a *design deficiency*: the control (a prompt instruction) was not capable of meeting its objective under foreseeable conditions. It was not an *operating* failure of a sound control. The remediation replaces the design. It does not ask the existing control to try harder.

## Applying the lesson in other domains

The same structure, specialist agents behind an orchestrator with one agent writing to the user, appears across the use cases in [agentic-model-validation](https://github.com/Augamila/agentic-model-validation/blob/main/docs/03-use-cases.md). In each, F1–F4 have a more serious equivalent:

| Domain | Equivalent of F1 (orchestrator speaks) | Equivalent of F3/F4 (unverified or invented facts) |
|---|---|---|
| Bank disputes (Reg E) | Internal routing or fraud-scoring logic disclosed to a disputing customer | Telling a customer a provisional credit applies when eligibility was not checked |
| Insurance claims (FNOL) | Claims-triage reasoning or SIU flags exposed to a claimant | Stating coverage that the policy wording does not support |
| Financial advice (Reg BI) | Internal suitability scoring shown to a client | Asserting a client's risk profile or account type without looking it up |
| AML alert triage | An agent's case reasoning leaked into an external communication (tipping-off risk) | A narrative that states facts not present in the transaction record |

In regulated settings, the fixed-fallback-plus-filter pattern is a minimum, not an end state. The stronger forms are structured outputs, grounding checks on every factual claim, and human review for high-impact replies. They belong in the design from the start, and [07](07-limitations-and-next-steps.md) sets them out for this system.
