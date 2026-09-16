# benchmark.md — voicebench

the actual numbers. `research.md` said "score against real samples where available, LLM-judge otherwise" — this doc is what that means concretely: a parameter registry, a formula or proxy for each parameter, a status ladder tracking which ones are actually validated, and a benchmark suite (**voicebench**) that runs baselines against the same fixed test so "sounds like them" turns into a number that can go up or down and be argued with.

the governing idea, borrowed straight from stylometry: a voice isn't one similarity score. "voice match: 0.81" is exactly as meaningless as "style similarity: 0.81" — it's mark-making and shape language and color for an illustration; here it's code-switch rate and register and filler density. each one has to be independently measurable and independently movable before "we tuned the voice" means anything more than vibes with a progress bar.

---

## status ladder

every parameter below sits at one of four stages. nothing skips a stage.

| stage | means |
|---|---|
| **candidate** | named and hypothesized to matter, no formula yet |
| **measurable** | has a concrete formula/proxy, produces a number, not yet checked against ground truth |
| **validated** | checked against real samples or human judgment, and it actually tracks what people mean by that term |
| **controllable** | can be dialed up/down in the tuning engine and *only* that parameter moves — see delta vectors below |

v1's job is to get the parameters in the registry from candidate to measurable, and the highest-value few (code-switch rate, register classification, filler density) to validated. controllable is a v2 bar — it requires the tuning engine to expose that parameter as an independent dial, which most of `agent.md`'s current schema doesn't yet support per-field.

---

## parameter registry

grouped by layer, matching `research.md`'s three-layer model. `proxy metric` is what actually gets computed; `compared against` is the ground truth or reference it's checked against.

### lingvist

| parameter | proxy metric | compared against | status |
|---|---|---|---|
| code-switch rate | switches per 100 tokens | real scraped corpus | measurable |
| code-switch point accuracy | precision/recall on switch-point detection | hand-labeled eval set (`research.md` transliteration eval) | measurable, target for validated |
| code-switch reason accuracy | classification accuracy over {emphasis, quoting, no_clean_translation, in_group_signal} | same hand-labeled eval set | measurable |
| transliteration consistency | edit distance between synthetic and target's actual romanization choices, per loanword | real scraped corpus | candidate |
| loanword usage rate | frequency of specific loanwords per 100 words | real scraped corpus | measurable |

### lexica

| parameter | proxy metric | compared against | status |
|---|---|---|---|
| slang inventory overlap | jaccard similarity between target's real slang set and synthetic output's slang set | real scraped corpus | measurable |
| generational marker density | count per 100 words | real scraped corpus | measurable |
| filler/hedge rate | count per 100 words, broken out separately (fillers ≠ hedges, don't merge) | real scraped corpus | measurable, target for validated |
| lexical diversity | type-token ratio (TTR), or a length-corrected variant | real scraped corpus | candidate |
| idiom/metaphor family usage | frequency of idiom preferences actually appearing in synthetic output | real scraped corpus | candidate — blocked on the open "conflict/war metaphor" question in `research.md` |

### lingvica

| parameter | proxy metric | compared against | status |
|---|---|---|---|
| register classification | LLM-judge or trained classifier assigns {casual, consultative, frozen}, compare to labeled target register per medium | real scraped corpus, per medium | measurable |
| tone vector distance | cosine distance between {warmth, sarcasm_level, playfulness, confidence} as scored on synthetic output vs. target-labeled tone | real scraped corpus or LLM-judge | measurable |
| punctuation-as-tone density | count of caps/ellipsis/emphasis markers per 100 words | real scraped corpus | measurable |
| emoji/emoticon density | count per 100 words | real scraped corpus | measurable |
| channel-shift correction fidelity | for the voice channel specifically: did emoji density actually convert to vocal expressiveness rather than get dropped or read aloud — see below, this one needs audio | live interaction transcript + audio | candidate, highest-priority to get to measurable |
| prosody: speech rate | words per minute in actual call audio | no direct ground truth for a reconstructed phone voice — score against internal consistency (does it match the pacing implied by the target's punctuation density) | candidate |
| prosody: pause frequency/placement | pauses per minute, placement relative to clause boundaries | same caveat as above | candidate |

prosody parameters are flagged candidate rather than measurable because there's no real recorded voice-channel audio from the target to compare against — by definition, if the target had recorded voice-channel audio in a given register, that would just be scraped ground truth instead of a reconstruction problem. these get scored for internal consistency (does the audio match what the written-register profile implies) rather than accuracy against a target recording. this is a real limitation, not an oversight — see `research.md`'s honesty section.

### cross-cutting

| parameter | proxy metric | compared against | status |
|---|---|---|---|
| cross-medium consistency | for traits expected to be stable across mediums (e.g. core slang inventory), variance across medium-specific profiles; for traits expected to diverge (e.g. formality register), confirm they actually do | sandbox runs across ≥2 mediums | measurable |
| overfitting check | sandbox output accuracy on scenarios *not* represented in the source medium's typical content, vs. scenarios that closely match it | sandbox_runs, scenario-tagged | measurable |

---

## delta vectors

a single "voice match: 0.81" number hides which traits actually moved and which ones moved by accident. instead, every tuning run reports a **delta vector** across every parameter in the registry, not just the ones the call context asked for.

concretely: if a call context requests a shift toward higher code-switch rate for a specific audience, the delta vector should show something like

```
code_switch_rate:        moved 88% of the way toward target profile
filler_hedge_rate:       moved 3%   (unrelated — should stay near 0)
register_classification: moved 1%   (unrelated — should stay near 0)
```

the first number is the actual claim ("we tuned code-switch rate"). the second and third numbers are the collateral-change check — a tuning engine that can't move one parameter without dragging three others along with it isn't controllable yet, per the status ladder above, no matter how good the primary number looks. this is the same discipline stylometry-style benchmarks use to avoid claiming "style transfer" when what actually happened was "we changed everything a little."

---

## voicebench: the benchmark suite

a fixed test, run identically against every candidate configuration of the pipeline, so results are comparable instead of anecdotal.

**fixed inputs, held constant across all runs:**
- one scenario set (the same greeting/disagreement/scheduling-request/joke/bad-news set from `research.md`'s sandbox step)
- one eval set for code-switch detection (the hand-labeled set from `research.md`)
- one target corpus per test run, split into a training portion (used to build the voice profile) and a held-out portion (used only for scoring, never seen by the distillation pass)

**baselines, named by number, not by claim:**

| baseline | what it is |
|---|---|
| `baseline_000` | generic assistant voice, no target profile at all |
| `baseline_001` | naive persona prompt — "talk like {target}" with no structured profile |
| `baseline_002` | single collapsed persona vector — lingvist/lexica/lingvica merged into one blob before prompting, instead of kept separate |
| `yaptele_v0` | the actual three-layer pipeline in `research.md`/`agent.md`, with medium-shift correction applied |

`baseline_002` exists specifically to test the open question from `research.md` — does keeping the three layers separate through the whole pipeline actually outperform collapsing them early? if `yaptele_v0` doesn't beat `baseline_002` on the parameter registry above, that open question has an answer, and it's not the one the current architecture assumes.

**what voicebench actually evaluates, per run:**
- does it correctly identify which parameters should shift for a given call context (register, tone) — the primary claim
- does it leave unrelated parameters alone — the delta vector collateral check
- does it hold up on the held-out scoring portion, not just the training portion — the overfitting check
- does it correctly return "absent" rather than a guess when a call context requests a medium in `coverage.mediums_absent` — the scope-compliance check, borrowed from `scope.md`'s hard boundary, and arguably the single most important pass/fail check in the whole suite, since getting it wrong isn't a quality issue, it's a scope violation

---

## what this changes in the other docs

- `research.md` step 4 ("score") now means: run the parameter registry above, not an unspecified similarity score.
- `agent.md`'s `voice_profile.scoring` field and `sandbox_runs.score` in `applications.md`'s xano schema should store per-parameter scores from this registry, not a single aggregate number — the aggregate hides exactly the collateral-change problem the delta vector section above exists to catch.
- any claim in a submission or demo ("the voice sounds like them") should be traceable to a specific row in the parameter registry, at minimum "measurable," ideally "validated." a claim that can't be pinned to a row here is a vibe, not a result.
