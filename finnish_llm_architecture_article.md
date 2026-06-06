# Building a Finnish Language Learning App with a Deterministic Core

*In one sentence: I found that open models struggled with Finnish, but that they partnered well with a deterministic engine that got around the proximity problem.*

Finnish has fifteen grammatical cases. A single verb form can carry tense, person, number, negation, and mood simultaneously. The same noun root produces dozens of valid surface forms depending on what the sentence is doing — forms built by stacking suffixes that interact with vowel harmony, consonant gradation, and irregular stems. These aren't edge cases; they appear throughout ordinary Finnish text.

I built a reading comprehension app for Finnish learners. The core design question turned out not to be which LLM to use, but where not to use an LLM at all.

**This article covers:**
- Why Finnish is structurally adversarial for LLMs — tokenization, semantic proximity, and coverage
- What the Finnish-specialized model evaluation found, and why the results aren't a verdict on those models
- Why frontier APIs aren't the answer for this use case
- What the local model benchmark shows — and why the bar is lower once Voikko is in the picture
- Voikko — the deterministic morphological analyzer that owns the fact layer
- How the two-service architecture (Go + Python/FastAPI sidecar + Ollama) divides responsibility
- A concrete walkthrough of a word-click interaction end-to-end
- CEFR difficulty scoring as a second example of the deterministic layer in action
- The phrase expansion affordance — a UX detail that only works because the deterministic layer is reliable
- The reading corpus: 20 Finnish books from Project Gutenberg, embedded in the binary and organized by CEFR level
- Why htmx was the right frontend choice
- How the architecture changes for Spanish — and what that reveals about the general principle

---

## The Problem with LLM-Only Morphological Analysis

Morphological analysis is the process of breaking a word down into its grammatical components — identifying the base form, the part of speech, and every grammatical feature the word carries: case, tense, number, mood, and so on. For a language learning app, it's the foundation everything else is built on. Get it wrong and the flashcards teach bad rules, the difficulty scoring misfires, and the lesson targeting breaks down.

Finnish cases work like prepositions in English, but instead of separate words before the noun — "to the house," "in the house," "from the house" — they're suffixes attached directly to it: *taloon*, *talossa*, *talosta*. The meaning is equivalent; the encoding is completely different. Finnish has fifteen of these cases. German has four. English has effectively none in nouns. In practice, everyday Finnish speech uses a modest subset of the fifteen regularly, but real texts — especially literary ones — use all of them, and they interact with possessive suffixes, vowel harmony, and consonant gradation in ways that multiply the space of valid word forms considerably.

Finnish is also agglutinative, meaning it builds words by stacking suffixes rather than using separate function words. The word *taloissaan* is a single token carrying six distinct grammatical facts: the lemma (the base form, *talo* — house), part of speech (noun), case (inessive — location, "in"), number (plural), possessive suffix (3rd person singular — "his/her/its"), giving the surface meaning "in his/her/its houses." In English that's five words. In Finnish it's one, and that one word is built by a set of interacting rules.

The straightforward approach to building the app: the user selects a word, phrase, or passage, it goes to the model, the model returns the grammatical analysis and an explanation. This works for the common forms that appear frequently in everyday text — basic nominative nouns, simple present-tense verbs. These forms are short, appear often, and LLMs handle them reliably.

The failure mode appears with forms that are grammatically regular but individually rare — a possessive suffix on a plural inessive of an irregular stem, or a verb combining passive, perfect, and conditional into one word. These constructions are correct Finnish and they appear in real texts, but no single form appears often enough in training data for LLM analysis to be consistent. The model sometimes produces the right analysis, sometimes a plausible-sounding but wrong one.

For a language learning app this is a specific kind of problem. A wrong case identification doesn't just display incorrectly — it flows downstream into flashcards, lesson targeting, and the user's developing mental model of the language. Finnish morphology is systematic: once a learner internalizes a pattern for one noun, they apply it to hundreds of similar ones. A wrong rule, reinforced by repeated incorrect flashcards, corrupts that generalization. Inconsistent accuracy is functionally the same as unreliable accuracy when correctness is a hard requirement.

---

## Why Finnish Is Structurally Hard for LLMs

The inconsistency has a structural cause. Three factors interact and compound each other.

**Tokenization.** Most modern LLMs use BPE (byte-pair encoding) tokenizers trained on corpora dominated by English and other high-resource languages. A BPE tokenizer learns to split text into subword units that reflect the statistical patterns of its training data. For English, this works well — common words stay intact, rare words split at meaningful boundaries. For Finnish, it breaks down. A word like *taloissaan* carries six morphological facts in a single token, but a tokenizer trained on English-heavy data fragments it into subword pieces that don't correspond to meaningful linguistic units. The model then has to reconstruct grammatical meaning from fragments — a harder task than reading coherent forms, and one it was never explicitly trained to do.

This is the key contrast with English prepositions. "In his houses" is three tokens in English, each independently meaningful. *Taloissaan* is one word in Finnish, fragmented into arbitrary pieces by a tokenizer that doesn't know what it means.

**Semantic proximity.** The same lemma appears across many distinct surface forms in Finnish. *Talo* alone generates over a dozen case forms before possessive suffixes multiply the space further. In embedding space, these forms often don't cluster the way inflected forms do in morphologically simpler languages. Retrieval and in-context similarity methods that work well for English degrade for Finnish because the surface variation is too high relative to the underlying semantic content.

**Coverage.** Finnish has approximately five million native speakers. It is not a low-resource language in an absolute sense, but it is significantly underrepresented in LLM training data relative to English, Spanish, French, or German. The morphological system is large and the training signal is thin — particularly for complex forms, which are individually rare even in a large Finnish corpus. Tokenization fragmentation makes it harder to learn morphological patterns from the Finnish text that does exist; low coverage means there isn't enough signal to compensate.

German is a useful comparison. German has four cases, a large and well-represented presence in LLM training data, and enough morphological regularity that general-purpose models handle German grammar reliably. Finnish has fifteen cases, thin training coverage, and an agglutinative structure that fights the tokenizer. The structural difficulty isn't just a matter of case count — it's the combination of all three factors.

---

## The Model Evaluation

Before designing around the problem, the obvious question was whether a model specifically trained on Finnish might solve it. Finnish-specialized and multilingual open models exist. The hardware has 128GB unified memory — large models fit. The question was worth a structured answer.

Three models were evaluated against a rubric designed for the app's actual use case: morphological analysis of real Finnish text, explanation quality in English, handling of the specific constructions that appear in the reading material. Pass threshold: 52/75. All three failed.

**Qwen2.5 72B** (generalist) — 50/75. Came closest, but failed by inventing a correction to correctly-formed Finnish.

**EuroLLM-9B** (EU multilingual specialist) — 30/75. Factual errors, not just morphological ones.

**Poro 34B** (Finnish-first academic model) — categorical fail. Responded in Finnish rather than English.

Pass threshold: 52/75. All three failed.

EuroLLM-9B is an EU-funded model trained on 4 trillion tokens across all EU languages, instruction-tuned, with explicit Finnish coverage. Poro 34B is a Finnish-first model developed by LumiOpen, TurkuNLP, and Silo AI — an academic collaboration specifically targeting Finnish language capability. Both exist because there is a genuine gap in Finnish LLM coverage, and both represent serious work toward closing it.

Qwen2.5 72B came closest — 50/75, two points under the threshold — and its failure was instructive: it invented a third correction on a test sentence where only two were warranted, changing correctly-formed *kaupassa* (inessive: "in the shop") to *kaupasta* (elative: "from the shop"). A model that confidently corrects correct Finnish is a liability in a language learning app.

A note on these results: this was a narrow evaluation conducted by someone still learning Finnish, against a rubric designed for one specific use case. EuroLLM and Poro are valuable open projects, and their scores here say nothing about their performance on other Finnish tasks. I'm grateful these models exist — without them, this question would have had no candidates worth testing. The evaluation was designed to answer one question: can any available model handle morphological analysis reliably enough to skip a deterministic layer? The answer was no.

That answer reframed the problem. The question wasn't "which model handles Finnish best" — it was "Finnish morphology is actually quite regular; is there an existing deterministic engine that handles the structural facts, leaving the LLM to explain rather than analyze?" That line of thinking led to Voikko.

---

## Why Not a Frontier API

The remaining option is a frontier API — GPT-4o, Claude, Gemini. These models handle Finnish morphology substantially better than any local open model. They're not the answer for this app for two reasons.

The first is expense. A reading app that calls an external API on every word selection — and a serious learner might make dozens of selections per session — accumulates cost at a rate that makes the tool impractical for sustained daily use. The economics don't work.

The second is ownership. The app is designed to be a personal tool running on personal hardware. The reading history it accumulates — what texts the user is working through, what words they find difficult, what constructions they're learning — is a detailed record of someone's intellectual life. That data doesn't need to leave the local network, and keeping it local isn't just a privacy preference; it's part of what makes the tool feel like yours rather than a service you're renting.

The capability gap between frontier and local models is real. The architecture closes it for the specific task where it matters, without needing frontier access.

---

## Local Model Benchmark

With the architecture settled — Voikko handles morphological analysis, the LLM explains Voikko's output — the remaining question was which local model to use as judge. The role is narrower than full analysis: the model receives Voikko's output as established fact and explains it in plain English. Explanation quality and latency are what matter; morphological accuracy from the model's own knowledge is irrelevant because the model is never asked for it.

The app includes a benchmark harness for this — a standalone Go binary baked into the Docker image, runnable via `docker exec terve ./benchmark -models model1,model2`. It runs eight Finnish test cases of graduated complexity against each candidate model, measures latency using Ollama's internal `eval_count` / `eval_duration` fields rather than wall-clock time, and outputs a multi-model comparison table with per-test tokens/second. Full structured responses are printed at the end for manual review.

Results on current hardware (ASUS GX10, GB10 Grace Blackwell, 128GB unified memory):

**Qwen2.5 32B Q4** — ~8,300ms average latency, ~9.8 tok/s. Production model.

**Qwen2.5 32B Q8** — ~13,200ms average latency, ~6.1 tok/s. 37% slower, 9x worse cold-start, output identical to Q4.

**Qwen3 32B** — ~42,000ms average latency. Best quality ceiling, unusable latency for interactive use.

**Llama4 Scout** — crashed Ollama. 67GB exceeded available memory.

The Q4 vs Q8 comparison is the clearest result: across all eight test cases, translations and morphological breakdowns were word-for-word identical between quantization levels. At 32B parameters, linguistic knowledge is largely quantization-invariant. The Q8 penalty — 37% latency increase, 9x worse cold-start, higher memory pressure — produced no measurable quality gain.

Current production model: Qwen2.5 32B Q4. The harness stays in the image so model selection can be re-evaluated as new models become available.

---

## Voikko

Voikko is a Finnish morphological analyzer — open source, built specifically for Finnish linguistics, FST-based (finite-state transducer). It applies the formal rules of Finnish morphology directly, without inference or approximation. Given any Finnish word form, it returns the lemma, part of speech, and all grammatical features: case, number, person, tense, mood, possessive suffixes, comparative markers. It either returns a correct analysis or returns nothing. It does not guess.

The division of labor follows directly: Voikko owns the morphological facts, the LLM explains them. The LLM receives Voikko's output as ground truth and is asked only to reason about it — why this form is used here, what the learner should notice, how it relates to other constructions they've seen. It is never asked to perform morphological analysis from its own knowledge.

---

## Architecture

The app runs as four cooperating Docker Compose services:

- **Go backend** — coordinates everything: routing, template rendering, OAuth, and calls to the other services
- **Voikko sidecar** — a persistent Python/FastAPI service wrapping Voikko for morphological analysis
- **Ollama** — serves Qwen2.5 32B for explanation and generation
- **SQLite** — user model, flashcards, and progress

The browser talks only to the Go backend via htmx; the backend calls the other services as needed.

**Go backend** handles routing, template rendering, OAuth (Google and GitHub), and coordination between services. Most interactions are server-driven — user clicks, server returns an HTML fragment, htmx swaps it into the DOM.

**htmx** was a deliberate choice. The app's interactions are almost entirely server-driven request-response cycles: click a word, get an analysis panel; add a flashcard, get a button state update. htmx handles this pattern cleanly without a build pipeline, without npm, and without a frontend framework to maintain. One vanilla JS module (~50 lines) handles multi-word phrase selection via the browser's Selection API, firing an htmx request on mouseup. Everything else is Go templates and htmx attributes. One language throughout the stack; the frontend is a thin rendering layer over server logic.

**Voikko sidecar** is a persistent Python/FastAPI service wrapping the Voikko library. Persistent matters here: spawning a new Python process per request costs 250–350ms on this hardware (Python startup plus FST load), which is noticeable on every word selection in an interactive reading flow. Running it as a persistent service inside the Docker network brings per-analysis latency to single-digit milliseconds.

**Ollama** runs Qwen2.5 32B on local hardware. With the deterministic layer handling all morphological facts, the model's role is narrowed to contextual explanation and generation — tasks where its output quality at Q4 quantization is sufficient and its latency is acceptable.

---

## The Handoff in Practice

A concrete example. The user clicks *taloissaan* in a reading text.

1. The Go backend calls the Voikko sidecar with the word.
2. Voikko returns deterministically: lemma *talo* (house), noun, inessive case (plural), 3rd person singular possessive suffix — "in his/her/its houses."
3. The backend constructs a prompt with this analysis as given fact: *"The word 'taloissaan' has been analyzed as the plural inessive of 'talo' (house) with a 3rd-person singular possessive suffix, meaning 'in his/her/its houses'. The sentence context is: [sentence]. Explain why this form is used here and what a learner should notice."*
4. Qwen2.5 returns an explanation — the relationship between plural inessive and possessive suffixes, why Finnish encodes possession this way rather than using a separate pronoun, what other nouns follow the same pattern.

The analysis panel shows the Voikko data (always correct) and the LLM explanation (grounded in the verified analysis). The LLM cannot produce a wrong case identification here because it wasn't asked to identify the case — it was told what the case is. Confabulation on the morphological facts is structurally blocked.

The same flow applies whether the user selects a single word, a phrase, or a full paragraph. Voikko analyzes each token; the LLM explains the whole selection in context.

---

## CEFR Difficulty Scoring: The Deterministic Layer Again

The same principle shows up in text difficulty rating. The app sources reading material from YLE Selkosuomi, YLE regular news, and Project Gutenberg, and needs to assign each text a CEFR level (A1 through C2) to match content to the user's current ability.

The scoring pipeline runs entirely on Voikko output, with no LLM involved:

- Proportion of non-nominative case forms in the text
- Number of distinct grammatical cases present
- Incidence of possessive suffixes
- Presence of complex verb forms

A text heavy in nominative-only forms with simple present-tense verbs scores A1. A text with consistent use of rare cases, possessive suffixes, and compound verb forms scores C1 or higher. The signal is structural and computable — a direct function of the same morphological facts Voikko produces for word-click analysis. The LLM isn't needed and isn't used.

This is the deterministic layer doing real work in a second part of the system, not just compensating for one specific LLM failure.

---

## Phrase Expansion

When a user selects part of a noun phrase — say, just the noun without its modifying adjective — Voikko detects that the selection boundary cuts across a grammatical unit. The app highlights the full phrase and shows a subtle "Expand?" affordance. The user keeps control; the hint is available.

This only works because the morphological analysis is reliable. A hint system built on LLM analysis would carry the same inconsistency problem as the analysis itself — expansion suggestions that are sometimes wrong are worse than no hint at all. The feature exists because Voikko's output can be trusted to drive UI behavior.

---

## The Reading Corpus

The app ships with 20 Finnish books from Project Gutenberg, embedded directly in the Go binary via `//go:embed` and organized by CEFR level:

**A1** — Topelius *Lukemisia lapsille* (7 volumes)
**A2** — Härkönen folk tales, Grimm fairy tales (Finnish), Andersen fairy tales (Finnish)
**B1** — *Liisan seikkailut ihmemaassa* (Alice in Wonderland), *Pekka Poikanen* (Peter Pan)
**B2** — Larin-Kyösti, Rauanheimo, Juhani Aho *Yksin*
**C1** — Minna Canth, Runeberg *Vänrikki Stoolin tarinat*, Aleksis Kivi *Seitsemän veljestä*

Embedding in the binary keeps deployment simple — one image, no external corpus dependency, works on first launch. CEFR ratings are assigned at import time by the LLM sampling the first 800 characters of each text. This is one of the few places in the app where LLM output isn't fact-critical — an approximate difficulty rating is useful even if occasionally off by one level, and a human sanity check pass can correct outliers.

The A1 weighting is deliberate: new learners need sustained reading volume at their level, and Topelius provides it.

---

## Spanish

Finnish was a useful first case because the argument for a deterministic layer is hard to avoid — morphological complexity is high, LLM failure modes are concrete, and Voikko exists as a mature solution.

Spanish is next, and the architecture will be different. Spanish morphology is more regular than Finnish, and LLMs are trained on vastly more Spanish text. The LLM reliability floor for Spanish morphological analysis is substantially higher. The deterministic layer will likely be thinner — possibly covering only irregular verb forms and a handful of edge cases, rather than the full pipeline Voikko handles for Finnish.

The question for Spanish isn't "do we need a deterministic layer" but "how deep does it need to be." That's determined by the same test: where does wrong analysis cause downstream damage, and how reliably does the LLM handle those cases. Finnish required a deep deterministic layer. Spanish may require a shallow one. A future language might require none.

The pattern isn't "always use a deterministic sidecar." It's "identify your LLM's reliability floor for the specific task, and build deterministically for exactly that gap." Finnish made that principle unavoidable. Spanish will test whether it generalizes.

---

*Code: [github.com/lehmann314159/Terve2](https://github.com/lehmann314159/Terve2). Architecture current as of mid-2026.*
