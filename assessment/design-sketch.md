# Design sketch — honest skill assessment + small P0 fixes

> Produced per pstack:architect's Sketch/Agree phases, using the Option A/B/C
> analysis in `step-4-revamp-strategy.md` (already validated against real
> Gemini output) as Ground + Sketch input, screened against
> `architect/references/design-red-flags.md`. No multi-model arena was run —
> the design was already empirically validated in this session, and running
> one would spend budget re-deriving a conclusion already proven live.

## Ownership rule (fixes "split ownership" + "two ways to do one task")

`assessment_status` has exactly one writer: `Portfolios::Generator`, at
generation time. Every other reader (`FitGap::Engine`, the API serializers,
the PDF export, the frontend) reads the stored value. Nothing downstream
re-derives it from coverage data — that would be a second writer with its
own copy of the rule, which is the "split ownership" flag.

`ai_level` becomes `integer | nil` everywhere, full stop. The frontend's
`ai_level: string` annotation in `types/index.ts:93` does not match what the
backend has ever sent (confirmed: `portfolio_skill_json` emits the raw
integer column). `parseLevel`'s string-parsing branch was converting a type
that was never actually a string at runtime. Fix: correct the type, delete
the string-parsing branch, keep call sites reading the number directly.
Backend: no change needed, `ai_level` was already `integer` in the DB.

## Backend sketch

```ruby
# db/schema.rb (after migration)
create_enum "assessment_status", ["assessed", "insufficient_evidence", "not_assessed"]
# portfolio_skills gains:
#   t.enum "assessment_status", null: false, enum_type: "assessment_status"
#   t.integer "ai_level"              # now nullable
#   check_constraint "(assessment_status = 'not_assessed') = (ai_level IS NULL)"

# app/services/portfolios/skill_status.rb  (new, single-purpose, the one writer's helper)
module Portfolios
  class SkillStatus
    Result = Struct.new(:status, :level, :confidence, keyword_init: true)

    # coverage: CoverageMap for this skill, or nil if none exists.
    # raw_level: whatever the LLM returned for "level" (Integer, Float, String, nil).
    # evidence_count: Integer, already computed by the caller.
    def self.derive(coverage:, raw_level:, evidence_count:)
      raise NotImplementedError
    end

    def self.parse_level(raw_level)  # Integer in 1..5, or nil. No clamping.
      raise NotImplementedError
    end
  end
end
```

```ruby
# app/services/portfolios/generator.rb — changed shape
def save_skills(portfolio, response)
  # not implemented: wraps the whole method body in `portfolio.transaction do ... end`
  # per AC-13. Builds attrs via Portfolios::SkillStatus.derive, one call per
  # configured skill in @session.coverage_maps order (AC-5: missing skills from
  # the AI response are not dropped — iterate configured skills, not response keys).
end
```

```ruby
# app/services/fit_gap/engine.rb — changed shape
def build_skill_comparisons
  # not implemented: only ASSESSED or overridden skills compute delta/result
  # via level comparison. insufficient_evidence / not_assessed short-circuit
  # straight to a comparison hash with result == their own status, delta: nil.
  # Always emits: skill_label, skill_id, expected_level, candidate_level,
  # provisional_level, ai_level, result, delta, confidence, evidence_count,
  # is_override.  (AC-9 — is_override was never emitted before.)
end
```

```ruby
# api/app/models/session.rb
def invite_url
  base = ENV.fetch('WEB_BASE_URL')  # no silent localhost fallback in prod path;
  "#{base}/interview/#{invite_token}"  # Rails.env.production? guards fetch without default
end
```

## Frontend sketch

```ts
// types/index.ts
export type SkillAssessmentStatus = "assessed" | "insufficient_evidence" | "not_assessed";

export interface PortfolioSkill {
  // ...
  ai_level: number | null;          // was: string, wrong
  assessment_status: SkillAssessmentStatus;  // new
  evidence_count: number;           // new — count, not full quotes, for the table
}

export interface SkillComparison {
  // ...
  result: SkillComparisonResult | SkillAssessmentStatus;  // engine emits 5 values now
  provisional_level?: number;
  is_override: boolean;             // was optional+unused; now always present
}
```

```tsx
// components/fitgap/ComparisonTable.tsx — row rendering branches on result,
// not on candidate_level presence (fixes required_level -> expected_level typo
// in the same pass). not_assessed / insufficient_evidence get their own
// ResultBadge variants, not reused gap/match colors.
```

## Deviations log (Phase D requirement)

- Did not literally spawn the Arena skill's multi-model runners. Logged here
  instead of silently: the Option A/B/C trade-off table already in
  `step-4-revamp-strategy.md` fills that role, and was checked against real
  Gemini output in this session (ai_level=1, confidence=low, evidence=[] for
  an untouched skill, flowing into a false "gap"), which is stronger evidence
  than a second synthetic design pass would add.
