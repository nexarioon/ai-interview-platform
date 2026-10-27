# Step 4 — Revamp Strategy: Honest Skill Assessment

> Draft. Finding IDs refer to the Step 3 findings list.

## 1. The problem in one paragraph

The fit/gap report is the output a recruiter acts on. Today it cannot tell the difference between "the candidate is weak at this skill" and "the system never asked about this skill". A skill the interview never reached (time ran out, coverage state `not_yet`) is forced to a level by the portfolio generator, defaults to **L1**, and the fit/gap engine then reports it as **Gap −2** with nothing on screen to show it was never assessed. The same screen also hides the "Required" level (frontend/backend field mismatch) and never marks human overrides, even though its legend promises to. The candidate is penalised for a gap in the system, not a gap in their ability.

Reproduced locally with seeded data: a skill with `confidence: low` and zero evidence rendered as `⚠ Gap -2`. The narrative read "1 skill matches … and 1 gaps" and did not mention unassessed skills.

## 2. Root causes

| # | Layer | Location | Defect |
|---|---|---|---|
| R1 | Generation | `api/app/services/portfolios/generator.rb:161,173` | Every skill must get a level 1–5. A missing level becomes L1 via `to_i.clamp(1, 5)`. **`"L3".to_i` is 0, so an AI answer of `"L3"` is stored as L1.** |
| R2 | Generation | `generator.rb` prompt | Confidence comes from the LLM's own report, although the rule it is asked to follow (probe count, coverage state) is data we already hold in `coverage_maps`. |
| R3 | Computation | `api/app/services/fit_gap/engine.rb:46-50` | Match/gap/exceed ignores confidence and evidence. A guessed L1 counts the same as a proven L1. |
| R4 | API contract | `engine.rb:58-66` vs `web/src/types/index.ts:131-138` | Backend emits `expected_level`; frontend reads `required_level`, so "Required" is always blank. Engine computes `overridden` but never emits `is_override`. The PDF export reads `expected_level`, so web and PDF disagree. |
| R5 | Input | `web/src/components/assessment/SkillPicker.tsx` | `skill_id: undefined` drops the taxonomy ID. All matching falls back to label text, and the generator prompt labels every taxonomy skill "custom". |
| R6 | Presentation | `web/src/components/fitgap/ComparisonTable.tsx` | No confidence, evidence count or provenance shown. The summary leaves out not-assessed skills. |

## 3. Assumptions

1. **The coverage map is the source of truth for what was discussed.** `Coverage::StateEngine` already requires ≥ 2 probes before a skill can reach `partial`, so "assessed" reuses that rule instead of inventing a new threshold.
2. **The LLM is a judge of level, not of evidence sufficiency.** It may suggest a level; whether that level counts is decided by deterministic code.
3. **An assessor override is a human judgement** and may be applied to any skill, including one the AI could not assess. It is shown as such, never blended in silently.
4. No production Gemini key is available. AI responses are simulated with an injected fake client (`Portfolios::Generator` and `FitGap::Engine` already accept `gemini_client:`).
5. Existing rows may exist in production. The migration must be safe on them and must be reversible.

## 4. Domain model: assessment status per skill

Each configured skill ends a session in exactly one status. The status is derived in code from data we own, evaluated top to bottom:

| Status | Rule | Used in match/gap? |
|---|---|---|
| `not_assessed` | No coverage map row, **or** coverage state `not_yet`, **or** the AI returned no valid level | No. Shown as "Not assessed: needs follow-up" |
| `insufficient_evidence` | Coverage state `initiated` (probed once), **or** zero evidence quotes | No. The AI level is shown greyed out as *provisional* |
| `assessed` | Coverage state `partial` or `covered` (≥ 2 probes), a valid level, and ≥ 1 evidence quote | Yes |

Confidence is computed from the coverage map using the rule the prompt already states, instead of being read from the LLM: `high` when covered with ≥ 3 probes, `medium` when partial or 2 probes, `low` otherwise.

An **assessor override** turns any status into a human-assessed result. Its result is computed from `override_level`, flagged `is_override: true`, and shows the AI level next to it ("AI L4 → Assessor L3").

## 5. Acceptance criteria

Each criterion has an ID so tests can reference it.

### Portfolio generation (backend)

- **AC-1** A configured skill whose coverage state is `not_yet` is stored as `not_assessed`, whatever level the AI returns for it.
- **AC-2** A skill probed once (`initiated`), or with zero evidence quotes, is stored as `insufficient_evidence` and keeps the AI level as provisional.
- **AC-3** Level parsing accepts `3`, `"3"` and `"L3"` (case-insensitive) as 3. `3.0` counts as 3. `3.7`, `0`, `6`, `-1`, `""`, `null`, `"three"` and any other value count as **no valid level**, so the skill becomes `not_assessed`. Values are never clamped into range.
- **AC-4** Confidence is computed from the coverage map (§4). An AI confidence value such as `"High"` or `"very high"` cannot make generation fail.
- **AC-5** A configured skill missing from the AI response is stored as `not_assessed`, not dropped.
- **AC-6** Skills are matched to the AI output by `skill_id` first and by normalised label second (trimmed, case-insensitive, collapsed whitespace). Two configured skills with the same normalised label are rejected when the assessment or vacancy is saved.
- **AC-7** Taxonomy skills keep their `skill_id` from the picker through to `assessment_skills` and `vacancy_skills`. Custom skills keep `skill_id: null`.

### Fit/gap computation (backend)

- **AC-8** Only `assessed` skills and overridden skills produce `match`, `gap` or `exceed`. `insufficient_evidence` and `not_assessed` produce results of the same name, with `delta: null`.
- **AC-9** Each comparison contains: `skill_label`, `skill_id`, `expected_level`, `candidate_level` (null unless the result is match, gap or exceed), `provisional_level`, `ai_level`, `result`, `delta`, `confidence`, `evidence_count`, `is_override`.
- **AC-10** The fallback narrative (used when the Gemini call fails) states the counts for all five results with correct singular/plural forms. It is stored as the overall narrative only, never under "Culture & Competency Fit".
- **AC-11** The narrative prompt says that not-assessed and insufficient-evidence skills are not evidence of weakness.
- **AC-12** A vacancy with zero skills produces a report with an empty comparison list and an explanatory message. It does not raise `RecordInvalid`.

### Failure paths

- **AC-13** If any skill fails to save, no skill from that run is persisted: generation is one transaction, and the portfolio is marked `failed`.
- **AC-14** Two generation runs for the same session, such as a Sidekiq retry and a manual "Retry", leave exactly one consistent set of skills. The portfolio row is locked for the duration of the write.
- **AC-15** Two concurrent fit/gap generations for the same portfolio and vacancy leave exactly one report and never return a 500. A unique-index conflict is retried once.
- **AC-16** `generation_error` stores the error class and a generic message. It never contains raw model output, transcript text or evidence quotes, because those are the candidate's personal data.

### Data migration

- **AC-17** The migration adds `assessment_status` to `portfolio_skills` (`assessed | insufficient_evidence | not_assessed`, NOT NULL) and backfills existing rows: `not_assessed` where the session's coverage map shows `not_yet` or has no row, `insufficient_evidence` where confidence is `low` or evidence is empty, otherwise `assessed`.
- **AC-18** `ai_level` becomes nullable, with a check constraint that it is NULL exactly when the status is `not_assessed`.
- **AC-19** `down` succeeds when no `not_assessed` rows exist. When they do, it stops with a clear message instead of deleting or inventing data.
- **AC-20** Reports generated before the change, which lack the new fields, still render. The frontend shows a "Generated with an older version — regenerate for confidence details" banner instead of blanks.

### Frontend presentation

- **AC-21** The "Required" column shows the expected level for every row.
- **AC-22** An overridden row shows a pencil marker and "AI L4 → Assessor L3". Notes are reachable on hover or tap.
- **AC-23** `insufficient_evidence` rows show the provisional level greyed out with "Low evidence". `not_assessed` rows show "Not assessed" with the hint "Not covered in the interview — consider a follow-up". Neither uses the colour or icon of a gap.
- **AC-24** The summary row counts all five results. When at least one skill is not assessed or has insufficient evidence, the header shows "N of M skills could not be fully assessed".
- **AC-25** Each assessed row shows confidence and evidence count.
- **AC-26** Loading, generating, failed and error states each have a distinct UI. A non-404 error shows an error message with a retry, not an empty page.
- **AC-27** Long text: skill labels up to 255 characters wrap without breaking the table; competency summaries over three lines are clamped with "Show more".
- **AC-28** At 375 px width there is no page-level horizontal scroll. The table either scrolls inside its container or becomes a card list.

## 6. Options considered

### Option A — Persisted assessment status, computed in code (recommended)

Add `assessment_status` to `portfolio_skills` and set it in the generator from the coverage map. The engine, the API, the PDF export and the UI all read the stored status.

### Option B — Derive status at read time, no migration

Keep the schema. Stop clamping in the generator. The engine, `portfolio_json` and `pdf_generator` each derive status at read time from confidence, evidence and the coverage map.

### Option C — Frontend-only relabelling

Show confidence in the table and restyle low-confidence gaps as "verify". No backend change.

| | A — Persisted status | B — Read-time derivation | C — Frontend only |
|---|---|---|---|
| **Fixes the wrong verdict** | Yes, everywhere: web, PDF, JSON export, narrative | Yes, if all three read paths stay in sync | Web table only. PDF, JSON export and narrative still say "gap" |
| **Fixes `"L3"` → L1 parsing** | Yes | Yes | No |
| **Auditability** | Status is fixed when the report is generated; later rule changes do not silently rewrite past verdicts | Past reports change whenever the rule changes | None |
| **Cost** | Medium: one migration with backfill, generator, engine, contract, UI | Lower: no migration, logic in three places | Lowest |
| **Maintainability** | One place decides status. New consumers just read a column | Three call sites must share one helper or drift apart | Business rule lives in the view layer |
| **Reversibility** | Reversible migration (AC-19). Rollback leaves old behaviour intact | Trivial to revert | Trivial to revert |
| **Failure modes** | Wrong backfill for legacy rows (mitigated by a conservative rule and a dry-run count) | Read paths drift; a heavier read path on every request | Users trust the PDF, which still shows a gap |
| **Forecloses** | Nothing; status can later carry more values | Hard to add history or reporting on status later | Leaves the core defect in place |

### Decision: Option A

The report is the record of a hiring decision about a person. Freezing the status when the report is generated means a candidate's verdict cannot change silently later, and one place in the code decides it. That matters for an explainable result under UU PDP: Article 10 of Law No. 27 of 2022 gives data subjects the right to object to decisions based solely on automated processing *(verify exact article wording before submission)*. The extra cost over B is one reversible migration. Option C was rejected because the PDF a hiring manager forwards would still label an unasked skill a gap.

## 7. Out of scope (documented, not fixed here)

- Cross-tenant access to portfolios, overrides and fit/gap reports (Step 3 constraint signal, P0). Fixing it properly needs `tenant_id` on seven tables and should be its own reviewed change.
- Fit/gap job failure status column and an exhausted-retries handler.
- Re-interview or follow-up scheduling for not-assessed skills. This change surfaces the need but does not automate it.

## 8. Test plan

| Area | Test type | Covers |
|---|---|---|
| Level parsing, status derivation | RSpec unit (pure functions) | AC-1 – AC-6 |
| Generator with fake Gemini client: malformed JSON, missing skills, bad confidence, mid-loop failure | RSpec service | AC-5, AC-13, AC-16 |
| Engine result matrix (status × override × level delta) | RSpec service, table-driven | AC-8 – AC-12 |
| Duplicate and concurrent runs | RSpec with two threads | AC-14, AC-15 |
| Migration up, down and backfill | RSpec migration spec on seeded rows | AC-17 – AC-19 |
| Fit/gap API response shape | RSpec request spec | AC-9 |
| `ComparisonTable` states, legacy report, long text | Vitest + React Testing Library | AC-20 – AC-27 |
| 375 px layout | Manual check with screenshots in the report | AC-28 |

**Seeded fault test candidate:** on a scratch branch, change the engine so `insufficient_evidence` falls through to the level comparison. The AC-8 table-driven spec must fail. Then revert, with both commits left visible in history.
