# SongRecognizer — File-Only Product: Checkpoint Handoff

Written 2026-10-11 ~07:30Z, immediately before workspace-credit retirement (~07:20Z window).
Status: **inspection + design checkpoint only — no implementation yet.** Draft PR exists so
these findings survive; the next writer implements from this file.

## Objective

Turn the existing CogWorks capstone implementation (this repo) into one installable,
file-only song-recognition tool that can:

1. index local PCM WAV tracks from an explicit manifest,
2. recognize a local query clip (correct track + offset),
3. reopen a saved index with identical results.

Hard constraints: the normal path must not record a microphone, fetch music, deserialize
pickle, or contact a service. Old notebooks stay as historical material. No use of the
repo's MP3/WAV assets in tests (reuse rights not established) — generate synthetic audio.

## State at checkpoint

- Base: `main` @ `f8dbf9a1e39afefebcf9e892709497fe8db20d60` (verified by fresh clone).
- Rerun guard was clean: no prior task branch, no prior PR existed. This branch is the first.
- Branch: `obv/products-l5-songrecognizer-file-product-20261011`.
- Done: full repo inspection (fingerprint.py, interface.py, spectrogram.py, recording.py,
  README, tree) and the design decisions below. Nothing else.

## Inspection findings — what the repairs must fix

### fingerprint.py (lines 24–31) — suffix-relative indexing bug
- Line 24 computes neighbor distances over the **suffix** `peaks[i:]`, but line 27 indexes
  `peaks[k]` into the **full** array with suffix-relative indices — every anchor after the
  first pairs against the wrong peaks (off-by-`i`).
- Intended method (must be preserved): combinatorial acoustic hashing —
  `(f_anchor, f_target, delta_t)` pairs from a forward-time fanout, stored with the
  **absolute** anchor time. Do NOT substitute a filename or exact-byte lookup.
- Line 25 selects `fanout` nearest by Euclidean distance ignoring direction, then line 28
  filters `new_time <= absolute_time` after selection, so effective fanout varies. Fix
  shape: select the next `fanout` peaks within a forward-time window from the full peak
  list; keep the absolute anchor time on each triple.

### interface.py — broken identity/schema + pickle persistence
- Persistence is pickle (lines 15, 22–28). Replace with a versioned, non-executable JSON
  format, atomic save (temp file + `os.replace`), so a failed/interrupted save never
  destroys the previous index, and a corrupt file is a clean error, not a crash.
- `add_song` (62–96): line 82 KeyErrors on a fresh db (`self.db["artist_information"][1]`);
  line 85 overwrites the whole `"artist_information"` mapping with a tuple; line 94
  KeyErrors because `self.db["songs"][(fm, fn, dt)]` is never initialized. Track identity
  must be a stable, explicitly assigned id (not `len()` of a mutated dict, not the name).
- `inspect_song` (30–34): references undefined `id` (should be `fp`).
- `delete_song` (39–50): `for songs in self.db["songs"]` iterates dict KEYS, then calls
  `.values()` on a key string.
- `compare_fingerprints` (124–135): the `highest[1]/len(fingerprints) > 0.5` gate treats a
  raw match count as a calibrated probability. Required: report ranked matches with counts
  and offset evidence, and an explicit "unknown" result when there is no evidence — never
  a probability claim.
- Re-indexing a known track must not duplicate entries (idempotent upsert by stable id).

### spectrogram.py
- Keep the reference semantics of `mlab.specgram` (NFFT=4096, noverlap=2048, hanning) but
  reimplement as a numpy-only STFT so matplotlib/librosa/numba are not runtime deps
  (numba is imported yet `peak_finder` is plain Python loops; librosa/matplotlib imports
  are heavy and graph-plotting has no place in the CLI path).

### recording.py
- `from microphone import record_audio` at module top level: the shipped package must have
  ZERO import-time side effects (no mic, no filesystem writes, no network). The recording
  path stays out of the package entirely.
- `librosa.load` silently resamples to 22050 Hz mono. The product must handle sample
  rate/channels explicitly: read them from the WAV header, downmix stereo, require the
  clip rate to match the indexed rate, and record both in the index schema.

### Repo hygiene
- No packaging (no pyproject.toml), no pytest suite — notebooks only. `tests/*.ipynb` and
  `.ipynb_checkpoints` remain as historical material; do not delete or rewrite them.
- `assets/*.mp3`, `clips/*.wav`, `music/bestsong.mp3`, `example-recording.npy`: NOT used in
  tests or fixtures (reuse rights not established).

## Target design (decided at checkpoint)

- Package `songrecognizer/`:
  - `wavio.py` — stdlib `wave`, PCM 16-bit decode, bounded file-size/duration limits.
  - `spectro.py` — numpy-only STFT (mlab-equivalent params).
  - `fingerprint.py` — fixed forward-fanout combinatorial hashing (absolute anchor time).
  - `index.py` — `SongIndex`: `hash -> [(track_id, anchor_t)]`; stable track ids
    (deterministic, e.g. content hash of decoded PCM); JSON schema
    `{"format": "songrecognizer-index", "version": 1, ...}`; atomic save; idempotent
    re-index; explicit sample rate/channel fields.
  - `match.py` — offset-alignment histogram over candidate (track, offset) pairs; ranked
    results carrying counts + offset evidence; explicit UNKNOWN when no evidence.
  - `cli.py` — `songrecognizer index --manifest M --index I` and
    `songrecognizer recognize --clip C --index I`; `pyproject.toml` console-script entry.
- Manifest: explicit text/JSON file listing WAV paths + track names.
- All test audio generated locally in-test (sine/tone motifs with hand-derived controls).

## NOT done (everything below this line)

No package code, no CLI, no tests, no fixtures, no verification runs, no fresh-install
smoke test, no index/query/reopen transcript. Tests run: 0. Nothing is verified.

## Exact next steps (in order; commit + push each tested milestone)

1. `git fetch origin && git checkout obv/products-l5-songrecognizer-file-product-20261011`
   (branch exists — do not recreate; base work on the tip of this branch).
2. Set git config before committing:
   `git config user.name 'Sam Gu' && git config user.email '127461594+SamGu-NRX@users.noreply.github.com'`
   Every commit message ends with:
   `Co-authored-by: obvious-autobuild[bot] <262744130+obvious-autobuild[bot]@users.noreply.github.com>`
   Sam Gu stays the author; verify with
   `git log --format='%an <%ae>' -1` before every push. Never rebase/amend/force-push.
3. Implement in this order: (a) wavio + spectro + fingerprint with one generated fixture
   and a hand-derived control fingerprint; (b) index.py JSON persistence, atomic save,
   idempotent re-index; (c) match.py ranked/UNKNOWN + CLI + pyproject; (d) edge-case suite:
   known track/offset, later-anchor fingerprints, repeated motifs, silence, short clips,
   tied matches, duplicate indexing, corrupt index, interrupted save; (e) usage docs.
4. Exercise the **installed** CLI: `python -m venv /tmp/venv-sr && /tmp/venv-sr/bin/pip
   install .` then run the installed `songrecognizer` end-to-end; record the
   index/query/reopen transcript. Tests: `python -m pytest tests/ -v` — commit exact
   pass/fail counts and observed limitations in the PR body.
5. Push at least every 30 minutes. Keep the PR draft. Never merge, deploy, change
   accounts, or rewrite history. Do not touch CogWorks benchmarks/learning-lab repos or
   any CI/deploy settings.

## Verify commands

- Author check: `git log --format='%an <%ae>' origin/obv/products-l5-songrecognizer-file-product-20261011 -1`
  → `Sam Gu <127461594+SamGu-NRX@users.noreply.github.com>`
- Suite: `python -m pytest tests/ -v` (after `pip install -e .`)
- End-to-end: `songrecognizer index --manifest examples/manifest.txt --index /tmp/sr-index.json`
  then `songrecognizer recognize --clip <clip.wav> --index /tmp/sr-index.json`

## Resume

Steer the existing task (do not create a competing writer):
`obvious tasks steer --taskId=todo_a9UI6Lst --message="resume SongRecognizer file product"`
If the workspace is gone, re-clone and continue on this branch:
`git clone https://github.com/SamGu-NRX/SongRecognizer.git && cd SongRecognizer && git checkout obv/products-l5-songrecognizer-file-product-20261011`
