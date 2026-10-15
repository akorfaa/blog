---
title: "I Fixed a Bug My AI Agent Didn't Know It Had"
description: "How I diagnosed and fixed a data-integrity bug in an Amazon Bedrock agent, where the model filled a required field with a placeholder instead of asking the customer."
pubDate: 2026-10-15
---

While building a customer support agent on Amazon Bedrock AgentCore, I ran into a bug that taught me something important about agentic systems: you can't always trust prompt-level instructions alone.

## The setup

I was building a three-intent support chatbot (bug reports, platform questions, general handoff) connected to an AWS Lambda tool through a Gateway, with DynamoDB handling ticket persistence.

## The bug

The bug report flow had a simple contract. Before filing a ticket, the agent had to collect every required field from the customer. If something was missing, it should ask.

Instead, the agent sometimes called the ticket tool early. When a customer reported a problem without giving all the details, the model filled the missing required parameter with a placeholder value like "N/A" or "unknown" and filed the ticket anyway. Nothing crashed. The tool call succeeded, DynamoDB happily stored the record, and the agent told the customer their ticket was created.

That's what made it a data-integrity bug rather than a visible failure. Everything looked fine from the outside, but the database was quietly accumulating tickets with junk in them, and the customer never got asked the question that would have fixed it.

## Why prompt instructions weren't enough

My first fix was the obvious one: tighten the system prompt. I added explicit rules saying the agent must never call the ticket tool until every required field has been collected from the customer, and must never invent or substitute values.

This helped, but it didn't eliminate the problem. Running the same test cases repeatedly, I still saw occasional premature tool calls. [Add here what you observed, e.g. how often it happened, or which phrasing of the customer message triggered it.]

The reason is that a prompt is a request, not a guarantee. The model weighs your instruction against everything else in the conversation, and it can decide that a plausible-looking value satisfies the schema. The tool's input schema said the field was required, but a placeholder string is still a valid string. From the model's point of view, it had complied.

So I stopped trying to make the model perfectly obedient and moved the rule to a place that could actually enforce it: the Lambda.

## The fix: server-side validation

The Lambda now validates its inputs before touching DynamoDB. If a required field is missing, empty, or a known placeholder, it rejects the call and returns a clear error message that tells the model exactly what to ask the customer for.

```python
PLACEHOLDERS = {"", "n/a", "na", "none", "unknown", "tbd", "null", "not provided"}
REQUIRED_FIELDS = ["customer_name", "issue_description", "contact_email"]  # adjust to your schema

def validate_ticket_input(params: dict) -> list[str]:
    """Return a list of missing or invalid required fields."""
    problems = []
    for field in REQUIRED_FIELDS:
        value = str(params.get(field, "")).strip().lower()
        if value in PLACEHOLDERS:
            problems.append(field)
    return problems

def lambda_handler(event, context):
    problems = validate_ticket_input(event)
    if problems:
        return {
            "statusCode": 400,
            "body": (
                "Ticket not created. Missing or invalid required fields: "
                + ", ".join(problems)
                + ". Ask the customer for these before calling this tool again."
            ),
        }

    # ...write to DynamoDB only after validation passes
```

Two details mattered here:

- **The error message is written for the model, not for a human.** It names the missing fields and says what to do next, so the agent turns the rejection into a natural follow-up question for the customer.
- **Nothing is written on failure.** The ticket is only persisted after every check passes, so bad data never reaches the table.

After this change, my repeated test runs stopped producing placeholder tickets. [Add your result here, e.g. "the evaluation set hit full correctness" if you want to mention it.]

## What I'd tell someone building their first agent

**Don't treat your prompt as a contract.** A system prompt shapes behavior, but it doesn't guarantee it. Anything that must always be true should be enforced in code.

**Validate at the boundary, not just at the model.** Your tools are the point where model output turns into real side effects, like writing to a database. Treat the model like any other untrusted input source: check it before you act on it, and return errors that help it recover.

The prompt is still worth getting right. It just shouldn't be your only line of defense.
