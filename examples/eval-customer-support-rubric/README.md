# eval-customer-support-rubric (Customer Support Rubric with Severity Gating)

Score a customer support assistant on five quality dimensions at once, and fail high-severity cases when any single dimension fails.

## Setup

```bash
npx promptfoo@latest init --example eval-customer-support-rubric
cd eval-customer-support-rubric
export OPENAI_API_KEY=your-key
```

The assistant under test uses `openai:chat:gpt-5.4-mini` and the grader uses `openai:chat:gpt-5.6`, so one API key runs the whole example. The grader is deliberately a stronger model than the assistant, so the assistant is not grading its own replies.

## Run

```bash
npx promptfoo@latest eval --no-cache
npx promptfoo@latest view
```

Run only the high-severity cases:

```bash
npx promptfoo@latest eval --no-cache --filter-metadata severity=high
```

## How It Works

The assistant works for Acme Mobile, a fictional carrier whose policy is the system message in `prompts/support.json`. The 11 test cases in `tests.csv` are customer messages in English and Spanish. They include an out-of-policy question (US roaming) that checks whether the bot admits it doesn't know instead of guessing.

### Five dimensions as named metrics

`defaultTest` applies five `llm-rubric` assertions to every case. Each one has a `metric`, so the results view shows a pass rate per dimension:

| Metric        | Question the grader answers                                           |
| ------------- | --------------------------------------------------------------------- |
| `Correctness` | Do the claims match the case's `ground_truth`? (weight 2)             |
| `Resolution`  | Does the reply address the request with an answer or next step?       |
| `Honesty`     | Does it avoid promising refunds or discounts, or inventing facts?     |
| `Handoff`     | Does it transfer to a human exactly when `handoff_expected` is `yes`? |
| `LanguageFit` | Is it written in the customer's language, with a support-chat tone?   |

Each row carries its own `ground_truth`, so the grader judges against the policy facts for that case instead of guessing.

### Severity gating

Severity is set per row in `tests.csv` with two columns:

| `__metadata:severity` | `__threshold` | Result                                                 |
| --------------------- | ------------- | ------------------------------------------------------ |
| `high`                | (empty)       | Fails if any dimension fails, whatever the total score |
| `low`                 | `0.8`         | Passes if the weighted average score is at least 0.8   |

Correctness has `weight: 2`, so the weights total 6. A low-severity reply can miss one other dimension (5/6 ≈ 0.83, pass), but a wrong fact always fails it (4/6 ≈ 0.67).

This uses built-in behavior, with no custom scoring code. Without a threshold, a test fails when any assertion fails. A test-level threshold switches pass/fail to the aggregate score. `__metadata:severity` makes severity filterable with `--filter-metadata` and visible in the results view.

## Adapting It

- Replace the `providers` entry with your real bot, for example an [HTTP provider](https://www.promptfoo.dev/docs/providers/http/).
- Add rows to `tests.csv`. Give each one a `ground_truth`, and leave `__threshold` empty for cases where any failure is unacceptable.
- Add a dimension by adding another `llm-rubric` with its own `metric` to `defaultTest.assert`.
