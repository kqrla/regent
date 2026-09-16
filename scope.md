# scope.md

what yaptele actually has to do, and where the line is. read this before `research.md`, `agent.md`, or `applications.md` — it's what keeps "research" from quietly turning into impersonation, and what keeps "one number, every channel" from quietly turning into "we vacuumed up someone's private messages."

---

## terminology, up front

**medium** = where research signal comes from when building a voice profile (twitter, a slack sandbox, a discord sandbox). see `research.md`.
**channel** = where the live agent actually operates, on its one number (voice, sms, imessage, whatsapp). see `agent.md` and `applications.md`.

a voice profile is *built from* mediums and *deployed across* channels. these are not the same axis, and every doc in this repo should keep them separate.

---

## v1 definition of done

v1 is done when all of these are true at once, not just individually demoable:

1. one target's public speech has been scraped across at least two research mediums (twitter/x plus one sandboxed medium).
2. that speech has been distilled into a lingvist / lexica / lingvica voice profile (schema in `agent.md`).
3. the profile has been sandbox-simulated in at least two mediums and scored against real samples for at least one of them (`benchmark.md`).
4. a deterministic tuning engine turns (voice profile + interaction context) into an instruction set, and running it twice on the same inputs produces the same instruction set.
5. one phone number resolves a contact against the falkordb identity graph and completes a real interaction on at least two channels — voice, plus one of sms/imessage/whatsapp — using the same identity and the same tuned voice.
6. the disclosure line is present and enforced on every channel the number operates on, not just voice.
7. a demo shows the same contact resolving correctly across two channels, with the voice register shifting appropriately between them.

if any of these is faked, mocked, or "would work if" — it's not done.

---

## in scope

- one target voice reconstructed at a time. multi-target is a v2 concern, not v1.
- public, already-published speech only, for building a voice profile — nothing behind a login wall, nothing private.
- twitter/x as the anchor research medium; imessage/slack/discord as **simulated sandboxes** for testing generalization, not scrape targets — see the medium/channel distinction above, and don't confuse these sandboxes with the *live* imessage/whatsapp channels the agent actually operates on.
- one phone number as the identity anchor, resolving contacts via falkordb across at least two live channels end to end for v1.
- code-switching and transliteration as a first-class research signal, not noise filtered out before analysis.
- disclosure on every channel: a contact can always ask whether they're talking to a reconstruction and get a true answer, whether on a call, a text, an imessage, or a whatsapp thread.

## out of scope

- private data used to build a voice profile — dms, private group chats, anything behind a login wall, anything the target didn't publish for public consumption.
- a persona for a research medium the target has no public footprint on. if the medium doesn't exist for that person, the sandbox for it is **absent**, not filled in with a plausible-sounding guess.
- using this to impersonate a real person to deceive a contact on any channel, or to get someone to do or believe something they wouldn't if they knew who or what they were actually talking to. the disclosure requirement in the definition of done exists specifically to block this path, and it applies identically to voice, sms, imessage, and whatsapp — not just to calls.
- reconstructing a private individual's voice without their involvement. public figures with a large public corpus are a very different consent situation than a random private person — if a demo target isn't a public figure, get their actual consent and say so.
- claiming the voice reconstruction is "them," on any channel. it's a reconstruction from public signal. say that, every time.
- faking the filler-plus-cue behavior (`agent.md`) — a typing sound on voice, a typing indicator on text channels — when no real backend lookup is actually in flight. that behavior exists to cover a genuine wait; using it when there's nothing to wait for means performing a fake tell about being an AI doing a lookup, which undercuts the disclosure principle above rather than supporting it.

---

## consent & disclosure (this is the actual hard part)

two separate obligations, don't collapse them into one:

1. **disclosure to whoever's on the other end**, on whichever channel they're on. this is a hard constraint in the tuning engine's output (`agent.md`), not a suggestion, and it doesn't get weaker on text channels just because there's no voice to give it away.
2. **consent from the target being reconstructed**, or a clear public-figure justification for why that consent bar is different. the cleanest path for a demo is a public figure with an enormous public corpus and no reasonable expectation that public speech won't be studied — and saying exactly that, plainly, wherever the project is presented.

if this continues past v1, #2 needs a real answer (opt-in only, target-controlled deletion, a way for a target to see and correct their own profile) before it points at anyone who isn't already a heavily public figure.

---

## the medium/channel distinction that matters

**research mediums** (signal goes in): whatever the target has actually published publicly, used to *build* a voice profile. twitter/x primarily, plus sandboxed imessage/slack/discord simulations used to test generalization — synthetic output, not scraped private messages, per the out-of-scope section above.

**live channels** (the agent operates on them): voice, sms, imessage, whatsapp — all resolving against the same phone number and the same falkordb identity graph. these are where a *contact* interacts with the tuned agent, and they have nothing to do with whether the target's voice profile happened to be built partly from a simulated imessage sandbox. a live imessage channel talking to a real contact, and an imessage sandbox simulating a target's texting style during research, are two unrelated uses of the same platform name — keep them straight.

---

## non-goals (things people will ask for that aren't v1)

- perfect voice cloning of audio (literal TTS timbre matching) — a fine-tuning/audio-model problem, not this pipeline's job in v1.
- multi-target interactions, or blending two people's voices.
- real-time learning during an interaction — the profile is fixed going in; mid-interaction adaptation is a fallback-rule problem (`agent.md`), not a re-training problem.
- covering every possible channel. voice plus two text channels is enough to prove the thesis.

## v2 / stretch scope

- fine-tuning a model on cadence/prosody for the voice channel specifically, so the match goes past word choice into how it actually sounds.
- more mediums for research, more targets, a real opt-in flow for non-public-figure targets.
- a proper human-judgment validation study for sandbox scoring, instead of LLM-judge or self-consistency scoring.
- richer per-channel behavior (e.g. read receipts, media messages on whatsapp/imessage) once the core identity + disclosure layer is solid.

none of this blocks the v1 definition of done above.
