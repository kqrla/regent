# applications.md

the actual system underneath `research.md`, `benchmark.md`, and `agent.md`. this doc owns architecture, tool responsibility, the identity graph, backend schema, api surface, and deployment.

---

## architecture, end to end

```
research side (mediums)                    live side (channels)
------------------------                    ---------------------
scrape                                       one phone number
  ↓                                             ↓
distill                                      channel router
  ↓                                             ↓  (looks up contact_id in falkordb)
sandbox + score                              falkordb identity graph  ←→  mem0 (working memory)
  ↓                                             ↓
voice profile      ─────────────────────→   deterministic tuning engine
                                                 ↓
                                              instruction_set
                                                 ↓
                                              channel adapter: voice | sms | imessage | whatsapp
```

xano remains the system of record for the *research* side — scrape runs, voice profiles, sandbox scores. falkordb is the system of record for the *live* side — it's not a call-memory add-on anymore, it's the identity graph the whole number resolves against. see the section below before assuming these two stores overlap; they don't, and they shouldn't.

---

## tool responsibility table

| tool | stage | v1 or v2 | notes |
|---|---|---|---|
| firecrawl | scrape (research mediums) | v1 | primary scrape, target's anchor medium |
| tavily | scrape (research mediums) | v1 | supplementary search for context/interviews/transcripts |
| browserbase | scrape (research mediums) | v1 | js-rendered pages firecrawl can't reach cleanly |
| adaptionlabs.ai | eval / research support | v1 (eval), v2 (human-judgment study design) | not a scrape tool — builds and iterates the transliteration eval set and the sandbox scoring methodology, see `research.md`. does not touch raw target data |
| tensormux → glm-4.7-flash | distill | v1 | high-volume extraction pass, not a generation pass |
| xano | research backend | v1 | system of record for scrape runs, voice profiles, sandbox scores |
| falkordb | identity graph | v1 | the number's identity backbone — contacts, channels, prior interactions across all of them |
| mem0 | working memory | v1 | conversational memory layered on top of the falkordb graph, per interaction |
| channel adapters (voice, sms, imessage, whatsapp) | live delivery | v1 (voice + 1 text channel), v2 (remaining channels) | deliver the tuning engine's instruction_set on each channel; specific provider/SDK choice per channel is an implementation decision, not fixed in this spec |
| tokenfactory (nebius) | fine-tune | v2 | training infra for cadence/prosody matching on the voice channel |
| amd developer cloud | fine-tune | v2 | hardware for the above |

---

## the identity graph (falkordb)

this is the part that changed the shape of the whole system: falkordb isn't a memory cache anymore, it's the thing that makes "one number, every channel" true.

**node types:**
- `Number` — the one phone number the agent operates on
- `Contact` — a person who's interacted with the number, on any channel
- `Channel` — voice, sms, imessage, whatsapp
- `Interaction` — a specific call, text, imessage thread, or whatsapp thread
- `TargetVoiceProfile` — the reconstructed voice being used (from `agent.md`'s voice_profile schema)

**edges:**
- `Contact —[USED]→ Channel` (this contact has interacted via this channel)
- `Contact —[HAD]→ Interaction` (history, across all channels)
- `Interaction —[ON]→ Channel`
- `Interaction —[USED_VOICE]→ TargetVoiceProfile`

resolving `interaction_context.contact_id` (from `agent.md`) means walking this graph: does the inbound number/handle on this channel already match a known `Contact` node, and if so, what's their interaction history across *other* channels too — not just this one. that cross-channel continuity is the actual point of having one number instead of four disconnected integrations.

mem0 sits above this graph as the working-memory layer for a single active interaction — it's not a separate identity store, it reads and writes against the same `Contact`/`Interaction` nodes falkordb already holds.

---

## xano schema (research side only)

unchanged in shape from the research pipeline — falkordb, not xano, now owns anything to do with live contacts or channels, so this schema stays scoped to `research.md`'s pipeline.

### `targets`
| field | type | notes |
|---|---|---|
| id | pk | |
| display_name | string | |
| public_figure | boolean | gates which consent path applies per `scope.md` |
| mediums_scraped | array[string] | research mediums, not live channels |

### `scrape_runs`
| field | type | notes |
|---|---|---|
| id | pk | |
| target_id | fk → targets | |
| tool | enum(firecrawl, tavily, browserbase) | |
| medium | string | research medium |
| raw_corpus_ref | string/blob ref | reference, not the corpus inline |
| run_at | timestamp | |

### `voice_profiles`
| field | type | notes |
|---|---|---|
| id | pk | |
| target_id | fk → targets | |
| lingvist / lexica / lingvica | json | schema per `agent.md` |
| coverage | json | mediums_with_real_data / sandbox_only / absent |
| scoring | json | per-parameter scores, per `benchmark.md` |
| source_scrape_runs | array[fk → scrape_runs] | traceability back to raw data |
| generated_at | timestamp | |

### `sandbox_runs`
| field | type | notes |
|---|---|---|
| id | pk | |
| voice_profile_id | fk → voice_profiles | |
| medium | string | research medium (sandbox) |
| scenario | string | fixed scenario set, `research.md` |
| synthetic_output | text | |
| score_method | enum(real_sample, llm_judge) | |
| score | object | per-parameter, per `benchmark.md` — not a single scalar |
| real_sample_ref | string, nullable | present only when score_method = real_sample |

### `eval_results`
| field | type | notes |
|---|---|---|
| id | pk | |
| eval_type | enum(code_switch, cross_medium_consistency, human_judgment) | |
| target_id | fk → targets, nullable | |
| metric | string | e.g. "switch_point_precision" |
| value | number | |
| notes | text | |

---

## api surface

**research side (xano-backed)**
- `POST /targets/{id}/scrape` — trigger a scrape run for a research medium
- `POST /voice_profiles` — run distillation over a target's scrape_runs
- `POST /voice_profiles/{id}/sandbox` — run sandbox simulation
- `POST /eval_results` — record a code-switch or consistency eval run

**live side (falkordb-backed)**
- `POST /identity/resolve` — given a channel + inbound handle (phone number, imessage/whatsapp identifier), resolve or create a `Contact` node
- `POST /interactions` — given `voice_profile_id` + `interaction_context` (including a resolved `contact_id`), run the tuning engine, get back an `instruction_set`, and hand it to the appropriate channel adapter
- `GET /interactions/{id}` — fetch interaction record including fallback_triggers, for post-interaction review
- `POST /channels/{channel}/webhook` — inbound event from a channel adapter (an inbound call, sms, imessage, or whatsapp message hitting the number) — the entry point that triggers identity resolution and then `POST /interactions`

---

## deployment notes

- the tuning engine (`agent.md`) must run as a pure function with no model call in it, so it stays deterministic across every channel. model calls (distillation, sandbox generation) stay strictly upstream of it.
- each channel adapter (voice, sms, imessage, whatsapp) is a separate integration surface with its own provider/SDK — none of that is fixed in this spec on purpose, since the choice per channel is an implementation decision that can change without touching the identity graph, the tuning engine, or the research pipeline. the contract each adapter must satisfy is defined in `agent.md`'s integration points section.
- track usage against credit/rate limits for every external research-side service (firecrawl, tavily, browserbase, adaptionlabs.ai, tensormux), since the scrape stage is the most volume-heavy part of the pipeline.

---

## v2: fine-tuning track

everything above tunes *what* is said and *how it's phrased* through prompt-level instruction, not through retraining a model. the fine-tuning track is the path to matching actual cadence/prosody on the voice channel specifically, past what prompt-tuning can reach.

- **tokenfactory (nebius)** — training infra for a fine-tune pass on a target's voice.
- **amd developer cloud** — hardware for that fine-tune run.

not required for the v1 definition of done in `scope.md`. it's the next thing to build once the deterministic prompt-tuning path (this doc + `agent.md`) is proven out across at least two channels, not a parallel track to build simultaneously with the identity layer.
