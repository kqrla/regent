# proposal: regent x resemble

this document explains how regent is designed to work hand in hand with resemble, and which side of the synthetic voice problem each of us is working on. the short version: resemble protects the sound of a voice. yaptel and regent work on everything in a voice beyond the sound. a synthetic identity needs both halves covered, and neither half covers the other.

everything below references the specs in this repo directly: `scope.md` for the medium/channel distinction and consent rules, `agent.md` for the voice profile schema and tuning engine contract, `benchmark.md` for the parameter registry and status ladder.

---

## the thesis: a voice is not located in the vocal chords

you can scan and clone someone's vocal tract perfectly and still not have cloned their voice. the part of a voice that makes it *theirs* is also distributed across how they write and speak behaviorally: which words they reach for, how they code-switch between languages, how their formality shifts by medium and audience, how punctuation reads as tone, which fillers they use and where. none of that lives in the vocal chords. it lives in behavior.

this is the gap the yaptel ecosystem exists to measure and render. and it is also why the current generation of voice protection is structurally incomplete on its own: a reconstruction that lives in text channels is acoustically empty. there is no waveform to watermark, no playback to detect. existing voice integrity tooling has nothing to attach to, by design rather than by oversight.

---

## the two sides

### resemble's side: the acoustic layer

- watermarking and detection for cloned voice audio
- certifies the *sound* of a voice: timbre, vocal tract characteristics, playback integrity
- proven products and standards here (perth watermarking, detection models) that already define the acoustic integrity market

### yaptel's side: the stylometric layer

- **yaptele** measures a voice across three layers with real, per-parameter metrics: lingvist (languages and code-switching behavior), lexica (word choice, slang, fillers, hedges), lingvica (register, tone, punctuation-as-tone, per-medium shifts). see `research.md` and `benchmark.md`
- **regent** renders that measured profile as one reachable identity: one phone number across voice, sms, imessage, and whatsapp, resolving contacts against a shared identity graph with shared memory. see `regent.md` and `applications.md`
- disclosure on every channel, per `scope.md`'s definition of done: a contact can always ask whether they're talking to a reconstruction and get a true answer

the medium/channel axis from `scope.md` keeps these sides from even appearing to overlap: research signal comes from mediums (twitter, sandboxed slack/discord), the agent operates on channels (voice, sms, imessage, whatsapp). resemble certifies the acoustic rendering on the voice channel. yaptel certifies the stylometric rendering everywhere, including the text channels where no acoustic signal exists at all.

---

## where the two sides meet: regent spans both

regent is the proof that both halves matter, because it is the one place both render at once:

- a regent **call** renders acoustically. this is resemble's home ground, and the obvious integration point: regent's rendered call audio can carry resemble's watermark, so anyone running detection can verify a regent voice is a disclosed reconstruction
- a regent **text message** renders stylometrically. there is no audio at all. acoustic detection has nothing to inspect, and that is the point: an identity that is honest and disclosed still needs an integrity story in this layer

so a regent deployment with resemble integration covers the full surface: watermarked audio on calls, stylometric provenance on text.

---

## what we bring: provenance from the actual spec

the stylometric layer currently has no equivalent of an audio watermark. nothing on the market can answer: was this text rendered from a stylometric reconstruction, from which profile, built on what corpus, under what consent basis. the primitives below are not aspirational design; they follow directly from schemas already written in this repo.

### profile provenance

`agent.md`'s voice profile is the natural provenance artifact. it already carries everything a third party would need to verify a reconstruction's provenance:

- `target_id` and `generated_at`: which profile, when
- `coverage`: `mediums_with_real_data`, `mediums_sandbox_only`, and `mediums_absent` state exactly what the profile was built from, and what it provably was not. `scope.md` already forbids filling absent mediums with plausible guesses
- `scoring.method` (`real_sample_comparison | llm_judge`), `scoring.confidence_by_medium`, and `scoring.parameter_scores` keyed by `benchmark.md`'s parameter registry: how much the profile was validated, per parameter, not as a single aggregate
- the consent basis from `scope.md`: public figure, public-domain character, or self, all built from public, already-published speech only

the missing piece is a signature: rendered messages carrying a verifiable reference to the exact profile version that produced them, so provenance is checkable by a third party rather than claimed by the operator. the schema is ready for it; the signing layer is the work.

### measurement discipline

`benchmark.md`'s status ladder (candidate, measurable, validated, controllable) and per-parameter registry are exactly the rigor a detection standard needs: "sounds like them" decomposed into independently movable numbers, each labeled with how validated it actually is. no aggregate similarity score anywhere, which is the same honesty resemble brings to audio.

### absent-is-correct as a signal

`coverage.mediums_absent` is already a machine-checkable negative signal: a text whose register mismatches every medium the target's corpus covers is evidence the text did not come from the disclosed profile. a detection scheme can use this today, straight from the schema.

### deliberate stylometric markers

the text-layer analog of a watermark: low-salience, profile-unique behavioral quirks a reconstruction emits and a detector can verify. per `benchmark.md`'s ladder these sit at candidate status: no formula yet, and an honest open problem on robustness against paraphrase stripping without degrading the fidelity the reconstruction exists to provide. this is where shared research with acoustic watermarking experience would transfer best, even though the signal is different.

---

## what a collaboration could look like

a sketch, not a commitment. roughly in order of shared work:

- **integration only**: regent's call audio carries resemble watermarks. regent becomes a clean, disclosed demonstration corpus for acoustic detection: known-synthetic audio that detection *should* flag, by design
- **joint coverage argument**: a shared framing document for what full synthetic-identity integrity requires: acoustic watermarking plus stylometric provenance, since a synthetic identity can walk past one layer simply by rendering in the other
- **shared research on the text-layer gap**: marker design, provenance signing formats, and evaluation methodology, where acoustic-layer experience in watermarking and detection is directly adjacent

## what each side keeps

- yaptel keeps: the measurement layers, the voice profile schema, the benchmark and provenance architecture, regent's delivery and memory stack
- resemble keeps: everything acoustic. yaptel never touches the audio layer, in either direction: we do not process audio, and we do not compete on it
- neither side builds the other's layer. the whole point is that the problem is bigger than either half

---

## the one sentence version

resemble proves which vocal chords made a sound. yaptel proves which behavior made a text. regent is where the two meet, and a synthetic identity that is honest about being one should be verifiable in both layers at once.
