Load the radas skill and run each test below (sequentially). For each test (unless stated other wise), create a new random tmp directory under user's home/tmp/ for the test to avoid conflicts with existing code. Use the radas-3.11 python environment.

1. (under new random folder) Run a simple grid search experiment. Plot the search results with sns.relplot, save the plot to a file. Check all complete without errors and the saved plot exists and has valid content.
2. (under the same folder as test 1) Re-analyze the experiment from test 1 (use the same experiment_name) without re-running it, via dos=["analyze"]. Use sns.lineplot this time, save the plot to a file. Check all complete without errors and the saved plot exists and has valid content.
3. (under new random folder) Run a simple experiment of OptunaSearch.
4. (under the same folder as test 1) Restore the experiment from test 1 by run_choice="restore" and check it resumes correctly (check the logs indicate resuming from previous state, rather than starting a new experiment).
5. (under new random folder) Manage life cycle of a long-run experiment: start a long-run experiment, when seeing it starts correctly, stop it and check it stops correctly (via reading the logs, it should indicate if it stops correctly).
6. (under new random folder) Test radas storage (both methods): (a) On the head node, write a file to `get_data_dir()`, then read it from inside trainable. (b) Inside a trainable, save an object via `cloud_artifact`, then read it back on the head node.

In case any test fails at any stage, stop and report to me instead of trying to fix anything. You need to provide a comprehensive report of what is wrong so that it can be copy and paste to the agent who is developing this radas skill to fix the issue.

Any error you got or confusion you got indicate that the skill or the python package need to be improved?
