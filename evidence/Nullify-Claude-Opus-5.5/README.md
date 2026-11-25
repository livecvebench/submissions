# Nullify autofix evaluation with Claude Opus 5.5

Nullify resolved 117 of the 190 LiveCVEBench leaderboard cases (61.58%) with the `user_report` input. The maintained verified subset accounted for all 117 successes across 183 cases (63.93%). [Browse the 190 cases and their test logs](CASES.md) or [open the submitted result file](../../submissions/Claude-Opus-5.5_Nullify.json).

## How to inspect a result

Choose a CVE in the [case index](CASES.md). The **Delivered fix** column links to the patch the harness accepted, including unsuccessful fixes. The **Official test log** gives the final grading run, delivery status, Terminal-Bench verdict, failure mode, test timestamps and individual parser outcomes. Its last section contains filtered terminal output in recorded order. Unavailable environments have a setup record instead.

Of 185 attempted cases, 179 had a delivered patch and six ended with an agent error before the harness accepted a fix. All 117 successes have a patch. For 174 deliveries the published diff is byte-for-byte identical to the sealed patch. The other five publish the implementation changes only. Their logs identify the omitted regression test files and provide both the original and published SHA-256 values.

The test log export keeps pytest outcome lines and numeric summaries. It strips terminal formatting, omits assertion bodies and shell commands, and replaces test parameter values with short hashes. The official parser results are listed separately because the parser can retain a verdict even when terminal output is incomplete. The source for 184 cases was the saved test session log. For CVE-2025-55741 the retained source was the post-test terminal pane. Four sessions had no matching terminal verdict or summary lines. Their records retain the timeout or parse error. The logs include digests of the source output and the sealed grade record. These hashes support file comparisons when the original records are available. [Public file checksums](SHA256SUMS)

## How Nullify builds a repair

LiveCVEBench supplies a user report. The harness starts the task services, waits for them to settle, and captures the application source and runtime configuration. A read-only localization agent inspects the captured files and identifies a primary file and line range, a defect description, a CWE and any related locations. Nullify passes that finding, the report and the related locations to its SAST autofix planner, editor and reviewer. The editor works on its own source copy. The reference fix and hidden grading tests stay outside the agents' workspace.

### Recipe discovery

A **recipe** records how to exercise this particular checkout. Each step has an ordered position, a phase, a command, a working directory, a timeout and an expected exit code. The recipe also records which capabilities the runner demonstrated, such as building the project, running its unit or integration tests, or starting a service.

1. **Read the project.** On the original source, discovery identifies manifests and lockfiles, toolchain versions, package scripts and test files. The planner can inspect Makefiles and CI workflows and use the ecosystem's standard commands as a starting point. It proposes the project's own setup, build, lint, test and service-start steps where available.
2. **Try the commands.** The runner executes the plan in the captured runtime and records exit codes, timeouts and output. If a command fails, the recipe repairer can use that evidence to revise the plan and retry. A separate reviewer checks whether model-proposed commands actually performed the work claimed for their phase. Setup file edits also get reviewed. Only edits classified as setup can travel with the recipe.
3. **Replay from scratch.** The runner extracts a fresh copy of the original source, applies any approved setup edits and reruns the retained steps without asking the model to plan again. It compares the results with discovery, including the expected exit codes. Only a successful clean replay produces a frozen recipe. The artifact ties its steps and capabilities to the original source hash and the task. A build can be supported even when integration tests are unavailable.

### What happens inside the sandbox

For a single project, the runner uses its captured, pinned Docker image and mounts the working source at the captured paths. For a multi-service application, it can recreate the services together in a private Docker Compose project with pinned images. The benchmark configuration permits dependency downloads through a per-task registry proxy. It disables web search and retrieval, and masks the hidden test directory.

For each candidate, the harness creates a fresh working copy, applies the recipe's approved setup edits and runs the project formatter on changed paths when one is available. It copies verified formatter changes back to the editor's candidate. It replays the supported lint, build, unit-test and integration-test steps in order with their prerequisites. If the recipe proved that the application can start, the plan can include startup and readiness steps so the reviewer can exercise the running service.

| What the runner observes | What Nullify does with it |
| --- | --- |
| A supported test command exits successfully | It records a pass only when the output shows tests actually ran. An empty suite or an unsupported phase is marked `ci_fallback`. This run gives that phase no credit. |
| A command fails with a nonzero exit already seen during discovery | When time permits and the suite is applicable, the runner repeats the prerequisites and failing step in a second sandbox on untouched source. Matching exit codes and named failures can establish a pre-existing failure. A new named failure goes back to the editor. Inconclusive evidence defers the check. |
| A command exposes a regression, or review finds the fix incomplete | The sandbox reviewer receives the intent of the fix, the candidate diff, changed paths and replay output. It can inspect the candidate in the same runner. A failed check or a review that finds a broken fix returns concrete feedback to the editor, which can revise and resubmit the patch. Rechecking the same diff can also produce another recorded round. |
| Discovery cannot prove a recipe, a phase is unavailable, or the sandbox runs out of setup time | Repair can continue with less local evidence. Source preparation enforces limits of 8,192 paths and 128 MiB. When the original tree exceeds those limits the run skips discovery, sandbox validation and test generation. |

On eligible projects, a test writer can propose a regression test. To retain it, the harness checks that the security assertion fails on the original and passes on the candidate, and that ordinary input passes on both. For the leaderboard, the harness delivers the final patch to the task environment, repeats the declared build steps and restarts the source service. Terminal-Bench then runs the hidden tests. A case scores only when delivery is accepted and the official grader reports `is_resolved: true`.

### Sandbox coverage in this run

Of 185 attempted cases, **84 produced a frozen recipe**. Those cases recorded **256 candidate validation rounds**. Of the other 101 attempts, 52 recorded an error preparing the original source for discovery, 45 recorded a discovery error, and four have no recorded discovery outcome. Five additional leaderboard cases had no usable environment.

| Check | Distinct cases with a recorded pass in at least one round |
| --- | ---: |
| Build | 45 |
| Lint | 15 |
| Unit tests | 7 |
| Integration tests | 2 |
| Sandbox fix review | 53 |

The [sandbox round records](sandbox-rounds.csv) cover all 190 leaderboard cases. Each numbered row is one validation call. Repeated patch hashes identify rechecks of the same candidate before formatting. Cases with no recorded round have one row with an empty round number. A blank check has no recorded outcome in that round. `ci_fallback` means the check was deferred, and `undetermined` means the reviewer reached no verdict. The independent leaderboard verdict and official test output are in the [case index](CASES.md).

### Worked example from AIDE

For CVE-2025-54389, discovery proved a four-step project recipe on the original source. It bootstrapped the toolchain, ran configure, built AIDE with make and ran the project's unit tests with `make check`. The clean replay reproduced all four expected zero exit codes. Lint and integration testing were deferred throughout candidate validation.

| Sandbox round | Candidate SHA-256 prefix | Build | Unit tests | Fix review |
| --- | --- | --- | --- | --- |
| 1 | `7d3a0ac8360b` | Passed | Passed | Failed. The reviewer observed an unescaped control character in AIDE's diagnostic output. |
| 2 | `96a6c7e13cca` | Passed | Passed | Passed on the revised patch. |
| 3 and 4 | `96a6c7e13cca` | Passed | Passed | Passed again on the same patch. |

The delivered patch escapes control characters in report and logging paths. Its full digest matches the candidate from rounds 2 through 4. The official grader then marked the case resolved. [Delivered patch](fixes/CVE-2025-54389.patch) · [Official test log and patch digest](test-logs/CVE-2025-54389.txt)

## Evaluation settings and coverage

- The run used dataset revision `b11057ac5288aa4ed595452f7e5f1df9919dbc00`, Nullify harness revision `e8118e18431d933985367ac2ec9a38c04ce0a25c` and Terminal-Bench 0.2.18. The model identifiers were `claude-opus-5-5` and `claude-haiku-5-5`.
- Each available case received one model attempt. The per-attempt agent limit was 6,840 seconds, including up to 1,080 seconds for recipe discovery and 2,700 seconds for repair. Official grading used each task's declared test timeout. The remaining cases after the 48-case pilot ran with up to 12 concurrent task workers.
- Infrastructure failures on CVE-2025-46346 and CVE-2025-9394 led to model-free regrades of the same saved attempts. Each case links the original delivery to the final regrade log. Both remain single entries in the denominator.
- The no-fix control passed on zero of 185 available environments. Reference fixes passed on 178 of 185. The seven reference failures were CVE-2025-24359, CVE-2025-48957, CVE-2025-52904, CVE-2025-55742, CVE-2025-6272, CVE-2025-9157 and CVE-2025-9796.
- Five unavailable environments count as unsuccessful with zero turns and tokens. CVE-2025-62416 was in the verified set. CVE-2025-12276, CVE-2025-68129, CVE-2025-9136 and CVE-2025-9670 were outside it. Of the three other cases outside that set, the original tests reported CVE-2025-49844 and CVE-2025-5644 unresolved, and CVE-2025-5501 ended with an agent error.
- Turns count model calls across roles. Tokens include recorded input and output, including cache usage. Counts are lower bounds for CVE-2025-14967, CVE-2025-46346, CVE-2025-46551, CVE-2025-61915, CVE-2025-8521 and CVE-2025-9772 due to incomplete provider usage records.
- The environment depended on upstream images and package availability at run time. Planning recorded ten unavailable cached references for auxiliary services. These tasks used the harness's normal checks for official auxiliary services. Their observed outcomes are included.
