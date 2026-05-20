---
name: service-release-check
description: Compare a service's recent release with symptoms and telemetry, then recommend continue, mitigate, or roll back.
---

# Service Release Check

Use when asked whether a recent release may explain a service problem.

Inputs: service, environment, timeframe. Default timeframe is the last two hours.

Steps:

1. Identify deploy markers, version changes, config changes, and recent PRs.
2. Compare error rate, latency, saturation, restarts, logs, and alerts before and after release.
3. Separate correlation from proof.
4. Recommend `continue`, `mitigate`, or `roll back`.

Include the evidence that would change the recommendation.

