# When the Easy Language Turned Out Not to Be

*Adding Spanish to a Finnish reading app — and what went wrong with the first plan*

*In one sentence: local models aren't reliable enough to generate Spanish morphology on the fly, but taking a page from the Finnish solution led to giving a page back via prompt engineering.*

---

I've been building Terve2, a reading comprehension app that helps learners work through native-language texts. The Finnish version uses a morphological analyzer called Voikko as a deterministic tool layer: Voikko identifies the grammatical form of each word with certainty, and a local LLM explains what that form means in context. The LLM never guesses at grammar; it only contextualizes facts the tool has already established.

When I decided to add Spanish, my working assumption was that this architecture was Finnish-specific — a workaround for an unusually hard language with an unusually large LLM capability gap. Spanish, I thought, would be different. The morphology is simpler, training data coverage is strong, and I'm near-fluent, which means I can evaluate output quality directly. The plan was to let the LLM handle morphological analysis and use a light tool layer — maybe a dictionary API for authoritative lookups — rather than building a full deterministic sidecar.

That assumption turned out to be wrong. This is the story of how I found that out, what I built instead, and what the Spanish work unexpectedly fixed in the Finnish prompt engineering.

---

**What this covers:**
- A 16-case Spanish morphology harness across five models, and why I stopped it early
- The architectural pivot: from LLM-primary to tool-primary, for different reasons than Finnish
- Building the Spanish tool layer: spaCy, an enclitic decomposer, and a dropped dependency
- The prompt engineering finding that transferred back to Finnish
- Four production lessons from the deployment

---

## The Harness

Before building anything, I ran an evaluation harness: 16 Spanish test cases across five groups (verb morphology, ser/estar, command forms, noun/adjective agreement, reflexive and compound forms), scored against a rubric covering accuracy, explanation quality, and ambiguity handling.

Five models ran against the full suite. Scores out of 160:

- Qwen3.6-27B: 154
- Qwen3.5-27B: 128
- Qwen2.5-72B: 103
- Qwen3-8B: 93
- Qwen2.5-32B: 91

Qwen3.6 dominated, and the gap was large enough to feel decisive. But I stopped the harness before completing the fast-format runs — the production format I actually needed, which requires a TRANSLATION / FORM / EXPLANATION response in under ten seconds per click. The slow harness scores weren't the point anymore.

The decision to stop early came from reading the failure modes rather than the totals. Three things were consistent across all models:

**The ser/estar heuristic.** Four of five models explained *estar* as "temporary" and *ser* as "permanent" without being prompted not to. This framing is wrong — it fails on sentences like *está muerto* (death is not temporary) and *es joven* (youth is not permanent) — and it's actively harmful for beginners who will apply it to new sentences. Only Qwen3.6 avoided it unprompted. The others could be corrected with a system prompt instruction, but the fact that it was the default across models was a signal about what I was working with.

**Clitic mechanics.** Cases S11 (*Dímelo*) and S15 (*comprártelo*) — affirmative imperatives and infinitives with fused pronoun clitics — broke every model. Not partially. Completely. The models either ignored the clitics or described them incorrectly. This is a well-defined grammatical structure with deterministic rules. The LLM was generating its best guess at a form that a rule layer could have handled exactly.

**Latency under format pressure.** The slow harness ran with comfortable inter-call delays. Production needs sub-ten-second responses in a reading panel where a user has clicked a word and is waiting. At warm temperatures, Qwen3.6 was running 20–30 seconds per case. That's not usable, and there's no prompt engineering fix for wall-clock time.

The harness was designed to answer "which model should we use?" The data was answering a different question: "should we be asking the LLM to do this at all?" The right call was to stop and pivot rather than finish collecting results that wouldn't inform the actual decision.

---

## The Architectural Pivot

The Finnish architecture wasn't a workaround for Finnish. It was the right pattern for any language where the LLM's failure modes are structural rather than factual.

For Finnish, Voikko exists because Finnish morphology is genuinely hard and LLM coverage is thin. For Spanish, the reasoning runs differently: Spanish morphology is well-defined, the rules are regular, and the LLM failure modes I observed — clitic mechanics, the ser/estar heuristic — were cases where the LLM was generating plausible-sounding output that was structurally wrong. A deterministic layer doesn't fix an LLM that lacks knowledge. It fixes an LLM that is applying the wrong heuristic to a problem that has an exact answer.

The generalized principle that came out of the Finnish work: identify what the LLM cannot do reliably for a given language, build a deterministic tool for exactly that gap, and let the LLM do everything else. For Finnish, the gap is the full morphological analysis — fifteen cases, possessive suffixes, complex verb forms. For Spanish, the gap is narrower but real: clitic decomposition and the structural forms that sit at the edges of what generalist training data covers well.

---

## The Spanish Tool Layer

The Spanish tool layer ended up with two components.

**spaCy with the `es_core_news_lg` model** handles the primary morphological analysis: lemma, part of speech, tense, person, number, mood, and dependency parse. For the cases the harness tested, it handles accent disambiguation correctly (*sé* vs *se*, *él* vs *el*), separates *ser* and *estar* as distinct lemmas, and correctly identifies the grammatical subject in *gustar*-type constructions where Spanish inverts the expected argument structure.

**An enclitic decomposer** handles fused verb+clitic tokens — the cases that broke every model in the slow harness. In Spanish, affirmative imperatives (*Dímelo*), infinitives (*verlo*), and gerunds (*haciéndolo*) fuse the verb with one or more pronoun clitics into a single orthographic word. spaCy returns a space-in-lemma signal for these tokens — the lemma of *Dímelo* comes back as `decir él` — which serves as a reliable detection gate. The decomposer splits the token, re-analyzes each component, and labels each clitic's grammatical role. About thirty lines of Python.

The decomposer completely fixed S11 and S15. Both scored 10/10 with tool support after scoring zero without it.

A third component I had planned to include was mlconjug3, a conjugation lookup library, pinned to scikit-learn==1.6.1 for compatibility. During the Claude Code deployment session, this failed at runtime with an error that no version pin could fix: mlconjug3's bundled model was pickled against sklearn 0.24.2, and the `Log` SGD loss class it depended on had been removed entirely from newer sklearn versions. The fix was to drop mlconjug3. spaCy's `es_core_news_lg` provides tense, person, number, and mood natively; mlconjug3 was redundant for the vetting use case. The lesson worth keeping: when a library ships a pre-trained pickle, you are tied to the exact environment it was trained in, and that environment may be incompatible with your runtime in ways no version pin can repair.

---

## The Vetting Layer

With tool output supplying the morphological facts, the LLM's job narrows to three things: confirm the reading fits the sentence context, select among ambiguous analyses when the tool returns multiple options, and produce a one-sentence explanation a beginner can apply.

Model selection for this role was also empirical. Qwen3.6-27B dense ran at 26 seconds per case — over the latency threshold even with tool support. A MoE variant at 4.7 seconds returned empty responses because the `/no_think` token I used to suppress extended reasoning suppresses all output on that architecture. Mistral Small 3.2 24B ran at 4.3 seconds average, 5.5 seconds maximum, and scored 149/160 on the vetting rubric after one system prompt iteration.

That iteration was the ser/estar fix. The original system prompt said, in effect, "do not use temporary or permanent to explain ser/estar." Mistral ignored the bare prohibition and used the heuristic anyway. The fix was to replace the prohibition with explicit positive framing and worked examples:

- Use *ser* for intrinsic identity, origin, material, or classification — what something *is*. Example: *La puerta es de madera* describes what the door is made of, a defining characteristic.
- Use *estar* for resultant states and conditions — how something has come to be. Example: *La puerta está abierta* describes the door as having been opened and now being in that state.

With that framing, Mistral followed the rule on every case that had previously failed.

The MoE latency finding is worth preserving separately: MoE models with thinking capability don't honor `/no_think` as a suppression token — they consume the full token budget on reasoning and produce no output. Raising `num_predict` to accommodate both thinking and response restores output but blows the latency budget. Dense models without thinking mode are the reliable path to sub-ten-second latency on consumer hardware at current model sizes.

---

## What Spanish Fixed in Finnish

The ser/estar framing finding — that positive framing with worked examples outperforms bare prohibition — applied directly to a problem in the Finnish system prompt that I hadn't fully addressed.

The Finnish LLM is given rules about several grammatical traps: don't explain adessive possession as spatial ("at me"), don't use interior/exterior framing as the primary explanation for locative case choice, don't describe the Finnish passive as agentless. These rules existed in the Finnish MCP system prompt — the interactive assistant version — but as bare prohibitions, not as framed explanations with worked examples.

After the Spanish work, I audited the Finnish prompt against the Spanish v2 structure and rewrote the prohibition sections to follow the same pattern: state what's wrong, state the correct framing, give an example that illustrates the difference. The same session also added an override clause — the LLM is explicitly licensed to correct Voikko's analysis when context makes a different reading clearly correct, with a requirement to state that it is doing so. This was already in the Spanish prompt because the tool layer is newer and less trusted than Voikko; it turned out the Finnish prompt needed it too.

Neither of these changes came from a Finnish-specific review. They came from working on Spanish and noticing that the solution structure generalized.

---

## Production

Deploying the Spanish driver produced four findings that belong in the record.

The mlconjug3 failure is covered above. The remaining three:

**Base image contents are invisible assumptions.** The Spanish sidecar's health check was written by analogy with the Finnish Voikko sidecar, which works because its base image happens to include curl. The `python:3.11-slim` image doesn't. The fix — using Python's own `urllib.request` — is cleaner than installing curl for a health check, but the failure mode only appeared at deployment.

**Partial refactors produce seams; the honest response is to name them.** The Go refactor introduced a `language.Driver` interface and replaced the Voikko client in the main handler path. But the quiz, flashcard, and article-difficulty handlers call Voikko and the Finnish prompt builder directly, and the spec said not to touch them. The resolution was to keep both the driver and the Voikko client in the Handlers struct, with backward-compatible stubs. This is not a clean refactor. It's an accurate representation of where the abstraction boundary currently sits — Finnish-only features haven't been brought into the driver model yet. Naming that boundary clearly in comments is better than pretending it's resolved.

**Volume mounts for templates are a production footgun.** The docker-compose file mounted `./terve/templates` as a volume for local hot-reloading convenience. On the production server, which has only `docker-compose.yml` and `.env`, the app panicked at startup with a template glob error. The fix is to bake templates and static assets into the image and make volume mounts opt-in for development. The gap between "works on the dev machine" and "works in production" is mostly invisible assumptions — none of which show up in `go build` or a local compose run.

---

## Where Things Stand

The app now runs in two languages, selected by a `LANGUAGE` environment variable at startup. Finnish uses Voikko. Spanish uses spaCy and the enclitic decomposer, with Mistral Small 3.2 24B as the vetting model. The `language.Driver` interface means adding a third language is a new driver and sidecar, not a refactor.

The open question for the next language is the same one that drove the Spanish work: where are the LLM's structural failure modes, and how thin does the tool layer need to be to cover them? For Finnish and Spanish the answer was different in specifics but the same in kind. That's probably the thing most worth carrying forward.

---

*Code: [github.com/lehmann314159/Terve2](https://github.com/lehmann314159/Terve2). Architecture current as of June 2026.*
