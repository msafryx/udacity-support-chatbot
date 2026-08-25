# Customer Support Chatbot — Amazon Bedrock AgentCore

A customer support chatbot for a fictional online shop, built on the **Amazon Bedrock AgentCore managed harness**. All routing, information-gathering, and grounding behavior lives in a single system prompt. The harness supplies the agent loop (model calls, stateful sessions, tool execution); the prompt supplies the behavior.

## Routes

The assistant classifies every incoming message into exactly one of three routes:

- **Bug report** — collects the bug *description*, *steps to reproduce*, and *environment* across the conversation (one question at a time), then files a ticket by calling the `create_bug_report` tool. The tool is a Lambda function exposed through an AgentCore Gateway that writes the ticket to DynamoDB. The customer is given the ticket ID.
- **Platform question** — answered **only** from the FAQ embedded in the prompt (orders, shipping, returns, payments, products, accounts, privacy). If the FAQ doesn't cover it, the assistant hands off to human support.
- **Other** — anything else is politely redirected to the human support line.

## Tech stack

- Amazon Bedrock AgentCore managed harness — runs the chatbot
- Amazon Bedrock AgentCore Gateway — exposes the bug-report Lambda as a tool
- AWS Lambda + Amazon DynamoDB — bug-report tool runtime and storage
- Amazon Bedrock Evaluations — LLM-as-a-judge quality evaluation
- Model: `us.amazon.nova-pro-v1:0`, region `us-east-1`

## Key files

| File | Description |
|------|-------------|
| `system_prompt.txt` | Main deliverable — the system prompt that defines all routing and behavior |
| `online_shop_faq.md` | FAQ embedded into the prompt at build time (replaces the `{{FAQ}}` placeholder) |
| `cloudformation-tool.yaml` | Deploys the DynamoDB table, the `create_bug_report` Lambda, and the gateway/harness IAM roles |
| `cloudformation-testing.yaml` | Deploys the S3 bucket and IAM role for evaluation |
| `setup_gateway.py` | Creates the AgentCore Gateway and registers the Lambda tool |
| `create_harness.py` | Creates/updates the managed harness from `system_prompt.txt` |
| `chat.py` | Terminal chat client for multi-turn testing |
| `harness-tests.json` | Automated test suite covering all three routes plus edge cases |
| `flow-tests.json` | Copy of the test suite (alternate name referenced by the rubric) |
| `generate-eval-dataset.py` | Runs the harness over the test suite and writes the eval JSONL |
| `output_eval_dataset.jsonl` | Eval dataset (one line per test) uploaded to Bedrock Evaluations |
| `README_observations.md` | Written observations on the evaluation results |
| `cleanup_agentcore.py` | Tears down the harness, gateway target, and gateway |

## How to run

```bash
# 1. Deploy the tool stack (DynamoDB + Lambda + IAM roles)
aws cloudformation deploy --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack --capabilities CAPABILITY_NAMED_IAM --region us-east-1

# 2. Create the gateway and register the tool
python setup_gateway.py

# 3. Create the harness from the system prompt
python create_harness.py

# 4. Chat with it
python chat.py

# 5. Run the automated test suite
python generate-eval-dataset.py --tests-json harness-tests.json
```

## Evaluation results

The chatbot was evaluated with Amazon Bedrock Evaluations using the `Builtin.Correctness` metric and `amazon.nova-pro-v1:0` as the LLM-as-a-judge. **Overall correctness score: 0.88.** See `README_observations.md` for a per-route breakdown and notes.

## Evidence

Screenshots are in the `screenshots/` folder:
- `screenshot_chat_bug.png` — bug-report conversation showing follow-up questions and the `[tool call] bugreports___create_bug_report` line
- `screenshot_dynamodb.png` — the DynamoDB table with a chatbot-created ticket
- `screenshot_eval_results.png` — the Bedrock Evaluations results page
