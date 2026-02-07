---
name: ask_user
description: "Interact with the user via email. Use whenever you would normally stop and wait for the user: asking questions, reporting results, or requesting next steps."
---

# ask_user - Email-based User Interaction

Send messages to the user via email and wait for their reply. Blocks until a reply is received (no timeout).

## Activation

```
/ask_user <email-address>
```

## Prerequisites

Requires the `radas` Python package. Use the Python environment specified by the user. If not specified, ask the user (via `AskUserQuestion`) **before** activating this skill.

## Running Scripts

```bash
source ~/miniconda3/etc/profile.d/conda.sh && \
conda activate <your-env> && \
python -u -c "<your code here>"
```

## API

```python
from radas.email import ask_user

answer = ask_user(
    to_address="user@example.com",
    question="Your message here (plain text or HTML)",
    images=None,  # Optional: list of base64-encoded PNG strings
    attachment_paths=None,  # Optional: list of file paths
    poll_interval=30,  # Seconds between checks (default: 30)
) -> str  # Returns user's reply
```

## When to Use

**Once this skill is activated, `ask_user()` is your ONLY way to communicate with the user.** The user is away from the terminal and will not see terminal output.

### Scenario 1: Need User Input
You need a decision, confirmation, or choice.

```python
answer = ask_user(
    to_address="user@example.com",
    question="""
<h3>Choose a database:</h3>
<ol>
    <li>PostgreSQL</li>
    <li>SQLite</li>
    <li>MySQL</li>
</ol>
<p>Reply with 1, 2, or 3.</p>
""",
)
```

### Scenario 2: Task Completed / Milestone Reached
You finished a task or hit a milestone. Send results and ask what to do next.

```python
answer = ask_user(
    to_address="user@example.com",
    question="""
<h3>Task Complete</h3>
<p>I've fixed the authentication bug. Here's what I changed:</p>
<ul>
    <li><code>auth.py</code>: Fixed token expiry check</li>
    <li><code>test_auth.py</code>: Added regression test (all tests pass)</li>
</ul>
<p>What would you like me to do next?</p>
""",
)
```

## Rules

Once activated:

1. **NO terminal interaction**: Do NOT use `AskUserQuestion` or stop expecting the user to read terminal output
2. **NEVER stop silently**: Always send an email with results before stopping
3. **Wait indefinitely**: `ask_user()` blocks until the user replies — this is expected
