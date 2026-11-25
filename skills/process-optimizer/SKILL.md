---
name: process-optimizer
description: Make slow, repeated or wasteful work faster or cheaper without changing what it produces. Use when a build, test run, render, export, install, data job or any command is slow or keeps being rerun; when asked to optimize, speed up, or cut the time, cost, energy, memory or size of something, including a product's runtime or battery budget; and before rerunning an expensive step. Measures first, changes one thing at a time in a copy, proves the result unchanged, reports before and after.
---

# Process optimizer

An optimization is the same result for less cost. This skill is the discipline that keeps both halves true: the cost really went down, and the result really did not change. It applies to anything with a cost you can count and a result you can compare: a build, a test suite, a render, a data job, a page's frame time, a device's energy per use, your own sequence of commands in this session.

`po` below means `node <this skill's folder>/scripts/po.mjs`. It has no dependencies and writes only to its own scratch folder.

## The contract

These hold for every change, however small. They are what makes the work safe to do without asking.

1. **Numbers before and after.** Measure the cost before you change anything and again after, on the same machine with the same inputs. No number, no claim.
2. **The result is the judge.** Before the change, decide how you will prove the result is the same (see "Choosing the judge"). After it, run that comparison. A change that alters the result, even slightly, is a trade-off and not an optimization: stop and put it to the user with the evidence.
3. **Work in a copy.** Try changes in a scratch copy, a worktree or a branch. The real thing stays untouched and runnable until a change is proven, and the old path stays available after.
4. **Nothing outside the copy.** No production, shared service, deploy, push, publish, paid resource, global install or global setting. Never run a process with outside effects (deploy, send, migrate, charge) to measure it: use its dry run, or reason from its logs. Never delete the user's files; `po` keeps its copies in its own folder.
5. **One change at a time.** Otherwise you cannot say which change helped or which one broke the result. Reject on doubt: an unproven speed-up is worth nothing.

## First, avoid the work

Most waste is work that did not need to happen. Before any command that takes more than a few seconds, ask:

- **What question does this run answer, and what is the narrowest run that answers it?** One test instead of the suite, one file instead of the project, one frame or page instead of all of them, a dry run, a smaller input.
- **Is it already answered?** If nothing it reads has changed since it last succeeded, the earlier result still stands. Do not rerun to "make sure".
- **Does this run need full quality?** Use draft settings while iterating and the full run once, at the end.
- **Am I redoing earlier stages to reach a later one?** Keep intermediate results and start from them.
- **Am I verifying by redoing?** Comparing two outputs is usually cheaper than producing a third.
- **Can it overlap with other work?** Start independent commands together, or in the background while you continue.

These cost nothing and change no files, so apply them always, asked or not. Everything below edits how a process works: do that when the user asked for it, or offer it in one line with the measured number and carry on with the task.

## The loop

1. **Define.** Name the cost (wall time, CPU, energy per event, bytes, money, manual steps) and the result that must not change. Note how often the process runs: a saving matters in proportion to cost times frequency. Work that runs once is not worth optimizing.
2. **Measure.** `po time -- <command>` for the cost and where it goes. `po null --out <outputs> -- <command>` runs it twice with nothing changed: it shows what an unchanged rerun costs and whether the outputs are deterministic. Measure each stage of a multi-stage process separately. Time what the user actually runs, cold and warm.
3. **Locate.** Rank the parts by share of the total. Only the biggest matters: halving a step that is 5% of the run saves 2.5%. Read the profile before guessing (table below).
4. **Pick one change**, the highest rung of the ladder that fits the biggest part.
5. **Try it in a copy and judge.** `po trial` runs before and after in two scratch copies, compares the outputs and gives a verdict. For changes it cannot express, do the same by hand: copy, run, snapshot, change, run, compare.
6. **Keep or reject.** Keep only on an unchanged result and a gain outside the noise (repeat runs when the gain is under about 10%). Then measure again from step 2: the bottleneck has moved.
7. **Stop** when the biggest remaining part is small, the next change would alter the result, or the gain is within noise. Say what is left and why.
8. **Report** (format at the end), and record the new way of running it where the project keeps such notes.

## The ladder

Try the rungs in this order. Higher rungs save more and risk less.

1. **Don't do it.** A step whose output nothing reads; a rerun with unchanged inputs; work done "just in case"; a wait that waits for nothing.
2. **Do less of it.** Only what changed (track by content, not dates); only the scope in question; only when it is actually needed, as late as possible; draft quality while iterating.
3. **Do it once.** Cache it, keep a warm process instead of starting one per item, hoist setup out of a loop, precompute at build time what is recomputed on every run or frame, batch many small calls into one.
4. **Do it at the same time.** Independent items across cores or connections, one process per worker, then join the pieces without re-processing them.
5. **Do it cheaper.** A better algorithm, a faster tool or setting that gives the same result, a hardware path, streaming instead of temporary files.
6. **Do it off the critical path.** Later, lazily, or in the background, so nobody waits for it.

## Reading the measurement

| `po` shows | It means | Look at |
| --- | --- | --- |
| Under about 0.3 cores busy | Waiting, not working | Sleeps and fixed waits, timeouts, retries, serial network or disk requests, a process that idles before exit |
| About 1 core busy, others idle | Serial CPU work | Rungs 1 to 3 first, then parallel workers |
| All cores busy | Already parallel | Only less work helps: rungs 1 to 3 and 5 |
| Between 1 and all cores | Partly parallel | The serial stretch: setup, one single-threaded step, the final join |
| Null run costs as much as the first | Nothing is cached | Skip unchanged inputs; find the step that ignores its cache |
| Null run outputs differ | Not deterministic | Fix the cause (timestamps, random ids, ordering) before judging anything by bytes |
| One process dominates "where" | That tool is the cost | Its own options, how many times it starts, what it is fed |
| Many short-lived processes | Startup cost repeated | One warm process, batching |

When a profile does not explain it, time the stages one by one, or use the tool's own profiler. Do not guess.

## Choosing the judge

Pick the strictest comparison the result allows, and pick it before the change.

- **Files from a deterministic process:** byte-identical. `po cmp`, or `po trial` does it.
- **Files with known noise** (timestamps, build ids): identical after ignoring those lines, `--ignore-line <regex>`. Name exactly what is ignored.
- **Pictures and video:** `po cmp` decodes them and compares pixels, so a re-encode with identical frames passes as equivalent. Use `--px` and `--pct` only for renderer noise you have measured between two unchanged runs, and state the tolerance.
- **Code behaviour:** the project's tests, with the same pass and fail list before and after, `--check "<test command>"`. Tests that already fail cannot judge.
- **Data:** row counts, checksums, a diff of sorted output.
- **Interactive or visual behaviour:** frames captured at the same moments and diffed; the same script of actions giving the same states.
- **Then look yourself** at whatever differed most. A comparison can pass on something a person would reject.

If no comparison is possible, the change is not provable and does not qualify.

## When you cannot run it

Some costs cannot be timed here: a device not built yet, production traffic, a monthly bill, battery life. Then the model is the measurement.

1. Write the budget as a table: each item, its cost per event, events per period, share of the total. Give every number its source: datasheet, measurement, or assumption.
2. Rank by share and take the ladder to the biggest item. The result that must not change is the list of requirements: what the user or the product must still be able to do.
3. Report the outcome as an estimate. Name the assumption it is most sensitive to and the measurement that would confirm it.
4. Compare alternatives on equal terms: give each side the same optimizations and conditions, or the comparison is not honest.

For runtime work (frame time, latency, startup), judge by the worst case on the slowest target that matters, not the average on your machine.

## `po`

```
po time   [--runs N] -- <command>                 wall time, CPU, cores used, which processes
po null   [--out PATH]... -- <command>            same command twice, nothing changed
po snap   <name> <path>...                        keep a copy of outputs to compare against
po cmp    <a> [<b>]                               snapshot, file or folder; bytes, then pixels or lines
po trial  --cmd "<command>" --out PATH... (--patch FILE | --apply "<shell>" | --cmd-b "<command>")
          [--check "<command>"] [--runs N] [--exclude PATH]...
po ledger                                         what this session spent its time on (with the plugin's hook)
```

`po trial` copies the project twice, runs the command unchanged in one copy and with the change in the other, compares the `--out` paths and prints `KEEP`, `NO GAIN` or `REJECT`. `--patch` and `--apply` change files in the copy; `--cmd-b` runs a different command. It exits 0 only on `KEEP`. The real project is never touched; apply a kept change yourself afterwards. Compare options: `--ignore-line REGEX`, `--px N --pct P`.

## Traps

- **A faster wrong answer.** Minifiers, optimizers and "safe" flags that rename, merge, reorder or round can break whatever reads the output. Allow only named, structure-preserving transforms, and judge.
- **Caches that lie.** A cache keyed on less than everything the step reads returns stale results. Key on content of all inputs, including config and tool versions.
- **Parallel work that shares state.** Workers writing the same file, port, temp folder or cache corrupt each other. Give each its own, and join in a fixed order.
- **Moving the cost instead of removing it.** A faster step that makes a later one slower, or a build-time saving paid at run time. Measure the whole.
- **Warm against cold.** The second run of anything is faster. Compare like with like, and say which.
- **Copies with absolute paths.** Some build folders store their own location and break, or write back to the original, when copied. Exclude them or start the copy clean.
- **Another machine.** Core counts, disks, encoders and fonts differ. Say where each number was measured, and measure again where it will run.
- **Noise.** One run proves little when the gain is small. Repeat, and report the median.

More patterns by kind of work (builds and installs, tests, renders and media, data jobs, interactive runtime, energy and cost budgets) are in `patterns.md` next to this file. Read the section that fits when the ladder alone does not suggest a move.

## The report

Keep it to what the user needs to decide and to trust the result:

- **Before and after**, per change and in total, with where it was measured: `build 94 s → 31 s (3.0x), 4-core Linux container`.
- **What was compared and the verdict:** `214 output files byte-identical`, `3,600 frames identical`, `tests: same 412 pass`.
- **What changed**, in a line each, and how to undo it.
- **What was tried and rejected**, with the reason, and what is left on the table.
- Estimates labelled as estimates.
