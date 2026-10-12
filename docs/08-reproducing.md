# 08 Reproducing

> **Cost.** Nothing in the base stack bills by the hour (DynamoDB on demand, S3, S3 Vectors). Costs come from Bedrock model calls, Knowledge Base retrieval and AgentCore Runtime invocations. Tear the stack down when you finish.

## Prerequisites

- Python 3.12 (the runtime uses `PYTHON_3_12`)
- AWS CLI v2 with credentials for `us-east-1`
- Node.js 20+, [uv](https://docs.astral.sh/uv/), and the AgentCore CLI: `npm install -g @aws/agentcore@0.30.0`
- Bedrock model access to Claude Haiku 4.5, Claude Sonnet 4.5 and Titan Text Embeddings V2

**Use one credential source per shell.** If you use temporary session keys, remove `AWS_PROFILE` first (`Remove-Item Env:AWS_PROFILE` in PowerShell). Python and the AWS CLI prefer the keys, but the Node.js SDK behind the AgentCore CLI and CDK prefers the profile, so a stale profile makes the deploy fail while everything else works ([05](05-results.md#the-deployment-incident)).

## Build and deploy

```powershell
# 1. Environment
py -3.12 -m venv venv; .\venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:PYTHONUTF8 = "1"
[Console]::OutputEncoding = [Text.Encoding]::UTF8; $OutputEncoding = [Text.UTF8Encoding]::new()

# 2. Foundation infrastructure (DynamoDB, S3, S3 Vectors, IAM, log group) and seed data
aws cloudformation deploy --template-file infrastructure\starter_stack.yaml `
  --stack-name udacity-agentcore --capabilities CAPABILITY_NAMED_IAM --region us-east-1
python infrastructure\seed_data.py
Copy-Item .env.example .env

# 3. Knowledge Bases (console): three "Unstructured Vector Store" KBs, Titan V2 / 1024 / float,
#    the existing S3 Vectors bucket with a matching index, data-source prefixes
#    policies/returns/, policies/shipping/, policies/warranty/. Sync each and put the IDs in .env.
#    `python config.py` prints the bucket and index names.

# 4. Guardrail check, then deploy
python tools\validate_guardrail.py
python src\agent_orchestrator.py deploy          # put the printed runtime ARN and guardrail ID/version in .env
```

## Verify

```powershell
python tests\test_agent.py all                   # expect 120/120
python src\agent_orchestrator.py test            # three core scenarios; each line shows Response [communication_agent]
.\tools\run_guardrail_e2e.ps1                    # eight prompts through the deployed runtime + ApplyGuardrail
```

Expected `reply_source` values from the end-to-end test:
- the two control prompts: `communication_agent`;
- the five blocked prompts: `guardrail`;
- the injection prompt: `fallback`.

Because model output varies, run the end-to-end test several times before drawing conclusions about the prompt-dependent behaviours ([02](02-method.md#limits-of-the-method)).

To see the control's warnings in CloudWatch, search the agent log group for the two WARNING messages. If they are not there, search the runtime's own log group under `/aws/bedrock-agentcore/runtimes/`:

```powershell
aws logs filter-log-events --log-group-name /aws/bedrock/agentcore/udacity-agentcore `
  --filter-pattern '?"Removed" ?"without the CommunicationAgent"' --max-items 20 --query "events[].message"
```

## Tear down

```powershell
python infrastructure\cleanup.py                                   # dry run
python infrastructure\cleanup.py --yes --disable-transaction-search
# wait 5-10 minutes (Knowledge Base and AgentCore deletions are asynchronous)
python infrastructure\cleanup.py --yes
```

The resource names (`udacity-agentcore`, `udacity_agentcore_runtime`, …) come from the course scaffold's `config.py` and are kept unchanged.
