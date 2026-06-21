# The Optimization That Broke the Experiment

I built a steganography tool that hides text messages inside Spotify playlists. The encoding scheme picks tracks whose titles contain the right letters in the right order, indexed by a seeded PRNG, with a Camelot-wheel compatibility check so the resulting playlist still sounds coherent rather than random. I built the first version myself, with Claude Code, in a single 18-minute session: 33 files, 16 passing tests.

Then I tried to answer a different question: could a small, locally-run open model build the same system, through a structured workflow where I act as an orchestrating layer between a Claude Mayor and a ~30B polecat model? The answer might have been yes, except I forgot the number one rule of side by side tests: don't make a small change, even if you think it'll be fine.

## Why build it by hand first

This workflow took a page from GasTown, a pattern Steve Yegge described for agent orchestration. I'd read several descriptions of how this kind of thing is supposed to work, but I wanted to implement it myself rather than trust that I understood it from descriptions alone. Building the whole system at once risked taking too much on faith, so I started with a Claude Mayor and a qwen3.6 Polecat, and I handled the notification and communication role manually myself, so I wasn't trying to build a whole automated system before I understood how the pieces actually worked together.

## What went right

The plan file for v0 was converted to nineteen scoped work units. Ten of them, B01 through B10, went cleanly: project scaffold, SQLite schema, a Spotify API client with pagination and retry, the audio provider interface and stub, Camelot mapping and compatibility scoring, the track indexer, the PRNG and key derivation, letter extraction, a greedy playlist builder, and the decoder.

The model doing the work, qwen3.6-27b, performed well within these scoped tasks. It wrote idiomatic Go, kept package boundaries clean, and produced meaningful error messages. On B05 it noticed a missing interface method on its own and added it without being prompted. It read existing tests before writing new ones and avoided naming collisions, which meant it had some awareness of the surrounding repo rather than just the task in front of it. B09 took an unusually high 52,000 tokens because the work units were ordered decoder-before-encoder, which made the round-trip test harder to reason about than it needed to be. The output was still correct. B10 was clean.

And then everything went terribly awry.

## Where it broke

B11, the constrained encoder, failed three times and was eventually marked done without ever passing its own round-trip test. B12, which built the CLI tools and the end-to-end test, failed immediately, on the same underlying problem. V1 was killed at that point — the remaining seven planned units were never attempted.

## The root cause

Both failures traced back to two changes Claude and I had introduced when writing the specs for these work units. Neither change was something the polecat decided on its own. Both were deviations from how the first version actually worked, and neither deviation was something the experiment was designed to test.

**A fixed quota instead of an opportunistic one.** The original version opportunistically encodes 1-3 bytes per track, depending on the track's own title. When Claude and I decomposed the V0 plan document into beads for V1, that changed to a hardcoded requirement of exactly 2 bytes per track. V1 was not able to work with that change. The decoder foundered trying to get the unit tests to pass, and the constrained encoder never got escape velocity.

**Indexed extraction instead of a flat letter string.** The original version concatenates every letter from every word in a track title into one flat string, then draws a position index into that string. This guarantees the extraction always returns exactly the number of bytes requested. My spec for the new version drew a word index, then a character index within that word, then filtered out non-alphabetic characters; depending on the title, that can return zero, one, or two bytes when the caller asked for two. The constraint-checking logic downstream had no way to know this and treated every short return as a pool problem rather than an extraction bug.

I had written both of these into the work-unit specs as straightforward optimizations. Neither was flagged as a deviation from the reference implementation, because I hadn't been tracking the reference implementation as a set of explicit constraints. It was just the thing the first version happened to do.

## Why this invalidated the comparison

The question I was trying to answer was whether a small model, working through this kind of structured workflow, could build the same system the first version built. That question only has an answer if the second version is implementing the same algorithm. It wasn't. It was implementing a different algorithm that happened to be harder to get right, and the model was never given a chance to push back on that, because nothing in the workflow gave it standing to question the spec it was handed.

The model's failures on B11 and B12 weren't capability failures. The model never reached a point on any of the ten completed work units where the task was simply beyond it. Every failure traced back to a decision I had made while writing the spec, not a decision the model made while executing it.

## The immediate fix

I went back through the first version's source and wrote down, explicitly, the algorithmic choices that any future version is required to preserve: an opportunistic, title-derived payload count per track rather than a fixed quota, extraction via a flat letter string with guaranteed exact-count returns, a cloneable PRNG so the constraint search can test candidates without advancing real state, and several other smaller specifics. These are now a standing document, not something to be reconstructed from memory each time. Any future version that wants to diverge from one of these has to say so and justify it; divergence is no longer something that happens by accident inside a work-unit spec.

That fixed the immediate problem. It didn't fix the underlying cause.

## The deeper problem

The important failure here wasn't "I wrote two bad specs." It was that Claude and I, sitting in the orchestrating role, were free to introduce unjustified algorithmic changes during decomposition, and nothing in the workflow was positioned to catch that before it reached the model doing the execution work. A written constraints document helps for this one project. It doesn't generalize, because the next project will have its own implicit assumptions that nobody wrote down either.

That's a structural gap, not a spotify-stego-specific one. Any workflow where an orchestrating layer breaks a design into scoped units for an executor to build has the same exposure: the executor can only build what it's told, and has no mechanism for catching a bad decomposition before acting on it. The orchestrating layer is the only thing that can catch its own mistakes, and in this workflow nothing was checking it.

That gap is what I started designing around. The result is a separate system, Ratchet, built around the idea that the various components of agentic orchestration need observability and auditing. An Adjudicator that takes a design document and turns it into scoped work units needs to verify that the decomposition doesn't introduce change. Part of its job includes preserving the constraints that are actually in the document rather than quietly "improving" on them. It's the role that would have had to catch the two-bytes-per-track change before it ever became a work-unit spec.

Whether it actually catches that kind of thing is a separate, ongoing question. The first real test of the Adjudicator role is running now, comparing candidate models against several real design documents, including this project's. It's not finished, and I'm not going to claim a result here that the experiment hasn't produced yet. But the lineage is direct: a failed comparison between two versions of a playlist encoder is the reason that role exists at all.
