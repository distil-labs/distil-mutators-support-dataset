# Brightpath support: a seed and test set for prompt mutators

Input data for the distil labs blog post on prompt mutators. A customer support assistant for
Brightpath, a fictional team task-management SaaS, with four tools and five customer intents.
The seed data is deliberately skewed to simulate gaps in production traces: the
billing dispute is almost missing, only few cases of rude customers. The mutator configs help to maintain synthetic data distribution.


## Files

| file | what it is |
|---|---|
| `job_description.json` | task description (plan table and policy per intent) and the four tool schemas |
| `train.jsonl` | 50 seed conversations, one JSON object per line |
| `test.jsonl` | 55 test conversations, stratified so the rare intent is measurable |
| `configs/no-mutators.yaml` | baseline `config.yaml`, no mutators |
| `configs/fixed-mutator.yaml` | one `intent` mutator with explicit target probabilities |
| `configs/adaptive-mutator.yaml` | the same mutator with `method: adaptive` |

Each conversation row is `{"metadata": {"intent": ...}, "messages": [...]}` in the OpenAI chat
format, agentic variant: `user`, `assistant` and `tool` roles, one tool call per assistant
message at most, every tool call followed by its `tool` result. The `metadata.intent` label is
for slicing results; the pipeline ignores it.

## The task

The assistant handles the whole inbound support stream. It can call:

| tool | used for |
|---|---|
| `send_password_reset(email)` | sign-in problems, after the customer confirmed the email |
| `get_subscription(email)` | plan, seats, renewal date and recent invoices |
| `open_billing_case(email, invoice_id, case_type, amount, description)` | refunds, double charges, invoice errors |
| `transfer_to_human(reason, summary)` | customers who ask for a person, or abusive customers after one warning |

The policy in `job_description.json` says how each intent is handled. The one that matters:
a billing dispute is resolved by looking up the subscription, identifying the invoice and opening
a case with the right type and amount. It is never handed to a human, although that is what the
majority of the seed data does for everything else, and what a student trained on seed-shaped
data learns to do.

## Intents

| intent | seed (50) | test (55) | mutator target |
|---|---|---|---|
| password reset or login problem | 21 (42%) | 10 | 25% |
| wants to speak to a human | 14 (28%) | 8 | 15% |
| plan and pricing question | 11 (22%) | 10 | 20% |
| billing dispute | 2 (4%) | 20 | 35% |
| rude customer | 2 (4%) | 7 | 5% |

A 15-customer roster with consistent invoices underlies every tool result, so cases can be
checked against the subscription they follow: a double charge on renewal day, VAT applied despite
a valid VAT id, a seat billed after it was removed, a monthly charge on a yearly plan, a charge
after cancelling. Several billing test conversations open with a request for a human, so the
tempting wrong action is present.

## Using it

Put `job_description.json`, `train.jsonl`, `test.jsonl` and one of the configs (renamed to
`config.yaml`) in a directory and point the distil labs pipeline at it. The three configs are
identical except for the `synthgen.mutators` block:

```yaml
synthgen:
  mutators:
    - name: intent
      method: fixed            # or adaptive
      target_distribution: [0.25, 0.15, 0.20, 0.35, 0.05]
      values:
        - password reset or login problem - ...
        - wants to speak to a human - ...
        - plan and pricing question - ...
        - billing dispute - ...
        - rude customer - ...
```

The values carry a dash and an explanation each; the teacher sees the whole string, and so does
the classifier the adaptive method uses to measure what came back.
