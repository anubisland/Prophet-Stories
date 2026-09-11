# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status: greenfield, design approved

No application code exists yet. The approved design is
[`docs/superpowers/specs/2026-09-11-prophet-stories-design.md`](docs/superpowers/specs/2026-09-11-prophet-stories-design.md)
— **read it before writing any code.** This file is the short form; the spec is the authority.

Reference implementations (separate repos, same author, same conventions):

- `../Sera` — Seerah / Rashidun timeline. Read its `AGENTS.md` for the bug catalogue.
- `../islamic_battles` — battles atlas. Source of the narration + intro-audio pattern.

This project is **independent** of both. It inherits their conventions, not their code.

## What this project is

An Arabic-first (RTL), **single-page, no-build, vanilla-JS** timeline of the stories of
the Prophets (قصص الأنبياء), in chronological order, one prophet at a time. English
fields exist in the schema but are left empty for now.

Entry point is an **intro screen** (`data/intro.js`) covering what the project is, why the
stories of the prophets matter, and the sourcing method. It has its own narration.

Each prophet has a variable number of **phases** — Musa may take ten, Idris one. The count
is never padded to a fixed template; padding is what invites Isrā'īliyyāt in.

Every prophet's **first phase is the state of his people before he was sent** — the
condition of the land or city that necessitated his mission.

Each phase carries: an ayah chosen for *that situation* (not generic), narration split
into **beats**, an SVG map whose focus advances with the audio, contemporary-figure cards,
lesson cards, and classified sources.

## Hard rules

1. **No build step, no framework, no runtime npm dependency.** Must work from `file://`
   and any static host.
2. **No `fetch`, no `<script type="module">`** — both are blocked on `file://` by CORS.
   Dynamic loading is by **injecting a classic `<script>` tag**. Every data and timing
   file is `.js` assigning to a global, never `.json`.
3. **Single state object** (`app/state.js`): `{ SCREEN, PROPHET, PHASE, LANG, VOICE, RECITER }`.
   Nothing else writes to it directly. No new globals.
4. **Inline SVG only** for maps — never raster.
5. **One concern per module.** `../Sera` has a 70KB `app.js` and a 661KB `data.js`; that
   is the thing we are deliberately not repeating. See the spec's file map.
6. Conventional Commits; bump the version and **cache-bust the `?v=` query** on any
   changed `app/*.js`, `data/*.js`, or `style.css` in `index.html`. Stale-cache
   regressions have shipped repeatedly in `../Sera`.

## Sourcing — the point of the project

Order of authority: **the Qur'an** first and judging over all else → **sahih hadith**
(Bukhari and Muslim especially) → **Tafsir Ibn Kathir** → **Qisas al-Anbiya by Ibn Kathir**
→ **al-Bidayah wa'l-Nihayah**.

**Tarikh al-Tabari** with caution — it transmits with chains and without grading. Accept
only what agrees with Qur'an and Sunnah; never rely on it alone.

**Rejected outright:** Qisas al-Anbiya of al-Thalabi, Ara'is al-Majalis of al-Kisa'i, and
all non-Sunni collections.

islamweb.net and islamqa.info are classification tools — record the fatwa URL in
`srcs[].url` as the witness that a classification was not self-invented.

### Isrā'īliyyāt are structurally impossible, not filtered

There is no "labelled Isrā'īliyyāt card" and no exception. Classification is binary:
`classAr: "ثابت"` enters; anything else does not exist in the file.

No automated test can detect Isrā'īliyyāt semantically — a keyword list fails both ways,
passing a reworded one and rejecting a text that cites a report **in order to refute it**.
So the gate is not detection after entry; it is **leaving no door**: narration prose
cannot exist without an established chain, and an Isrā'īliyyah by definition has none.

`tools/check_sources.py`, strongest layer first:

1. **Every `narr` beat has non-empty `srcRefs`** pointing at valid `srcs` entries. An
   unsourced beat fails. No orphan narration.

   **The one exception is `kind: "link"`** — a purely grammatical connective, exempt from
   sourcing. `kind` defaults to `narr`, so forgetting it demands a source: the oversight
   fails safe.

   "It's just a linking sentence" is exactly the cover any Isra'iliyyah would wear, so the
   guarantee is not trusting the label but **making the opening too narrow for a report to
   fit**. Five mechanical constraints on a `link` beat:

   - **≤ 80 characters** — a paragraph cannot hide in a connective.
   - **No digits** (Arabic or Arabic-Indic) — a number is always a report: a date, an age, a count.
   - **No `focus`, no `pin`** — a connective does not move the listener; geography is a report.
   - **No two adjacent `link` beats** — prevents chaining them into continuous narration.
   - **Never the first or last beat** — a phase opens and closes on sourced material.

   This does open a small door, with no claim that it is shut. But what fits through is not
   a report: «وهنا يبدأ الابتلاء» is a connective; «فركب معه ثمانون رجلاً» fails on digits;
   «فانطلق إلى أرض بابل» fails on place. Layers 2–4 below apply to every beat regardless.
2. **Only `classAr: "ثابت"` is accepted.** Any other value fails. No override flag.
3. **Blacklist of books and transmitters** — al-Thalabi, al-Kisa'i, Ara'is al-Majalis,
   and the known Isrā'īliyyāt transmitters **Ka'b al-Ahbar and Wahb ibn Munabbih**, plus
   any citation of the Torah / Genesis / Old Testament.
4. **Review flags, not rejections** — Dawud and Uriah, Sulayman's ring and the devil on
   his throne, the names and count of the ark's passengers, Adam's height and lifespan,
   the details of Harut and Marut. These may appear *to refute* them; a machine cannot
   tell, so the build stops for a human.

## Audio

Two independent paths with different failure behaviour.

**Narration** — pre-generated MP3s **committed to the repo**. Works offline and from
`file://`. Three voice slots, only two generated for now: **`shakir` (شاكر, `ar-EG-ShakirNeural`,
the default narrator)** and `story` (سلمى, `ar-EG-SalmaNeural`) — both Egyptian, so the
app's register stays consistent. `classic` (حامد, `ar-SA-HamedNeural`) is declared but not
generated until asked.

**Recitation** — streamed from everyayah.com, parsed from `ayahRefEn`. **Default reciter:
al-Husary**, with a picker. Not committed; needs network. If the network drops, narration
still works and recitation goes silent.

**No live TTS anywhere.** `speechSynthesis` and `translate_tts` were removed from
`../Sera` for sounding robotic and ignoring the chosen voice. A missing clip shows
"audio not available" and stops there.

### Map sync

`gen_tts.py` uses `comm.stream()` (not `comm.save()`) to capture edge-tts boundary events.

**Verified on edge-tts 7.2.8 (2026-09-11): `WordBoundary` is never emitted** — not for
Arabic, not for English (tested Shakir, Salma, Hamed, Brian). What comes back is
**`SentenceBoundary`**, one event per sentence carrying `offset`, `duration`, and `text`.
That is more precise for our purpose than word events would be.

**Consequence: every beat must be whole sentences**, ending in `.` `؟` `!` or `:`. The
generator maps events to beats by counting each beat's sentences in order; a beat's cue is
its first sentence's `offset`. `check_release.py` verifies each beat's sentence count
matches its events — a mismatch means the prose does not align to sentence boundaries and
is rejected. Output is
`audio/<slot>/<prophet>_<phase>_<lang>.cues.js` — a **JS file**, because `.json` dies on
`file://`. At runtime `timeupdate` compares `currentTime` against those cues and moves the
map focus. Cues are generated per voice, so differing voice durations break nothing.

### Arabic vocalization — the most expensive trap

**There is no stored `descAr`.** Narration text is derived by joining `beats[].textAr`.
In `../Sera` the text was stored twice — bare on screen and diacritized in a sidecar — and
the two drifted, shipping wrong audio repeatedly. One source of truth removes the entire
bug class.

On-screen text stays bare. Audio is generated from the diacritized
`tools/narration_ar.json`, keyed `<prophet>_<phase>_<beat>`, build-time only — the app
never loads it. Entries must keep the consonantal skeleton identical (add only harakāt)
and be genuinely vocalized. `check_voc.py` gates three failure modes that all shipped live:

- **BARE** — bare text copy-pasted into the sidecar. Shipped at 4.8% and 0.6% density.
- **FAKE** — a fatha mechanically stamped after nearly every letter, zero sukun/shadda.
  Scored 91–96% and passed the old gate while reading as garbage. **Density proves nothing.**
- **FUSION** — dropped separators fusing adjacent words into TTS non-words.

Narration text must contain **no non-Arabic letters** (stray CJK from machine translation
has shipped before).

## Commands

The toolchain is ported from `../Sera` as the project acquires code; none of it exists yet.

```bash
npx --yes serve .        # serve locally; never commit the resulting node_modules/
node --check app/main.js # no build, so this is the only "compile"
```

Once `tools/` exists:

```bash
python tools/gen_tts.py --prophet nuh --force   # regenerate that prophet's clips + cues
python tools/check_voc.py                       # after ANY data edit
python tools/check_sources.py                   # the sourcing gate
python tools/check_release.py                   # MANDATORY before every commit — exit 0 or don't commit
```

No test framework, linter, or formatter — and none is being added. Verification is the
gates plus opening `index.html` in a browser.

## Release gates beyond sourcing

- `ayahRefEn` parses and actually resolves on everyayah.com.
- `data-ar` count equals `data-en` count in `index.html`.
- `<section` count equals `</section>` count — a missing tag blanked the site **twice** in
  `../Sera`; the gate does not parse HTML nesting.
- Audio coverage: every phase × enabled slot × language exists and is non-empty.
- Every MP3 has a matching cues file whose entry count equals that phase's `beats` count.
- `?v=` bumped in `index.html` for every changed `app/*.js`, `data/*.js`, `style.css`.

## Debugging notes inherited from `../Sera`

- **Check `z-index` and `position: fixed` first** on any visual bug. A `z-index: 9999`
  diagnostic badge with a dark-green background caused a phantom "green strip" that
  survived **ten** false fixes.
- Any fixed-position element with `z-index > 100` and a background **must** default to
  `display: none`.
- Adding a top-level screen is never "just one screen": every `show*()` must set the
  *entire* state→DOM mapping, and every path keyed to a specific screen or to the current
  unit (safety-net CSS, map-focus gate, audio URL router) must be updated. Four separate
  live regressions came from this one pattern.
- Don't rely on `requestAnimationFrame` for positioning — browsers pause it in hidden tabs.
