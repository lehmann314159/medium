# I'm Going to Hide Messages in Spotify Playlists

Steganography is the practice of hiding a message inside something that doesn't look like a message. The classic example is invisible ink. The modern version is hiding text in the least significant bits of a PNG. The goal in both cases is the same: the carrier — the image, the letter, the whatever — should attract no suspicion on its own.

A Spotify playlist is a surprisingly good carrier. Playlists are public, shareable, and completely unremarkable. People send them to each other constantly. A playlist called "Sunday morning" or "gym mix" raises no flags. And unlike a PNG, a playlist contains structured text — specifically, track titles — which turns out to be exactly what you need.

The idea is this: hide a message in the letters of track titles, in an order determined by a shared secret. Anyone who knows the secret can decode it. Anyone who doesn't sees a playlist.

## Why track titles

Audio steganography — hiding data in the audio signal itself — is a well-studied field. It's also overkill for what I want to do, and it requires modifying files that Spotify controls. Track titles are different: they're metadata, they're human-readable, and every title in the Spotify catalog is already there waiting to be used.

The shared secret is three keywords. The sender and receiver agree on them in advance — "river", "atlas", "crow", or anything else. The keywords are joined and hashed with SHA-256, producing a 64-bit seed for a pseudorandom number generator. That seed is what makes the extraction deterministic: given the same keywords and the same playlist, the decoder always recovers the same message. The keywords never appear in the playlist. They're out-of-band, communicated separately, and without them the playlist is just a playlist.

The extraction mechanism works as follows: take a track title, strip everything that isn't a letter, lowercase the result, and concatenate into a flat string. The PRNG first draws a count — how many letters to extract from this track — then draws that many position indices into the flat string. Each index selects one letter. The count is capped at two for message tracks, so each track contributes one or two letters to the payload.

The decoder runs the same process in reverse: same keywords, same playlist order, same PRNG sequence, same extracted letters. No computation beyond that.

## A concrete example

Suppose the keywords are "river", "atlas", "crow" and the first three tracks in the playlist are "Stop Making Sense", "Night Moves", and "The One". Their flat letter strings are "stopmakingsense" (15 letters), "nightmoves" (10 letters), and "theone" (6 letters). The PRNG was seeded once from the keywords before any tracks were touched. For "Stop Making Sense" it draws count=2, then positions 3 and 4 — yielding 'p' and 'a' from "stopmakingsense". For "Night Moves" it draws count=1, position 3 — yielding 't' from "nightmoves". For "The One" it draws count=1, position 1 — yielding 'h' from "theone". Those three tracks encode "path". Across twenty tracks, a full short message emerges.

Encoding is the harder direction. Given a message, find a sequence of tracks where the PRNG-extracted letters spell it out. The tracks have to come from a real indexed pool — the Spotify catalog, or some subset of it — and they have to be findable without exhaustive search across millions of titles.

## The cover story problem

A steganographic carrier is only useful if it's convincing. A playlist of randomly selected tracks from across genres, tempos, and decades looks like noise. Anyone paying attention would notice it doesn't behave like a real playlist.

This is where the Camelot wheel comes in.

DJs use the Camelot wheel to mix tracks harmonically. It's a clock-face diagram with 24 positions — 12 major keys and 12 minor keys arranged so that adjacent positions are harmonically compatible. A track in 8B (C major) mixes well with tracks in 7B, 9B, and 8A. Moving more than one or two steps around the wheel produces clashes that trained ears hear immediately.

The wheel gives the encoder a musicality constraint: prefer tracks whose Camelot code is compatible with the previous track. A playlist built this way flows. It sounds like someone made deliberate choices. The steganographic content is invisible because the cover story is plausible.

BPM is a secondary constraint. Tracks with similar tempos feel cohesive; large BPM jumps feel jarring. The encoder scores candidates on both Camelot compatibility and BPM proximity, picks the best available track that also satisfies the letter requirement, and moves on.

The result is a playlist that hides a message and sounds like it was curated.

## What makes this non-trivial

The encoding step — find a track whose title yields the right letter, while also scoring well on Camelot and BPM compatibility — is a constrained search problem. At each position in the playlist, the encoder has a required letter and a previous track, and needs to find the best unused candidate from the pool that satisfies both.

The constraint is tighter than it looks. The PRNG state at each position determines not just which letter gets extracted from a given title, but how many letters are drawn and from which positions in the title. Two tracks with similar titles can yield completely different letters at the same PRNG state. Whether any track in the pool satisfies the requirement at a given position depends on the pool size and the distribution of title content across the catalog.

For small pools this fails regularly. A 26-track pool will routinely hit positions where no track yields the required letter. A real deployment needs thousands of indexed tracks — enough that at any PRNG state, the pool almost certainly contains at least one satisfying candidate at a reasonable musical score.

The search also has to be efficient. Scanning a pool of thousands of tracks at each position is feasible, but the PRNG state can't be advanced speculatively — testing a candidate track without committing to it requires cloning the PRNG state before each probe. The implementation uses a custom Xorshift64 generator with a Clone() method for exactly this reason. Go's standard `math/rand.Rand` doesn't support cloning, which is a silent footgun that only surfaces when you try to write constrained search.

## What I'm building

The system has five components: a Spotify indexer that pulls track metadata into a local SQLite database, an encoder that takes a message and keywords and produces a playlist, a decoder that takes a playlist and keywords and recovers the message, a web frontend for encoding and decoding interactively, and a Camelot wheel visualization that shows the musical structure of the generated playlist.

The indexer runs against the Spotify API and populates the database with track titles, artists, BPM, and musical key data. The encoder and decoder are deterministic given the same keywords — no state is stored anywhere except the playlist itself.

The next article covers the algorithm in detail: how the PRNG seeding works, how the constrained search finds satisfying tracks efficiently, and what the fallback strategy looks like when the pool can't satisfy a requirement at a given position. The article after that covers something I didn't anticipate when I started: what happens when you try to build this kind of system using small open models instead of Claude.

## "I thought you only posted about AI"

This project is partly an end in itself and partly a test case. I've been exploring a structured workflow for orchestrating small open models on complex tasks — what Steve Yegge calls GasTown, with a Mayor role that decomposes problems and a Polecat role that executes them. A real, non-trivial Go project with a clear spec and verifiable outputs is exactly the right environment to stress-test that workflow.

I've already built the core engine directly with Claude as a baseline. Now I'm learning some painful lessons about what happens when the Mayor makes bad decisions and the Polecat can't push back. More on that soon.
