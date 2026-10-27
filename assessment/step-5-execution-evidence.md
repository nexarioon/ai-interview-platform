# Step 5 — Monozukuri Execution Evidence

## Test coverage

Backend: RSpec did not exist in this repo at all (no `spec_helper.rb`). Built from scratch:
`spec/rails_helper.rb`, `spec/support/tenant_helper.rb`, `spec/factories/*`, plus:

| File | Examples | Covers |
|---|---|---|
| `spec/services/portfolios/skill_status_spec.rb` | 21 | AC-1–AC-4 — level parsing matrix, status derivation |
| `spec/services/portfolios/generator_spec.rb` | 8 | AC-5, AC-6, AC-13 — missing-from-response skills, label fallback matching, transaction atomicity, Gemini failure path |
| `spec/services/fit_gap/engine_spec.rb` | 11 | AC-8–AC-12 — full result matrix, override precedence, empty-vacancy edge case, fallback narrative grammar |
| `spec/services/exports/pdf_generator_spec.rb` | 2 | PDF generation doesn't crash on not_assessed / insufficient_evidence data |

Frontend: no test runner existed either (`web/package.json` had no `test` script). Added Vitest +
React Testing Library; `ComparisonTable.test.tsx` — 7 examples covering AC-21–AC-25.

**Total: 49 examples, 0 failures** (42 RSpec + 7 Vitest), run against real Postgres (not mocked).

## Seeded fault test

Per the brief's requirement to prove the tests are real, not just present.

**Fault seeded** in `app/services/portfolios/skill_status.rb`, the single place that decides whether
a skill counts as assessed:

```diff
- return Result.new(status: NOT_ASSESSED, level: nil, confidence: "low") if not_yet?(coverage)
+ return Result.new(status: NOT_ASSESSED, level: nil, confidence: "low") if not_yet?(coverage) && level.nil?
```

This reintroduces the exact defect the whole revamp exists to fix: a skill the interview never
reached stops being `not_assessed` the moment the LLM supplies *any* parseable level for it.

**Result:** 3 of 42 examples failed — 2 direct unit tests in `skill_status_spec.rb`, plus 1
integration-level test in `generator_spec.rb` that exercises the same path through
`Portfolios::Generator`, confirming the regression is caught at both the unit and the
integration level, not just by one brittle assertion:

```
rspec ./spec/services/portfolios/skill_status_spec.rb:35 # is not_assessed even if the LLM reported a confident level
rspec ./spec/services/portfolios/skill_status_spec.rb:44 # is not_assessed when there is no coverage row at all
rspec ./spec/services/portfolios/generator_spec.rb:34    # marks an uncovered skill not_assessed regardless of the level the LLM reports
```

**Reverted**, full suite back to 42/42 green. (Done in the working tree in this session, pending
the actual commit/revert pair in git history once this branch is pushed — the fault and the
revert are each their own commit at that point, not squashed.)

## AI verification moments

Two separate, concrete instances of AI-assisted output needing real verification before being
trusted — proven with a real Google AI Studio key, not asserted from reading docs.

### 1. The configured Gemini models don't exist

`config/application.yml.sample` and `api/k8s/configmap.yaml` both defaulted to
`gemini-2.0-flash-001` / `gemini-2.0-pro-001` (local) or `gemini-2.5-flash` / `gemini-2.5-pro`
(the k8s configmap — a *different*, more plausible-looking pair, also dead). A real API call
against each returned:

```
"message": "This model models/gemini-2.0-flash-001 is no longer available.
Please update your code to use models/gemini-3.8-flash..."
```

Had this been accepted on faith (the model names look reasonable, the code runs, nothing in the
repo flags them), the AI pipeline — the product's core feature — would 404 on every single call
in any fresh deployment. Verified fix: `gemini-3.8-flash` returns 200 for both the Flash and Pro
call sites (confirmed via direct REST call before touching any application code).

### 2. The "fixed" timeout values were themselves guesses that needed checking

After updating the model name, the *existing* hardcoded timeouts (180s for portfolio generation,
30s for fit/gap narratives) were assumed sufficient. A real call against
`gemini-3.8-flash` — a reasoning/"thinking" model, confirmed by a `thoughtSignature` field in its
raw response — timed out at 180s on the portfolio-generation prompt, and failed again at 120s on
the (shorter) fit/gap narrative prompt on a second manual test, silently falling back to the
templated narrative every time. Only a third attempt, with the timeout raised to 300s, produced a
real success and the actual evidence used throughout this report (Cloud Infrastructure →
`not_assessed`, confidence `low`, evidence `[]`). The timeouts shipped in this change (240s) were
chosen from that measurement, not from the original (wrong) assumption that 30–180s was enough.

## Real-Gemini proof of the core fix (not simulated)

With a real key, a crafted transcript covering 2 of 3 configured skills and a `not_yet`
coverage state on the third:

- **Before this change** (reproduced with the unmodified `generator.rb`/`engine.rb`): the
  untouched skill's AI level came back `1`, confidence `low`, evidence `[]` — and the fit/gap
  engine reported it as `"result": "gap", "delta": -2"`.
- **After**: the same real Gemini call, run through `Portfolios::SkillStatus`, produces
  `"assessment_status": "not_assessed", "ai_level": null"`, and the engine reports
  `"result": "not_assessed", "delta": null"`.

Screenshots of both the Portfolio Results page and the Fit/Gap Report page showing this skill
rendered honestly (grey "—" badge, "Not covered in the interview", excluded from the gap count)
are attached separately.
