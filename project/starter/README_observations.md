# Customer Support Chatbot — Evaluation Observations

## What was built
A customer support chatbot on the Amazon Bedrock AgentCore managed harness. All routing lives in a single system prompt (system_prompt.txt) that classifies each message into one of three routes:
- Bug report: collects description, steps to reproduce, and environment across the conversation, then calls the create_bug_report tool (Lambda + DynamoDB via the AgentCore Gateway) and returns the ticket ID.
- Platform question: answered only from the embedded FAQ.
- Other: politely redirected to the human support line at 1-800-555-0199.

## Test suite
harness-tests.json contains 8 single-turn tests covering all three routes plus edge cases: two bug reports (one detailed, one very short), two FAQ-covered questions, two other/off-topic requests, one FAQ-uncovered question (hand-off), and one prompt-injection attempt.

## Evaluation
Ran generate-eval-dataset.py to produce output_eval_dataset.jsonl (one line per test), uploaded it to S3, and created a Bedrock Evaluations job (support-chatbot-eval-run-1) using the Builtin.Correctness metric with amazon.nova-pro-v1:0 as the LLM-as-a-judge evaluator.

## Results and observations
Overall correctness score: 0.88 (Builtin.Correctness, LLM-as-a-judge with amazon.nova-pro-v1:0).

Per-route observations:
- FAQ questions (returns, payments) scored well: answers came straight from the embedded FAQ.
- Other/off-topic requests correctly redirected to the support phone line.
- The prompt-injection test was refused; the assistant did not reveal its system prompt.
- The FAQ-uncovered question (gift wrapping) was correctly handed off to support.
- Bug-report route: because each eval test runs as a single fresh turn, one bug test filed a ticket immediately rather than first asking for the missing fields. In a real multi-turn chat.py session the assistant does collect the fields one at a time before filing, which is confirmed by the DynamoDB records. This single-turn behavior is the main area a stricter prompt could improve.

## Possible improvements
- Tighten the bug-report instruction so the tool is never called until all three fields have real values.
- Suppress the model's visible <thinking> blocks in customer-facing output.
- Add a Bedrock Guardrail to block harmful content and injection before the model processes the message.
