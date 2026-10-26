# ADR-083: Speaker Identification and Voiceprints

| Field | Value |
|-------|-------|
| **Status** | Proposed (exploration) |
| **Date** | 2026-10-26 |
| **Deciders** | Engineering (pending) |
| **Supersedes** | — |
| **Amends** | — |
| **Related** | ADR-008 (event-sourced `VideoRecord` aggregate), ADR-015 (Fireflies import), ADR-021 (Zoom/Fireflies trim), ADR-050 (Fireflies-transcribed-from-Zoom), ADR-053 / ADR-063 / ADR-072 (transcript provenance/extraction) |

---

## Context

The catalog's `VideoRecord` aggregate (`src/catalog/video_record/mod.rs`) stores `participants: Vec<String>` and `transcript_text: Option<String>` as flat, source-supplied values. Speaker attribution today is entirely pass-through: Fireflies does its own voice-based diarization and returns `speakers[].name` plus a per-sentence `speaker_name`, which the import route merges into `participants` and flattens into `transcript_text` as `"[Alice] Hello, today we…"` turn markers (ADR-015). Zoom contributes VTT transcripts with whatever speaker info Zoom itself embeds. **The catalog performs no audio processing, diarization, or identification of its own**, and has no structured per-segment or per-speaker data model — a "speaker" is just a string label, trusted verbatim from whichever source produced it, with no identity that persists across videos or across platforms.

We want video-sync to learn who is speaking across the videos it indexes — independent of whether the source platform already labeled speakers — and build a per-person knowledge base over time (what someone has said across every video they've appeared in).

### Prior art: the pattern already working in two other internal tools

Two separate internal tools (ListeningBuddy and PeopleBuddy) already implement this end to end, over a much larger and messier corpus (~1,269 single-mic voice memos, up to 5 hours long, 1–6 speakers, cars/rooms/calls). Their approach is directly relevant as a reference design, not something to depend on as a library:

1. **Diarization + embedding** (ListeningBuddy, Python, `pyannote.audio`): `pyannote/speaker-diarization-community-1` (free, local, CC-BY-4.0, gated HF download) or `pyannoteAI precision-2` (paid cloud API) produces, per recording, a list of speaker turns (`{s, e, spk}`) plus a 256-dimension voice embedding per diarized speaker, persisted as `<recording-id>.json` (turns) and `<recording-id>.emb.json` (embeddings). A single-recording diarizer label (`SPEAKER_00`) means nothing across recordings on its own — it is local to that file.

2. **Cross-recording identity** (ListeningBuddy, Go, `site/voices.go`): groups every recording's per-speaker embeddings into clusters across the *entire* corpus with a simple iterative centroid clustering (biggest talkers seed the clusters, three re-assignment passes, cosine similarity ≥ 0.6 = "same voice"). Unnamed clusters are surfaced as cards with 2–3 short sample clips; a human names a cluster once and the name applies to every member retroactively. A named person's voiceprint is the running centroid of every clip labeled with their name (`prints.json`). New clusters are scored against known voiceprints and either surfaced as a suggestion (cosine ≥ 0.5) or offered as a one-tap confirm (cosine ≥ 0.65, with an explicit reject path that excludes a cluster from future auto-matching).

3. **Per-person knowledge base** (PeopleBuddy, Python + Go): once a voice is named and linked to a person record, `voices.py` collects every transcript line that person spoke (plus any tasks they own) into an evidence file; `deep.py` has an LLM write per-excerpt notes (chunked, cached by content hash) and then fold them into a narrative dossier per person (summary, relationship, facts, timeline, open promises, quotes, etc.), rebuilt incrementally as new material arrives.

None of this prior art is deployable as-is: it is Go/HTMX + Python scripts running against local files and a SQLite-backed app on the author's own hardware (a Mac mini reachable over Tailscale SSH), not a Next.js/Rust-WASM app on Cloud Run with Workspace IAP and multi-operator (Admin/Publisher/Viewer) access. It is useful as a validated *shape* for the four building blocks (diarize+embed → cluster → name once → build knowledge), not as code to port.

### Open questions this ADR does not resolve

This is deliberately a **Proposed (exploration)** ADR: it records the problem, the reference design, and the decisions that need to be made, without committing to compute placement or a build order. The following must be decided before Phase 1 implementation begins:

1. **Where does diarization run?** video-sync has no GPU host of its own — unlike ListeningBuddy, which has a Mac mini (M4, MPS/ANE) it can SSH a worker to. Candidates:
   - **pyannoteAI cloud API** (`precision-2`/`precision-3`): no infra to stand up; video-sync uploads audio, polls a job, deletes nothing itself (pyannoteAI deletes uploads within 48h, results within 24h). Ongoing per-job cost; audio leaves Google Cloud.
   - **A dedicated worker** (self-hosted GPU/CPU box, or a Cloud Run job with enough CPU for `community-1`, which runs on CPU/GPU/MPS): no per-job fee, audio stays in our infra, but is new infrastructure to operate and keep warm (HF model weights are gated and must be fetched once).
   - **Skip full diarization where a source already provides one.** Fireflies and (partially) Zoom already diarize. A hybrid could run pyannote only on sources with no speaker labels at all (e.g. raw Drive uploads, direct URLs), and compute an embedding-only pass (no re-diarization) on already-segmented sources, to get everyone onto one voiceprint space without redundant compute.
2. **Does identity need to unify across platforms**, i.e., should "spoke in a Fireflies meeting" and "spoke in a raw Zoom recording with no Fireflies pass" resolve to the same person? This is the main reason to run an embedding pass even on sources with existing diarization — otherwise cross-platform identity is impossible by construction.
3. **New domain concept.** The catalog's event-sourced style (ADR-008) suggests a new aggregate rather than overloading `VideoRecord` — something like:
   - `Speaker { id: Uuid, display_name: Option<String>, voiceprint: Vec<f32>, enrolled_from: Vec<SpeakerAppearance> }`
   - `SpeakerAppearance { video_id: Uuid, turns: Vec<{start, end}>, local_label: String }` (the per-video diarizer label, before identity resolution)
   - Events: `SpeakerClusterFormed`, `SpeakerNamed`, `SpeakerAppearanceLinked`, `SpeakerMatchConfirmed`/`Rejected`, `VoiceprintUpdated`.
   - `VideoRecord.participants: Vec<String>` likely stays as the human-readable summary (synchronized from linked `Speaker`s), the way `destination_id`/`destination_url` were kept as synchronized legacy scalars alongside `destination_outcomes` in ADR-077 §1 — avoids a breaking change for the ~current read sites that already use `participants`.
4. **Storage.** The corpus size here (hundreds to low thousands of videos, not millions) does not need a vector database — flat per-speaker embeddings in GCS (alongside the existing GCS-FUSE-mounted bucket pattern) with brute-force cosine comparison, as ListeningBuddy does with JSON files, is almost certainly sufficient. Revisit only if corpus size changes this calculus by an order of magnitude.
5. **The naming/confirm UI.** `voices.go`'s shape (queue of unnamed clusters with sample clips, confirm lane for high-confidence matches against known people, reject records that exclude a cluster from future matching) is a reasonable model for a new `/speakers` (or similar) page in the Next.js catalog app, backed by new API routes mirroring the Fireflies/Zoom import routes' shape. This is UI + API work, independent of where diarization itself runs.
6. **The knowledge-base layer (PeopleBuddy's `deep.py` equivalent)** — an LLM-written narrative per identified speaker built from everything they've said — is additive once identity exists, and should not block Phase 1. It is its own ADR when scoped.
7. **Access and privacy.** video-sync is gated by Workspace IAP but has multiple operator roles (Admin/Publisher/Viewer) by design (ADR-010), unlike PeopleBuddy, which is a single-user tool. A persistent, cross-video, named index of who said what — and eventually a narrative dossier per person — is a different privacy posture than today's per-video transcript display. Before Phase 1 ships, this needs an explicit answer: who can see the speaker index, who can see a person's full cross-video history, and whether identification is opt-in per person or applies globally to anyone who appears in an indexed video. This is a product/policy decision, not an engineering one, and belongs to whoever owns that call for the Agentics Foundation — not assumed by this ADR.

---

## Decision

**Not yet made.** This ADR exists to capture the problem and the reference design so Phase 1 can be scoped from a written record rather than from scratch. A follow-up ADR (or an amendment to this one) should record the actual decision once §1 (compute), §2 (cross-platform identity), and §7 (access/privacy) above are answered.

### Proposed phasing, pending the above

- **Phase 0** (this ADR): record the problem and reference design; get explicit answers to the open questions, especially compute placement and access/privacy, before any code.
- **Phase 1**: diarization/embedding pipeline (wherever §1 lands) + a new `Speaker` aggregate and events + GCS-backed voiceprint storage. No UI yet — this phase should be verifiable by running it against a handful of already-indexed videos and comparing clusters against known participants.
- **Phase 2**: cross-video clustering and a naming/confirm UI in the catalog dashboard, modeled on `voices.go`'s queue/confirm-lane/reject shape.
- **Phase 3**: per-speaker knowledge base (dossier) generation — its own ADR, built only once Phase 1–2 are in use and wanted.

---

## Consequences

**If this goes ahead as scoped:** the catalog gains a cross-video, cross-platform notion of a person independent of what any one source platform calls them, which several future features (search by person, per-person activity, provenance that follows a person rather than just a video) would build on.

**Costs and risks, independent of which compute option §1 picks:**
- A new aggregate and event stream to maintain, with its own migration/backfill story for the videos already in the catalog.
- Diarization is not perfect (ListeningBuddy's research notes 15–25% DER on far-field phone audio as a realistic expectation for the free local model; cleaner video/meeting audio should do better, but this needs measuring against this corpus specifically, not assumed from another corpus's numbers).
- A new privacy surface (§7) that does not exist in the product today and needs an explicit owner's decision, not an engineering default.
- Ongoing cost or operational burden depending on §1's answer (per-job API fees, or infrastructure to run and keep warm).

---

## Alternatives considered

**Do nothing; keep speaker labels as opaque, per-source strings.** Simplest, zero cost, but makes "who said what across every video" and any per-person knowledge base impossible — which is the actual ask.

**Depend on Fireflies/Zoom's own diarization exclusively, with no independent embedding pass.** Cheaper (no new pipeline), but identity never unifies across platforms or across videos that didn't go through Fireflies, and two raw Zoom recordings of the same person get no connection between them at all. Revisit only if cross-platform identity turns out not to matter in practice (§2).

**Port ListeningBuddy/PeopleBuddy's code directly instead of re-implementing the pattern.** Rejected: different language/runtime/hosting model end to end (Go/HTMX/Python/SQLite/local-files-over-SSH vs. Next.js/Rust-WASM/Cloud Run/GCS-FUSE). The value of that prior art is the validated shape of the four building blocks, not literal code.

---

## Related ADRs

- **ADR-008**: the event-sourced aggregate pattern this ADR's proposed `Speaker` aggregate would follow.
- **ADR-015 / ADR-021 / ADR-050**: today's only existing speaker-attribution logic, entirely Fireflies/Zoom pass-through — the baseline this ADR proposes to go beyond.
- **ADR-053 / ADR-063 / ADR-072**: transcript provenance/extraction plumbing — related but about *moving* transcript text correctly, not about identifying who produced it.
- **ADR-010**: the Admin/Publisher/Viewer role model that §7's access question has to be answered within.
- **ADR-077 §1**: the precedent for keeping a legacy scalar field (`destination_id`/`destination_url`) synchronized from a richer new structure (`destination_outcomes`) rather than breaking existing readers — the model proposed in §3 for `participants`.
