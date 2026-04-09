# Smoke Test Before Scale

Validate computation logic locally before committing to expensive cloud runs.

## The Problem

Cloud compute is cheap per hour, but bugs are expensive at scale. A computation with three independent bugs that each takes 20 minutes to hit will waste the first hour of a multi-hour cloud run — and potentially the entire run if the bugs cause silent wrong results rather than crashes.

The instinct in an agentic workflow is to launch immediately: the agent has the code, the cluster is available, the task is clear. This instinct is wrong. The cost of a 15-minute local smoke test is always lower than the cost of discovering a bug after an hour of cloud time.

**Real cost**: Three untested bugs in a computation launched directly to cloud cost approximately $48 in wasted compute time. All three failures were detectable in under 10 minutes on local hardware.

## What a Smoke Test Is Not

A smoke test is not a full validation run. You are not:
- Reproducing the full computation at reduced resolution
- Verifying that the results are scientifically correct
- Checking edge cases or parameter sensitivity

You are checking that the computation **starts, runs, and exits cleanly** on a minimal input.

## The Protocol

### Step 1: Define the minimal input

For every computation, identify a valid but cheap input:
- Smallest valid grid / smallest valid dataset
- Fastest-converging parameters (fewer iterations, lower resolution)
- Single example from a batch job, not the whole batch
- Short time range, not the full run

The goal: make the computation exit in under 5 minutes on local hardware.

### Step 2: Run the smoke test locally

```bash
# Example: a computation that normally runs on a 512x512x512 grid
# Smoke test at 32x32x32

GRID_SIZE=32 MAX_ITER=10 python run_computation.py --dry-run --output /tmp/smoke_test_out
```

The smoke test should:
1. Start without crashing on import
2. Initialize without error
3. Run at least a few iterations
4. Write an output file in the expected format
5. Exit cleanly (exit code 0)

### Step 3: Check the output structure

Before launching at scale, verify that:
- Output files are created at the expected paths
- Output format matches what downstream code expects
- Any required metadata (timestamps, parameter records, checksums) is present
- The computation log shows no warnings that indicate silent failures

### Step 4: Only then launch at scale

If the smoke test passes, launch the cloud run with confidence that at least the mechanical aspects work. Scientific correctness is a separate concern.

## Common Failures Caught by Smoke Tests

### Import errors and missing dependencies

A computation that imports a new library, or uses a library version that differs between local and cloud environments, will fail immediately. Smoke test catches this in 30 seconds.

### Path and environment issues

Hard-coded paths, missing environment variables, wrong working directory assumptions. A 5-minute smoke test reveals these before an expensive cluster allocates and immediately dies.

### Output format mismatches

A computation refactored to use a new output format, but downstream analysis code still expects the old format. Smoke test reveals this before spending an hour generating unreadable output.

### Silent numerical failures

Division by zero, NaN propagation, overflow — issues that produce a completed run with garbage results rather than an error. Smoke tests don't catch all of these, but they catch the cases where the failure appears in the first few iterations.

### Resource exhaustion at initialization

A computation that allocates too much memory or too many file handles during initialization will fail at scale but might succeed on a smaller machine with a smaller input. Smoke test catches this when the full input is used but the compute is done locally.

## When to Skip the Smoke Test

Almost never. The only legitimate reasons to skip:

1. **The computation is purely additive** — it appends to existing output and has already been run at least once in this code state
2. **The code is unchanged** — you are re-running a computation with different parameters on code that passed a smoke test within the current session
3. **The run is trivially short** — the full cloud run would cost less than $1 and take under 5 minutes

If you're unsure which case applies, run the smoke test.

## Integration with the Framework

- **Prompt Hardening** should include a smoke test requirement in the computation template
- The agent should not call the launch step until the smoke test step is complete
- Smoke test results should be recorded in the computation notes (pass/fail, what was checked)
- If the smoke test fails, the computation is blocked until the failure is diagnosed and fixed — not re-launched with fingers crossed
