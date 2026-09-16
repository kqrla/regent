# proposal: regent x resemble

this document explains how regent is designed to work hand in hand with resemble, and which side of the synthetic voice problem each of us is working on. the short version: resemble protects the sound of a voice. yaptel and regent work on everything in a voice beyond the sound. a synthetic identity needs both halves covered, and neither half covers the other.

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

- **yaptele** measures a voice across three independent layers: lingvist (which languages, and code-switching behavior), lexica (word choice, slang, fillers, hedges), lingvica (register, tone, punctuation-as-tone, per-medium shifts)
- **regent** renders that measured profile as one reachable identity: a phone number across sms, whatsapp, calls, and voicemail, with a single shared memory across all channels
- runtime disclosure everywhere: a reconstruction always discloses it is a reconstruction, on every channel, no matter who initiated contact

these sides do not overlap. resemble certifies the vocal chords. yaptel certifies what is beyond them. there is no competition for scope here because the layers are physically different signals.

---

## where the two sides meet: regent spans both

regent is the proof that both halves matter, because it is the one place both render at once:

- a regent **call** renders acoustically. this is resemble's home ground, and an obvious integration point: regent's rendered call audio can carry resemble's watermark, so anyone running detection can verify a regent voice is a disclosed reconstruction
- a regent **text or whatsapp message** renders stylometrically. there is no audio at all. acoustic detection has nothing to inspect, and that is the point: an identity that is honest and disclosed still needs an integrity story in this layer

so a regent deployment with resemble integration covers the full surface: watermarked audio on calls, stylometric provenance on text.

---

## what we bring to the table

the stylometric layer currently has no equivalent of an audio watermark. nothing on the market can answer: was this text rendered from a stylometric reconstruction, from which profile, built on what corpus, under what consent basis. the primitives yaptel is designing for this:

1. **profile provenance** - every rendered message carries a signed reference to the exact voice profile that produced it: content-addressed by the measured layers, with the public-only corpus sources and consent basis attached, so provenance is verifiable by a third party rather than claimed by the operator
2. **deliberate stylometric markers** - the text-layer analog of a watermark: low-salience, profile-unique behavioral quirks a detector can verify. an open research question we are honest about: robustness against paraphrase stripping, without degrading the fidelity the reconstruction exists to provide
3. **absent-is-correct as a signal** - a profile that provably could not have produced a given text (wrong register for a medium the corpus never covered) is a negative signal a detection scheme can use

none of this exists anywhere yet. that is the joint opportunity.

---

## what a collaboration could look like

this section is deliberately a sketch, not a commitment. candidate shapes, roughly in order of how much shared work each involves:

- **integration only**: regent's call audio carries resemble watermarks. regent becomes a clean, disclosed demonstration corpus for acoustic detection: known-synthetic audio that detection *should* flag, by design
- **joint coverage argument**: a shared framing document or reference architecture for what full synthetic-identity integrity requires: acoustic watermarking plus stylometric provenance, since a synthetic identity can walk past one layer simply by rendering in the other
- **shared research on the text-layer problem**: the stylometric detection gap is an open research problem, and resemble's depth in watermarking and detection methodology is directly adjacent to it. marker design, provenance formats, and evaluation methodology are all places where acoustic-layer experience transfers even though the signal is different

## what each side keeps

- yaptel keeps: the measurement layers, the voice profile format, the provenance and disclosure architecture, regent's delivery and memory stack
- resemble keeps: everything acoustic. yaptel never touches the audio layer, in either direction: we do not process audio, and we do not compete on it
- neither side builds the other's layer. the whole point is that the problem is bigger than either half

---

## tldr

resemble proves which vocal chords made a sound. yaptel proves which behavior made a text. regent is where the two meet, and a synthetic identity that is honest about being one should be verifiable in both layers at once.
