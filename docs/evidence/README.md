# Evidence

Raw outputs from each run on 2026-10-11 (US Eastern). All files are kept as produced, with two exceptions: lines that printed AWS credentials were removed (none remain), and so was a console hint the course scaffold printed after each run. The baseline end-to-end log keeps its original header, which uses the course's name for the exercise ("Stretch Goal A").

| File | Run | Build | Used in |
|---|---|---|---|
| [`test_output_baseline.txt`](test_output_baseline.txt) | Three core scenarios, as originally submitted | Baseline | F3, F4 |
| [`guardrail_e2e_before.txt`](guardrail_e2e_before.txt) | End-to-end test, 15:29 | Baseline runtime | F1, F2, F5, guardrail parity, idempotency |
| [`test_suite_hardened_120.txt`](test_suite_hardened_120.txt) | 120-point automated suite | Fix v1 | 120/120 retained |
| [`test_output_hardened_v1.txt`](test_output_hardened_v1.txt) | Three core scenarios | Fix v1 | R1 regression; F3 persisting |
| [`test_output_hardened.txt`](test_output_hardened.txt) | Three core scenarios, 18:21 | Fix v2 | The filter removing a sentence; F3 and R1 corrected |
| [`guardrail_e2e_after.txt`](guardrail_e2e_after.txt) | End-to-end test, 19:14 | v2 runtime | 8/8, `reply_source` values |
| [`baseline-to-hardened.diff`](baseline-to-hardened.diff) | `git diff` of `src/agent_orchestrator.py` | — | Every code change |

Each run was a single trial per prompt; see [02 Method, Limits](../02-method.md#limits-of-the-method).
