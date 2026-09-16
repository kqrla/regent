# glossary.md

grounding vocabulary, pulled from the linguistics/speech notebook this project started from. organized by which layer (`lingvist` / `lexica` / `lingvica`) each term maps to, since the point of this doc is keeping the three-layer model in `research.md` precise instead of a set of made-up buckets. use this when writing distillation prompts, sandbox scenarios, or eval labels — reach for the real term instead of reinventing one.

---

## lingvist — language substrate

| term | definition |
|---|---|
| code-switching | shifting between languages or styles mid-conversation |
| borrowing | adopting words or structures from another language |
| loanwords | words borrowed from other languages (e.g. "café") |
| bilingualism | fluency in two languages and its cognitive effects |
| diglossia | two language varieties used in different social contexts |
| convergence | languages becoming more similar through contact |
| pidgins / creoles | simplified contact languages, and pidgins that become full native languages |
| romanization | representing non-Latin scripts in Latin letters |
| transliteration | mapping characters between writing systems |
| orthography | the conventional spelling system of a language |

## lexica — vocabulary

| term | definition |
|---|---|
| slang | informal words that mark in-group belonging |
| jargon | specialized vocabulary of a profession or field |
| sociolects | language varieties tied to social class |
| ethnolects | speech patterns of ethnic communities |
| idiolects | an individual's unique way of speaking — the closest existing term to "a target's voice profile" |
| idioms | phrases whose meaning can't be guessed from words alone |
| euphemisms | softer substitutes for harsh or taboo terms |
| hedging | softening statements ("sort of", "i think", "maybe") |
| fillers | sounds or words that hold the floor ("like", "you know") |
| loanwords | (see lingvist — vocabulary-level borrowing shows up in both layers depending on whether you're measuring the word or the switch) |

## lingvica — delivery / register

| term | definition |
|---|---|
| registers | formality levels — casual, consultative, frozen |
| language attitudes | social judgments about "correct" or "prestige" speech |
| prosody | the music of speech — rhythm, stress, and intonation combined |
| cadence | the rise-and-fall pattern at the end of phrases |
| tempo / rhythm / flow | speed, stress pattern, and connectedness of speech |
| pitch contour / intonation | shape of pitch movement across an utterance; rising/falling patterns that convey meaning |
| timbre / resonance | the distinctive color of a voice, and the fullness produced by vocal cavities |
| vocal fry / breathiness / nasality | specific voice-quality textures |
| affect | the emotional tone conveyed through voice |
| warmth / sarcasm / sincerity / playfulness | delivery qualities directly mapped in the `lingvica.tone_defaults` schema in `agent.md` |
| uptalk | rising intonation at the end of statements (sounds like a question?) |
| monotone / sing-song / staccato / drawl / lilt | speech mannerism categories — useful vocabulary for describing a target's `pacing` field |
| dramatic pauses / trailing off | intentional vs. unintentional silence, relevant to the medium-shift correction table in `research.md` (a written "..." maps to one of these, not the other) |
| turn-taking / interruptions | conversational structure — relevant if a call context involves back-and-forth rather than monologue |

---

## why this doc exists instead of just winging terminology

"sounds genz" and "sounds sarcastic" are not measurable fields. "uses uptalk," "hedges frequently," and "code-switches on emphasis, not on quoting" are. every field in the schemas in `agent.md` and `research.md` should be traceable to a term on this page — if a field can't be pinned to real linguistic vocabulary, that's a sign it's a vibe pretending to be a measurement, and it should either get sharpened or get cut.
