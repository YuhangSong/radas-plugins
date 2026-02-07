Load the ask_user skill and test it. Use the radas-3.11 python environment.

**Test**: Ask user to reply "OK".

```python
from radas.email import ask_user

answer = ask_user(
    to_address="dr.yuhang.song@gmail.com",
    question="<p>Please reply <strong>OK</strong> to confirm.</p>",
)
print(f"Received: {answer}")
```

If any error occurs, stop and report.
