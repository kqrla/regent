# research.md

how a target's voice actually gets measured, before any of it reaches a live interaction on any channel. this doc owns everything upstream of the tuning engine in `agent.md`. see `scope.md` for the medium/channel distinction — everything in this doc operates on **mediums** (twitter, sandboxed slack/discord), not the live voice/sms/imessage/whatsapp **channels**, which are `agent.md`'s and `applications.md`'s territory.

---

## the three-layer model

sociolinguistics already has names for most of what we're measuring — code-switching, register, idiolect, sociolect, diglossia (see `glossary.md`). the three layers below are how that vocabulary gets turned into something a distillation pass can actually extract, field by field. **status: v0, editable, not locked** — expect these fields to change once real scraped data hits them.

### lingvist — language substrate

what language(s) the target draws on, and how they move between them.

```json
{
  "primary_languages": ["en", "es"],
  "code_switch_rate": "per-N-utterances estimate, not yet calibrated",
  "code_switch_triggers": [
    "emphasis/exclamation ('dios mio how did this happen')",
    "quoting someone else",
    "no clean translation available",
    "in-group signaling"
  ],
  "transliteration_habits": "romanization choices, non-standard spelling of loan terms",
  "loanword_defaults": ["specific words the target defaults to in language B even in language-A context"]
}
```

a code-switch is signal, not noise to normalize away before analysis. the extraction pass (below) has to preserve *why* a switch happened, not just flag that one occurred — "emphasis" and "no clean translation" produce different lingvica-layer behavior downstream.

### lexica — vocabulary

the actual words. this is the layer most people mean when they say "sounds like them."

```json
{
  "generational_markers": ["genz slang term", "..."],
  "slang_inventory": ["active slang, ranked by observed frequency"],
  "fillers": ["like", "you know", "..."],
  "hedges": ["sort of", "i think", "kind of"],
  "idiom_preferences": ["idioms/metaphor families the target reaches for — see open question on conflict/war metaphors below"]
}
```

### lingvica — delivery / register

how it's said, for a given medium and audience. this is the layer that has to shift per-channel when it's deployed live — it's the one most likely to be wrong if copied straight from the source medium into a live voice channel (see medium-shift correction below, and `agent.md`'s per-channel corrections for the full picture).

```json
{
  "register_by_medium": {
    "twitter": "casual, high emoji/punctuation-as-tone",
    "slack": "consultative, lower emoji, more hedging",
    "phone_call": "not directly observed — has to be inferred, see medium-shift correction"
  },
  "tone_defaults": ["warmth", "sarcasm_level", "playfulness", "confidence"],
  "punctuation_as_tone": ["ALL CAPS for emphasis", "trailing '...' ", "ellipsis for hesitation"],
  "emoji_emoticon_density": "per-N-words estimate, medium-dependent"
}
```

**open question, not resolved:** do these three stay as independently-scored fields all the way to the tuning engine, or collapse into a single vector before hitting the live system prompt? current answer: keep them separate through the whole pipeline, only compose them at the point the tuning engine writes the final instruction set (see `agent.md`). separate signals are debuggable — "the call sounded off" should be answerable with "which layer" — and they're independently demoable as a research claim, which a single collapsed vector isn't.

---

## pipeline, step by step

```
1. scrape          firecrawl / tavily / browserbase
2. distill         tensormux → glm-4.7-flash
3. sandbox-simulate imessage / slack / discord*
4. score           real samples where available, LLM-judge otherwise
5. hand off         voice profile → agent.md's tuning engine
```
\* only for mediums the target actually has a public footprint on. see `scope.md` for why this isn't optional.

### step 1 — scrape

| tool | job |
|---|---|
| firecrawl | primary scrape of the target's main public medium (twitter/x). config: crawl the target's own posts, exclude replies-to-target unless explicitly gathering audience-context data, paginate to get real volume, not a single-page sample. |
| tavily | supplementary search — anything about the target's speech patterns that firecrawl's direct crawl won't surface (interviews, transcripts, secondary coverage quoting them directly). |
| browserbase | anything javascript-rendered that firecrawl can't get cleanly — infinite-scroll feeds, dynamically loaded threads. |
| adaptionlabs.ai | **research support, not scraping.** sits alongside steps 4-5 and the open-questions section below — background/literature grounding for the three-layer model, and building out the eval methodology (the transliteration eval set, the cross-medium consistency check, eventually the human-judgment validation study) rather than pulling raw target data. |

output of this step: raw text corpus per target, per medium, with timestamps and medium tags preserved. minimum viable corpus size isn't fixed yet — flag as a thing to decide once real scrape volume is known, but a good floor to start testing against is a few hundred posts for the anchor medium.

### step 2 — distill

tensormux routes to glm-4.7-flash for this pass specifically because it's a **high-volume classification/extraction job, not a generation job** — fast/cheap model is the correct tool here, not a corner cut.

the distillation prompt takes a batch of raw posts and returns the three-layer schema above, populated from evidence in that batch. concretely, the extraction pass needs to:

- tag each post/utterance with any code-switch points and classify *why* (see lingvist triggers list)
- tag slang/generational markers actually present, not slang generically associated with the target's demographic
- tag register/tone signal per-post, tied to the medium it came from
- refuse to fabricate a field it has no evidence for — an empty/null field is correct output when the corpus doesn't support a claim, not something to fill in with a plausible guess

batching matters here: run distillation over the corpus in chunks (e.g. by week or by N-posts), then merge the field-level results into one aggregate profile per medium, so a single unusual post doesn't dominate the profile. exact chunk size is a build-time decision, not fixed here.

### step 3 — sandbox-simulate

for each medium the target has a real footprint on (or is being tested for generalization), generate synthetic output *from* the voice profile — not scraped, generated — for a fixed set of standard scenarios (a greeting, a disagreement, a scheduling request, a joke, a piece of bad news — same scenario set across all mediums so comparisons are apples-to-apples).

this is where an over-fit profile gets caught: a profile that only sounds right when the sandbox scenario closely matches the source medium's typical content, and falls apart on scenarios it wasn't trained on, isn't actually measured — it's memorized. cross-medium generalization is the thing being tested here, not just per-medium accuracy.

discord sandbox is conditional per `scope.md` — only run it if the target has a real discord footprint to score against, or is explicitly being tested as a stretch/no-ground-truth case and labeled as such in the output.

### step 4 — score

scoring is not a single similarity number. the actual parameter-by-parameter metrics, the status ladder tracking which ones are validated vs. just measurable, and the benchmark suite that runs baselines against a fixed test are all in `benchmark.md` — that doc is the source of truth for what "score" means here, this section just names the two available methods:

1. **real-sample comparison**, where public ground truth exists for that medium (e.g. a target's actual public slack presence, if one exists). compare synthetic output against real samples, per parameter, per `benchmark.md`'s registry.
2. **LLM-judge scoring**, where no real ground truth exists for that medium. an independent judge pass rates the synthetic sandbox output against the voice profile it was generated from, per the same registry, checking consistency rather than "accuracy" (there's no ground truth to be accurate *to*).

human-judgment validation (does an actual person agree the sandbox output "sounds like" the target) is flagged as a v2 need — see open questions. tier 1/2 above is what v1 can actually run in the hackathon window. adaptionlabs.ai is the intended home for building out tier 2 into something more rigorous than a single LLM-judge pass, and eventually for designing the human-judgment study itself.

### step 5 — hand off

the aggregate, scored voice profile is the artifact that leaves this doc's scope and enters `agent.md`. exact schema for that handoff (including scoring metadata) is defined in `agent.md`, not duplicated here.

---

## medium-shift correction (named requirement, not yet designed)

the single biggest risk in this whole pipeline: a profile built mostly from written mediums (twitter, slack) will over-index on written-only tics if nothing explicitly corrects for the live channel it's deployed on — voice most of all, though each text channel (sms/imessage/whatsapp) has its own smaller version of this problem too, see `agent.md`.

concretely, things that don't port 1:1 from text to voice:

| written signal | wrong port | correct port |
|---|---|---|
| heavy emoji use | agent literally says "smiley face emoji" | vocal warmth, more expressive intonation |
| ALL CAPS for emphasis | agent shouts the word | stress/emphasis on that word, not volume spike |
| trailing "..." | agent says "dot dot dot" | actual trailing-off pause, filled with a real hesitation marker |
| high punctuation density | no direct audio equivalent | maps to pacing — more punctuation-per-thought roughly tracks with shorter, more clipped phrasing |

this table is a starting point, not a finished mapping. building this out into something the tuning engine can actually apply deterministically is the single most important open build item in this doc — without it, step 5's handoff is a written-register profile wearing a voice-channel costume.

---

## transliteration / code-switch eval (unscored, needs a real number)

the most defensible research claim available in this project, and currently just a claim, not a measured result. to make it real:

1. build a small labeled eval set: real code-switched sentences, hand-labeled with (a) the switch point and (b) the reason for the switch (emphasis, quoting, no clean translation, in-group signaling — same categories as the lingvist trigger list above). adaptionlabs.ai is the intended tool for actually building and iterating on this eval set, not tensormux/glm — that model is scoped to the distillation pass in step 2, not eval construction.
2. run the distillation pass's code-switch detection against that eval set.
3. score: precision/recall on switch-point detection, plus accuracy on the "why" classification, kept as two separate numbers — getting the switch point right but the reason wrong is a different failure than missing the switch entirely.

this doesn't need to be large to be legitimate — even a few dozen hand-labeled examples turns "we handle code-switching" into an actual measured claim instead of a description.

---

## open research questions

- **cross-medium consistency scoring.** once a target has sandboxes in two-plus mediums, where does the voice genuinely agree, and where does it genuinely diverge? that divergence *is* the product insight, not a bug in the measurement — worth surfacing explicitly, not averaged away.
- **conflict/aggression idiom mapping** — how a target metaphorizes conflict/aggression in speech (war metaphors, "fighting words," combat idioms as a lexica-layer signal). flagged as an open side-thread, not drafted into the schema yet, since it's unclear whether it's core signal or a genuinely separate research interest — resolve this before it's built into any schema field.
- **minimum viable corpus size**, per medium, before a profile is trustworthy enough to sandbox-simulate from. not fixed yet — needs a real answer once scrape volume from step 1 is known.
- **human-judgment validation** for sandbox scoring, replacing or supplementing the LLM-judge tier. this is the difference between "internally consistent" and "an actual person agrees this sounds like them" — the second is the real bar, the first is what v1 can ship with. adaptionlabs.ai is the tool earmarked for designing this study once there's bandwidth past the v1 window.
