# QA Report: Final Sprint, Networking and Cloud Infrastructure

| | |
|---|---|
| **Section** | Shared VM (team ___) / Local VM |
| **Test author(s)** | ______________________ (Shared VM: QA) |
| **Tester** | ______________________ (Local VM: you are author and tester) |
| **Commit tested** | `git rev-parse --short HEAD` = ________ |
| **Dates** | Cases written: ________ Cases run: ________ |

## How to use this report

1. **Write the test cases first,** in Week 11, before the work they test exists. A test written after the fact tends to test what was built instead of what was asked for.
2. **Write the expected result before you run anything.** If the actual result surprises you, that is a finding, not a typo to fix.
3. **Paste real output or point to an evidence file.** "Works" is not a result. A command and its output is.
4. **Cover every success criterion with at least one case,** and include at least one negative test (something that must fail or be blocked) and the rebuild proof of life.
5. **Re-run every case in Week 13** on the rebuilt VM and update the status. A case that passed only on the old VM is a rebuild gap.

If your team replaced a criterion through [MAKE-IT-YOURS](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/MAKE-IT-YOURS.md), test the replacement instead, and link the approved design brief.

---

## Test Cases

Copy the block for each new case. Six slots are a starting point, not a limit.

<details>
<summary>Example of a filled-in case (delete when you start)</summary>

**TC-00: Flask scales out under load**

| Field | Value |
|---|---|
| Criterion | 1 |
| Steps | Run the stepped k6 profile in `capstone/networking-cloud/k6/step.js` while `kubectl get hpa -w` records in another terminal |
| Expected (written before running) | Replica count rises from 1 to at least 2 within a few minutes; `http_req_failed` stays under 1% |
| Actual | See `capstone/networking-cloud/evidence/<file>`; summary: ... |
| Status | Pass |
| Run on | Original VM, YYYY-MM-DD; rebuilt VM, YYYY-MM-DD |

</details>

### TC-01: (name your check)

| Field | Value |
|---|---|
| Criterion | |
| Type | Positive / Negative (must fail or be blocked) / Rebuild |
| Preconditions | |
| Steps (exact commands) | |
| Expected (written before running) | |
| Actual (output or evidence path) | |
| Status | [ ] Pass [ ] Fail [ ] Blocked |
| Run on | Original VM: ____ Rebuilt VM: ____ |
| Notes | |

### TC-02: (name your check)

| Field | Value |
|---|---|
| Criterion | |
| Type | Positive / Negative (must fail or be blocked) / Rebuild |
| Preconditions | |
| Steps (exact commands) | |
| Expected (written before running) | |
| Actual (output or evidence path) | |
| Status | [ ] Pass [ ] Fail [ ] Blocked |
| Run on | Original VM: ____ Rebuilt VM: ____ |
| Notes | |

### TC-03: (name your check)

| Field | Value |
|---|---|
| Criterion | |
| Type | Positive / Negative (must fail or be blocked) / Rebuild |
| Preconditions | |
| Steps (exact commands) | |
| Expected (written before running) | |
| Actual (output or evidence path) | |
| Status | [ ] Pass [ ] Fail [ ] Blocked |
| Run on | Original VM: ____ Rebuilt VM: ____ |
| Notes | |

### TC-04: (name your check)

| Field | Value |
|---|---|
| Criterion | |
| Type | Positive / Negative (must fail or be blocked) / Rebuild |
| Preconditions | |
| Steps (exact commands) | |
| Expected (written before running) | |
| Actual (output or evidence path) | |
| Status | [ ] Pass [ ] Fail [ ] Blocked |
| Run on | Original VM: ____ Rebuilt VM: ____ |
| Notes | |

### TC-05: (name your check)

| Field | Value |
|---|---|
| Criterion | |
| Type | Positive / Negative (must fail or be blocked) / Rebuild |
| Preconditions | |
| Steps (exact commands) | |
| Expected (written before running) | |
| Actual (output or evidence path) | |
| Status | [ ] Pass [ ] Fail [ ] Blocked |
| Run on | Original VM: ____ Rebuilt VM: ____ |
| Notes | |

### TC-06: (name your check)

| Field | Value |
|---|---|
| Criterion | |
| Type | Positive / Negative (must fail or be blocked) / Rebuild |
| Preconditions | |
| Steps (exact commands) | |
| Expected (written before running) | |
| Actual (output or evidence path) | |
| Status | [ ] Pass [ ] Fail [ ] Blocked |
| Run on | Original VM: ____ Rebuilt VM: ____ |
| Notes | |

---

## Success Criteria Verification

Evidence paths are relative to `capstone/networking-cloud/` and come from [EVIDENCE-CHECKLIST](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/EVIDENCE-CHECKLIST.md). Change a path if your file is named differently.

| Criterion | Evidence | Test case(s) | Met? |
|---|---|---|---|
| 1. Scales to 2+ replicas under at least 3x baseline load | `evidence/baseline.txt`, `evidence/hpa-over-time.txt`, `evidence/k6-load.txt` | | [ ] Yes [ ] No |
| 2. Under 1% failed requests during a 3-minute drill (Shared VM: one agent node stopped; Local VM: one `web` pod deleted and one rollout) | `evidence/drill-k6.txt`, `evidence/drill-times.txt` | | [ ] Yes [ ] No |
| 3. PodDisruptionBudget for each scaled tier and what happened to the pods | `evidence/pdb.txt`, `evidence/drill-pods.txt` | | [ ] Yes [ ] No |
| 4. Request trace in ARCHITECTURE.md and one tightened or added NetworkPolicy with a before/after test | `ARCHITECTURE.md`, `evidence/netpol-before-after.txt` | | [ ] Yes [ ] No |
| 5. Memory limits of additions under 768Mi, `maxReplicas` capped, arithmetic saved | `evidence/memory-budget.txt` | | [ ] Yes [ ] No |

---

## Rebuild Verification (Week 13)

| Check | Result |
|---|---|
| Rebuild date and VM (new VirtualBox VM, or Shared VM queue slot) | |
| Started from commit | |
| Proof of life, Shared VM: Three nodes `Ready`; HPA and PodDisruptionBudgets exist; `/health` returns 200 on your Kubernetes port; a 60-second k6 run completes | [ ] Pass [ ] Fail |
| Proof of life, Local VM: One node `Ready`; HPA and PodDisruptionBudget exist; `/health` returns 200 on the `web` Service address; a 60-second k6 run completes | [ ] Pass [ ] Fail |
| Every app change came back (schema, routes, services, images) | [ ] Yes [ ] No |
| Gaps logged in `capstone/networking-cloud/REBUILD-GAPS.md` and fixed by pull request | ___ gaps found, ___ fixed |
| All test cases above re-run on the rebuilt VM | [ ] Yes [ ] No |

---

## Deliverables Verification

- [ ] Implementation committed under `capstone/networking-cloud/` (merged by pull request)
- [ ] `capstone/networking-cloud/RUNBOOK.md`: operate, check, and recover
- [ ] `capstone/networking-cloud/TOOL-VERSIONS.txt` (and `requirements.txt` for Python packages)
- [ ] `capstone/networking-cloud/ARCHITECTURE.md` with a diagram and a "What we changed and why" section
- [ ] "How to run this" section in the top-level README
- [ ] `capstone/networking-cloud/REBUILD-GAPS.md` from the Week 13 rebuild
- [ ] This QA report, with every case run and signed
- [ ] Shared VM: `docs/final-sprint-retrospective.md`. Local VM: the Individual Reflection section
- [ ] `capstone/networking-cloud/DESIGN-BRIEF.md`, only if a criterion was replaced, with the approval noted
- [ ] Discussion answers in the Google Doc
- [ ] Final presentation and Week 14 rebuild presentation prepared (live on the Shared VM, recorded on the Local VM)
- [ ] Track-specific: `evidence/` with saved output for every criterion (HPA over time, k6 summaries, drill times, memory arithmetic)
- [ ] Track-specific: Network and service-discovery analysis in `ARCHITECTURE.md`, with one before/after NetworkPolicy test
- [ ] Track-specific: Shared VM: OpenTofu for the scaling objects marked **[PREDICT ONLY]**, prediction in the Google Doc. Local VM: HPA, PDB, and any new policy in `week-5/`, applied by the pipeline

---

## Rework Required

| # | Failed case or criterion | What is wrong | Fix and owner | Re-test date | Re-test result |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |

---

## Sign-Off

**Overall status:** [ ] All criteria met [ ] Met with documented exceptions [ ] Rework required

**Exceptions** (criteria not met, and why):

```

```

| Role | Name | Date |
|---|---|---|
| Test author | | |
| Tester (Local VM: same person) | | |

**Notes:** observations about technical quality and, on the Shared VM, team coordination.
