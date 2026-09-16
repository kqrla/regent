# yaptele

> yap (yapping) + tele (telephony) — one phone number, cross-referenced against a contact identity graph, running voice, sms, imessage, and whatsapp as one contact suite for an ai agent — not just a way to place calls.

---

## what is this

most agent-calling tools give an agent a way to dial a number. that's one channel, and it still talks the same way to everyone on it.

yaptele is trying to build the layer underneath that: an agent gets **one phone number**, and that number is a real identity — every contact who calls it, texts it, imessages it, or whatsapps it resolves against the same identity graph, gets the same continuity of memory, and hears or reads the right version of the agent's voice for who they are and which channel they're on.

that "right version" part is the actual research problem. a real person doesn't have one voice — texting-you, slack-you, twitter-you, and phone-you are all different, shaped by medium, audience, age, gender, generation, and things like code-switching between languages mid-sentence. yaptele scrapes a target's public speech, measures that variation as three separate layers instead of collapsing it into one persona, and uses the result to tune how the agent shows up per contact, per channel — not a generic average voice reused everywhere.

so, two things stapled together:

1. a research pipeline that measures how a voice actually shifts across mediums, with real metrics
2. an identity + channel layer — one number, one contact graph, four live channels — that puts that measured voice to work

---

## the idea: three layers, not one blob

| layer | what it measures | example signal |
|---|---|---|
| **lingvist** | language substrate — which language(s), and how someone moves between them | code-switch rate, transliteration habits, loanword defaults |
| **lexica** | the actual words | slang inventory, generational markers, fillers, hedges |
| **lingvica** | delivery — tone and register, for a given medium and audience | formality, warmth, punctuation-as-tone, emoji density |

kept separate on purpose, all the way through the pipeline — a bad interaction should be debuggable down to *which layer* misfired. `benchmark.md` tests whether that separation is actually worth it against a baseline that collapses all three into one vector.

**terminology note, because it gets confusing fast:** "medium" and "channel" mean two different things in this project. a **medium** is where research signal comes from when building a voice profile — twitter, a slack sandbox, a discord sandbox (see `research.md`). a **channel** is where the live agent actually operates on its one number — voice, sms, imessage, whatsapp (see `agent.md`, `applications.md`). a voice profile is built from mediums; it gets *deployed* across channels. don't conflate the two when reading the schemas.

---

## the identity layer: one number, one graph

the phone number is the anchor. every inbound or outbound touch on any channel resolves against the same falkordb-backed graph: who this contact is, what channel they're using, what's been said to them before (on any channel, not just this one), and which voice register applies to them specifically.

that's the actual shift from "an agent that can make calls" to "an agent with a phone number" — the number carries identity and continuity across voice, sms, imessage, and whatsapp, instead of each channel being a disconnected integration with its own memory.

full detail on the graph shape, the channel router, and how a contact resolves to a voice profile + register is in `applications.md` and `agent.md`.

---

## scope constraint

**in scope for v1:**
- one number as the identity anchor, resolving contacts across at least voice plus one text channel end to end
- one target voice profile at a time, reconstructed from public speech
- disclosure on every channel — a contact can always ask whether this is a reconstruction and get a true answer, whether they're on a call or in a text thread

**out of scope, on purpose:**
- private data, or anything behind a login wall, used to build a voice profile
- inventing a persona for a research medium the target has no public footprint on
- using this to deceive a contact about who or what they're actually talking to, on any channel

full detail in `scope.md`.

---

## the pipeline

```
scrape (research mediums: twitter, slack/discord sandboxes)
  ↓
distill into lingvist / lexica / lingvica signal
  ↓
sandbox-simulate per research medium
  ↓
score against real samples or LLM-judge (voicebench, see benchmark.md)
  ↓
deterministic tuning engine (voice profile + interaction context → instruction set)
  ↓
channel router — resolves a contact + channel against the falkordb identity graph
  ↓
live interaction: voice, sms, imessage, or whatsapp
```

mem0 + falkordb aren't just call memory anymore — falkordb is the identity graph itself (contacts, numbers, channels, prior interactions across all of them), and mem0 is the working-memory layer on top of it for a given conversation. whenever a real backend lookup is in flight — memory, identity resolution, or anything else — the agent fills the wait with a filler word pulled from the target's own voice profile, plus a channel-appropriate cue (a faint typing sound on voice, a native typing indicator on text channels), instead of going quiet or breaking character. it only does this for a genuine wait — see the boundary note in `scope.md` before wiring in a new backend call.

---

## how "sounds like them" gets measured

not with one similarity score. "voice match: 0.81" is as meaningless as any single collapsed style number — a voice is code-switch rate, register, filler density, tone, and each one gets measured independently, with a **delta vector** showing how far the targeted parameter moved versus how much everything else accidentally moved too. full parameter registry, status ladder, and baseline suite (**voicebench**) in `benchmark.md`.

---

## where this breaks (honest)

- **channel-shift correction is designed, not built.** a profile built mostly from written mediums will over-index on written-only tics unless something explicitly corrects for the *target channel* — voice needs those tics converted to prosody, but a text channel can often carry them more directly, which means the correction isn't the same problem twice, it's a different problem per channel.
- **identity resolution across channels is unproven.** the same contact showing up on voice one day and whatsapp the next needs to resolve to the same graph node reliably — that matching logic doesn't exist as a tested thing yet.
- **prosody metrics have no ground truth to check against**, since there's no recorded voice-channel audio from the target to compare a reconstruction to. scored for internal consistency instead of accuracy — a real limitation, not an oversight.
- **the transliteration/code-switch eval doesn't exist as a number yet**, even though it's the most defensible research claim available here.
- **the reconstruction-vs-deception line is a design decision, not a solved problem**, and now has to hold across four channels instead of one.

---

## reference points (not affiliations)

nothing here is part of a program or umbrella product. these are just things it's borrowing shape from:

| reference | why it's here |
|---|---|
| sociolinguistics (code-switching, idiolect, register, diglossia) | the actual vocabulary for what's being measured |
| classic stylometry, and per-parameter benchmarking practice | "measure how it's made, not what it says," and the discipline of scoring per-factor instead of one aggregate number |

---

## repo layout / spec docs

this readme is the pitch. the actual build spec is split across:

```
scope.md         → v1 definition of done, consent/disclosure rules, non-goals
research.md      → the three-layer model in detail, the scrape/distill/sandbox pipeline
benchmark.md     → the parameter registry, delta vectors, voicebench baselines
agent.md         → voice profile schema, the tuning engine contract, per-channel behavior
applications.md → architecture, identity graph shape, api surface, deployment
glossary.md      → linguistics vocabulary grounding every field above
```

if this readme and one of those docs disagree, the doc wins — this is the map, not the spec.
---

## repository layout

this repo is the working home of the whole ecosystem, both sides in one place:

- `scope.md`: what is in and out, the medium/channel distinction, and the v1 definition of done. read this first
- `research.md`: the yaptele measurement side, how a voice gets measured across the three layers
- `benchmark.md`: voicebench, the parameter registry, and the status ladder for turning "sounds like them" into numbers
- `agent.md`: the tuning engine, the voice profile schema, and per-channel rendering for the live agent
- `applications.md`: the system underneath it all, architecture, tool responsibility, identity graph, and deployment
- `glossary.md`: grounding vocabulary from linguistics, mapped to the three layers
- `regent.md`: the delivery and memory architecture for the one-number, every-channel live side
- `proposal/`: external collaboration proposals, currently the resemble.ai pitch (`proposal/resemble.md`)
