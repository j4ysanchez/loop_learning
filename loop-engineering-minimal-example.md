# Loop engineering: a minimal example

A simple, self-contained way to feel the mechanics of a loop before applying it to anything real: fix a single failing test with a verifier and a budget cap.

## 1. Create a failing test as your done-check

Pick a small repo — or a throwaway one — and write a single test that fails against current code. Something like asserting a function returns the right total. This is your mechanical success condition: the loop is done when this test exits 0, not when Claude says it thinks the fix worked.

## 2. Write your three stop conditions before touching the loop

Success = the test passes. Failure = N attempts with no progress. Budget = a hard turn cap. Write all three down before you run anything. Improvising them mid-run is how a five-minute demo turns into a runaway bill.

**Spec (what you'd write, or hand to `/goal`):**
```
Success: pytest tests/test_foo.py exits 0
Failure: same diff on src/foo.py two attempts in a row (no progress)
Budget:  5 attempts, hard stop
```

**Enforcement (what actually stops the loop):**
```bash
MAX_ATTEMPTS=5
last_diff=""

for i in $(seq 1 $MAX_ATTEMPTS); do
  claude -p "Fix the failing test in tests/test_foo.py. Only touch src/foo.py."

  # success — separate process, not the agent's opinion
  if pytest tests/test_foo.py; then
    echo "SUCCESS: passed on attempt $i"
    break
  fi

  # failure — no-progress check
  current_diff=$(git diff --stat src/foo.py)
  if [ "$current_diff" == "$last_diff" ]; then
    echo "FAILURE: no progress since last attempt, stopping"
    exit 1
  fi
  last_diff="$current_diff"

  # budget
  if [ "$i" -eq "$MAX_ATTEMPTS" ]; then
    echo "BUDGET EXCEEDED: $MAX_ATTEMPTS attempts, stopping"
    exit 1
  fi
done
```

Success is the `pytest` exit code — a fresh process, not something the agent reports on itself. Failure is the diff comparison — if `src/foo.py` looks identical to the last attempt, it's stuck, not close. Budget is just the loop bound, but it has to be a hard `exit`, not a `break` that lets the script quietly continue past it.

If you use `/goal` instead of the raw bash, the success line becomes the goal string and `--max-turns 5` covers budget — but the no-progress check doesn't exist natively, so you'd still want the diff comparison wrapped around it, or you're trusting the evaluator not to loop on a stuck agent.

## 3. Separate the maker from the checker

The agent that writes the fix should never be the one that decides if it worked. Run the test suite yourself, in a fresh shell, after each attempt — `pytest tests/test_foo.py`, not asking Claude whether the tests pass. This is the most-skipped step and the one that turns a loop into theater.

## 4. Wire up the loop

Five lines of bash is enough: a `for` loop that calls Claude to attempt the fix, then runs pytest and breaks on success. Or, on a recent Claude Code build, `/goal "tests/test_foo.py passes with exit code 0" --max-turns 5` does the same thing natively — the completion check still runs as a separate evaluator pass, not the agent's own say-so.

## 5. Cap the budget

Five attempts on one failing test is generous. If it's not fixed by then, the task was underspecified, not almost-done. When a loop stalls, the right move is narrowing the task, not widening the loop.

## 6. Add a human checkpoint

Even on something this small, don't let the loop merge its own fix. Have it stop on a pass and print a diff, or push to a branch, so you look at what changed before it lands anywhere real.

## 7. Run it once and let it fail on purpose

Cap it at one attempt against a test that needs a real fix, and watch it hit the budget and stop cleanly. That failure mode is more informative than the success — it's the behavior you're trusting it to have later on bigger, higher-stakes loops.
