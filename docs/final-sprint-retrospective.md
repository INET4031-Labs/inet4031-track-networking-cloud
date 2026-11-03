# Final Sprint Retrospective

**Sprint:** Final
**Date:** TODO
**Scrum Master:** TODO

## Sprint Goals

- Flask tier scales automatically to at least 2 replicas under a k6 load of at least 3x baseline, with evidence saved
- `http_req_failed` below 1% during a 3-minute node-failure drill on your own k3d node
- PodDisruptionBudget for each scaled tier, with evidence of behavior during node loss
- Written network and service-discovery analysis with one tightened NetworkPolicy and a before/after test
- Added memory limits under the team budget (768Mi) with `maxReplicas` capped

## What Went Well

TODO: List 2-3 things the team did well this sprint.

## What Could Improve

TODO: List 2-3 things that were difficult or took longer than expected.

## Process Change for Next Sprint

TODO: Describe one specific change the team would make if we had another sprint.

## Blockers Encountered and Resolved

TODO: List any technical or process blockers. How did the team resolve them? Did they require help from a TA?

## Metrics Achieved

- TODO: Baseline p95 response time: ___ms at ___ requests/second
- TODO: Peak load tested (multiple of baseline): ___x
- TODO: Maximum Flask replicas reached by the autoscaler: ___
- TODO: Time from load increase to first new ready replica: ___s
- TODO: Node-drill error rate (http_req_failed): ___ over ___ minutes
- TODO: Memory limits added beyond baseline: ___Mi of 768Mi

## Team Communication During Sprint

TODO: How often did the team check in? Did async communication work well? Any scheduling conflicts?

## Decisions Made This Sprint

TODO: List major technical or team decisions (CPU target, replica floor and cap, probe timings, which NetworkPolicy to tighten).

## Team Contributions

| Role | Name | Primary Contribution |
|------|------|----------------------|
| Scrum Master | TODO | TODO |
| System Admin | TODO | TODO |
| Developer | TODO | TODO |
| Developer | TODO | TODO |
| QA | TODO | TODO |

## Individual Reflection (Local VM only)

1. What did you build? TODO
2. What surprised you? TODO
3. What would you do next? TODO
4. Which success criteria did you not meet, and why? TODO
