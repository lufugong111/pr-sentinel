---
name: code-review-scoring
description: Review code changes systematically, score verified findings by impact and evidence, and return actionable, auditable feedback. Use when an agent is asked to inspect generated or modified code, prioritize defects, or provide a scored review.
---

# Code Review Scoring Skill

Use this skill to review code changes and produce consistent, evidence-backed scores. Apply the rubric to each finding separately. Do not treat a score as proof that a defect exists, and do not invent test, tool, or runtime results.

## Review Procedure

1. Establish the review scope. Identify the diff or changed files, intended behavior, relevant language/framework, and any repository guidance or tests available. Prefer the supplied diff as the change boundary; inspect surrounding code when tools or source files are available.
2. Inspect changes by risk-bearing behavior, not just keywords. Trace inputs to sensitive operations, authorization decisions, state changes, error paths, concurrency boundaries, and resource handling where applicable.
3. Form candidate findings only for actionable defects. For each candidate, verify the changed location, causal path, triggering preconditions, and user/system impact. Consider at least one plausible counterexample or mitigating control before reporting it.
4. Gather evidence with available tools when a claim depends on code outside the diff. State which checks were actually run. A tool name, scanner label, or model intuition alone is not evidence that a finding is true.
5. Score each supported finding using the rubric below. Keep impact, likelihood, exposure, and confidence distinct in the explanation.
6. Review coverage. List changed files or ranges inspected, checks not available, and material portions that could not be examined. Do not claim complete coverage when context was truncated or tools were unavailable.
7. Return concise, actionable feedback. Separate confirmed defects from uncertain observations. If no actionable issue is supported, say so and describe any material review limitation.

## Finding Rubric

Score dimensions independently. If the host project supplies an established severity taxonomy or rubric, use it and map these dimensions to it instead of silently replacing it.

### Impact (I): 1-5

- `1`: Negligible impact; informational only. Do not report as a defect unless informational findings were requested.
- `2`: Minor, localized inconvenience or easily recoverable behavior issue.
- `3`: Material failure for a limited feature, input class, or subset of users; workaround exists.
- `4`: Serious security, data integrity, availability, or core-function impact with a credible reachable path.
- `5`: Critical compromise or broad impact, such as unauthenticated code execution, cross-tenant exposure, widespread data loss, or equivalent severity.

### Likelihood (L): 1-5

- `1`: Requires an exceptional or implausible sequence of conditions.
- `2`: Uncommon conditions; important preconditions remain.
- `3`: Plausible under ordinary use for a defined input or state.
- `4`: Likely in normal operation when the affected path is used.
- `5`: Directly triggered by routine or attacker-controlled input with no meaningful barrier.

Base likelihood on reachability and preconditions. Do not infer exploitability solely from a dangerous-looking API name.

### Exposure (E): 1-5

- `1`: One local/development context or negligible data/users.
- `2`: Narrow feature or small, bounded set of users/resources.
- `3`: A normal service component, team, or moderate data scope.
- `4`: Broad production surface, sensitive data, or many users.
- `5`: System-wide, cross-tenant, internet-facing critical, or hard-to-contain blast radius.

### Risk score: 0-100

Use this transparent prioritization formula:

```text
risk_score = round(20 * (0.45 * I + 0.35 * L + 0.20 * E))
```

Interpretation:

- `80-100`: critical priority
- `60-79`: high priority
- `40-59`: medium priority
- `20-39`: low priority
- `0-19`: informational; normally do not emit as a defect

This is a triage score, not a calibrated probability or a universal industry standard. Include the I/L/E values and a short rationale so a reviewer can challenge the score.

### Evidence confidence: 0-100

Compute confidence from observable support, not from how certain the model feels:

| Check | Points |
|---|---:|
| Location maps to the actual changed line and the quoted evidence is exact or clearly identified | 20 |
| A concrete causal path connects the changed code to the claimed failure/security impact | 25 |
| Required triggering preconditions and relevant surrounding behavior were checked | 20 |
| Independent corroboration exists, such as a relevant test/reproduction, AST/scanner result, or verified call-chain evidence | 25 |
| Plausible counterexamples or mitigating controls were considered and do not invalidate the claim | 10 |

Sum supported checks, assigning zero when a check was not established. Do not award corroboration points for a tool that was unavailable or not run. If context is missing, lower the score and state what is unknown. This score is an evidence-strength rubric, not a probability that the finding is true.

## Reporting Rules

- Report only defects introduced or materially worsened by the reviewed change, unless the user explicitly requests broader findings.
- Every finding must identify a file and line/range, include a concise code quote or evidence reference, explain a concrete impact, and suggest a minimal remediation and a verification test.
- Avoid duplicate findings. Merge only when they describe the same root cause and affected behavior; preserve distinct impacts or independent fixes as separate findings.
- Do not inflate severity to compensate for low confidence. Keep risk score and evidence confidence separate.
- Mark assumptions explicitly. If a critical precondition cannot be verified, report the uncertainty rather than presenting it as established fact.
- If the review is incomplete, return a coverage/limitations section even when findings were found.
- Do not claim a clean review proves absence of defects. Say "no supported issue found in the reviewed scope."

## Output

Use Markdown by default. When the caller requests machine-readable output, return valid JSON with this shape:

```json
{
  "summary": "Short review summary",
  "coverage": {
    "files_reviewed": ["src/example.py"],
    "checks_run": ["unit tests"],
    "limitations": []
  },
  "findings": [
    {
      "id": "stable-local-id",
      "title": "Short defect title",
      "category": "security|correctness|reliability|performance|compatibility|other",
      "severity": "critical|high|medium|low",
      "path": "src/example.py",
      "start_line": 42,
      "end_line": 42,
      "evidence": "Exact changed code or tool evidence",
      "impact": "Concrete affected behavior and scope",
      "preconditions": ["Verified or explicitly uncertain condition"],
      "risk_dimensions": {"impact": 4, "likelihood": 3, "exposure": 2},
      "risk_score": 63,
      "confidence": 75,
      "confidence_basis": ["Changed-line evidence", "Call path inspected"],
      "fix": "Minimal remediation",
      "test": "Regression test that would detect the defect"
    }
  ]
}
```

Use only severity labels supported by the host system. The example values illustrate the schema; calculate scores from the rubric for the actual finding.

## Feedback and Calibration

When a user marks a finding false positive, missed issue, incorrect severity, or bad fix, record the category and their rationale as feedback; do not silently rewrite the current review. Treat feedback as a hypothesis until verified against code or labeled evaluation data.

To claim that numeric confidence is calibrated, evaluate predictions against a representative, human-labeled validation set and compare observed correctness by confidence band. Keep a separate holdout set for regression checks. Without that measurement, label confidence as rubric-based evidence strength, not probability.
