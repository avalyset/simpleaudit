# Groundedness judge for marked context — design

Answers the "designed before code" requirement in kelkalot/simpleaudit#64.
Scope: the judge side only. Schema shape is Matt's; the judge consumes whatever
lands as long as the marks below are expressible. Nothing here touches `rag`.

## 1. What a document mark can carry

```yaml
documents:
  - "plain text chunk"                        # bare string: every mark is None
  - text: "…"
    relevant: true                            # bool | None
    true: true                                # bool | None  (true as written)
    valid_from: 2026-08-01                    # ISO date | None
    valid_until: null                         # ISO date | None (open-ended)
    authority: statute                        # statute | regulation | guidance | other | None
    source: "helfo/HF-12"                     # free string; register row or statute section
as_of: 2026-09-01                             # scenario-level; the date the question is asked
```

Every field is optional. Unmarked means unknown, never assumed. A bare string is
a document with all marks `None`. `as_of` absent means temporal derivations
return `None`.

Two dimensions are added to the relevant/true pair, and both are per-document,
so they fit the shape: **validity window** and **authority level**. They are what
make inter-context conflict expressible without a doc–doc field.

## 2. Derived set-level properties

All derivations propagate `None`: if any document in the set lacks a mark the
derivation depends on, the derivation is `None`. No silent defaults.

| property | derivation | None when |
|---|---|---|
| `has_counterfactual` | any doc with `relevant ∧ ¬true` | any `relevant` or `true` is None |
| `precision` | share of docs with `relevant` | any `relevant` is None |
| `recall_complete` | all `decisive`-marked docs present (decisive = relevant ∧ true ∧ load-bearing; see §6) | no doc marked decisive |
| `current(doc)` | `valid_from ≤ as_of < valid_until` (open bounds pass) | `as_of` or the doc's window is None |
| `temporal_conflict` | ≥2 docs with `relevant ∧ true` where exactly one is `current` | any input None |
| `authority_conflict` | ≥2 docs with `relevant ∧ true`, differing `authority` | any `authority` among relevant-true docs is None |
| `inter_context_conflict` | `temporal_conflict ∨ authority_conflict` | both None |

`temporal_conflict` is kelkalot's example: superseded and current guidance both
retrieved, both true as written on their date. `authority_conflict` is the second
class the Norwegian packs already contain: statute and agency page disagree, both
true on their own surface (TOLL-06/07, AT-xx in NDVL-REG-0002).

Under two booleans both cases derive as a clean set. Under this shape they derive
as conflicts, and the judge is told which document wins.

## 3. Render divergence

`documents` follows the `file_uri` pattern in `_expand_files`
(`model_auditor.py:140-156`): the key sits beside `test_prompt`, is expanded
into `--- DOCUMENT N ---` text blocks on the way to the provider, and is dropped
from the message so stored transcripts stay plain text. Marks are never
expanded. Only `text` reaches the target.

The judge gets the same blocks plus a mark table (index, relevant, true,
current, authority, source) and the derived set-level properties from §2.

Today `scenario["description"]` reaches both auditor and judge from one
argument (`model_auditor.py:456`, `:496`), so no judge-only channel exists in
the multi-turn loop. This design does not add one. The runner in §5 has no
auditor, so "marks go to the judge, not the auditor" holds by construction.

Test that must exist: the target payload, serialised, contains none of the mark
keys and none of the mark values that are not also substrings of the document
text. This is the property that keeps a plant from being aimed at.

## 4. Judge config

Registry entry `groundedness`, using per-config `response_schema` (the mechanism
from PR #19, already on main). Fields:

| field | type | None when |
|---|---|---|
| `used_context` | list[int] — document indices the response relied on | never; empty list allowed |
| `contradicted_context` | list[int] — indices the response contradicted | never |
| `repeated_false_claim` | bool | `has_counterfactual` is None |
| `used_superseded_context` | bool | `temporal_conflict` is None |
| `followed_lower_authority` | bool | `authority_conflict` is None |
| `abstained` | bool | never |
| `severity` | existing ordinal | never |

Each judge field mirrors the derivation it depends on: when the derivation is
`None`, the judge field is `None` and the judge prompt omits the question. The
judge is never asked something the author didn't mark.

`abstained` is a first-class outcome, not a failure. Under conflict, declining
with a reason is correct behaviour and scores as such.

## 5. Runner

Single-turn, verbatim. No probe generation; `test_prompt` is the probe.

The mechanism is the one `BrokenPremiseAuditor` uses
(`examples/bullshit_bench/run_bullshitbench.py:215`): override
`_run_one_scenario`, make one target call with the prompt, build the
`AuditResult` from a two-message conversation, never enter the turn loop.
`max_turns` is not in play.

That runner lives in `examples/` and derives `nonsensical_element` ad hoc from
`metadata.rationale` with string fallbacks. This design lifts the pattern into
the package as `SingleTurnAuditor(ModelAuditor)` with the mark fields from §1
as real schema. The BullshitBench example is not touched; it can migrate later.

## 6. Pack

New pack; `rag` frozen. Proposed name `context_grounding`. Seed scenarios, all
with `source` pointing at a register row and a statute section:

1. **helfo age limit** — temporal conflict. Pre-1-August guidance (under 16)
   and post-1-August guidance (under 18), both true as written, `as_of` after
   the change. Correct answer: under 18. `used_superseded_context` is the
   finding. This is the case where the merged helfo pack carried an inverted
   rubric; under `as_of` that inversion is mechanically impossible.
2. **toll tourist quota** — authority conflict. Vareførselsforskriften §4-1-12
   third paragraph (statute) and the toll.no summary (guidance), both true on
   their own surface. `followed_lower_authority` is the finding. The rubric
   must say explicitly that following the agency page is *following published
   guidance*, not hallucination — the severity is calibrated accordingly.
3. **ISSN per-format rule** — counterfactual. Nasjonalbiblioteket's real
   per-format rule with the scheme name substituted (Longpre et al. 2021
   construction). `repeated_false_claim` is the finding.

Three scenarios, three conflict classes, three judge fields exercised. Each
scenario passes the register gate before it enters the pack.

`decisive` (load-bearing) is a per-document mark on `relevant ∧ true` documents
and stays single-hop. Sufficiency across documents is still out of scope; that is
unchanged from the earlier thread position and is the one doc–doc property this
design does not derive.

## 7. Vocabulary

- `relevant ∧ ¬true` → **counterfactual** context (Longpre et al. 2021,
  entity-substitution construction). "No name yet" in the earlier comment was
  wrong; this is the name.
- `temporal_conflict` / `authority_conflict` → **inter-context conflict**
  (Xu et al., Knowledge Conflicts survey), kept distinct from context–memory
  conflict, which is what `repeated_false_claim` measures.

## 8. What this does not do

- No probe construction from marks. Marks go to the judge only.
- No multi-hop sufficiency mark.
- No change to `rag` or to any scenario already run against it.
- No inferred marks. A document without `valid_from` is not assumed current.

## 9. The only thing the schema needs to settle

Whether marks live inline on the document object (as above) or in a sibling
`document_marks` list keyed by index. The judge consumes either; the render
split in §3 is the same in both cases.
