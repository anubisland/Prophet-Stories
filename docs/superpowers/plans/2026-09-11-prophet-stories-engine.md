# Prophet Stories Engine — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the intro screen and the complete story of Nuh ﷺ — narrated, mapped, sourced — on an engine that adding any later prophet is routine data work.

**Architecture:** No-build vanilla JS. `index.html` + `style.css` + one-concern modules under `app/`. Prophet data lives one file per prophet under `data/`, injected as a classic `<script>` on demand. Narration is pre-generated committed MP3s plus a sibling `.cues.js` of sentence timings; the map focus advances off those cues via `timeupdate`. Recitation streams from everyayah.com.

**Tech Stack:** Vanilla ES2018 JS (no modules, no framework, no bundler), Python 3.12 + edge-tts 7.2.8 for generation, Node 22 for syntax checks and gates.

## Global Constraints

Copied verbatim from [the spec](../specs/2026-09-11-prophet-stories-design.md). Every task's requirements implicitly include this section.

- **No build step, no framework, no runtime npm dependency.** Must work from `file://` and any static host.
- **No `fetch`, no `<script type="module">`** — blocked on `file://` by CORS. Dynamic loading injects a classic `<script>` tag. Every data and timing file is `.js` assigning to a global, never `.json`.
- **Single state object** in `app/state.js`: `{ SCREEN, PROPHET, PHASE, LANG, VOICE, RECITER }`. Nothing else writes it directly. **"No new globals" means no new mutable *state* globals** — scattered state is the thing being forbidden. With no module system (blocked on `file://`), attaching functions to `window` is how this codebase exports; each module namespaces its own and owns them. The only global *data* objects are `window.PROPHETS`, `window.PROPHET_INDEX`, `window.INTRO`, `window.CUES`, and the only global mutable state is `window.APP_STATE`.
- **`VERSION` in `app/loader.js` must equal `package.json`'s version.** The gate checks this — a drifted value silently requests stale data and cue files that the `index.html` cache-bust check cannot see.
- **Inline SVG only** for maps — never raster.
- **Arabic first.** `*En` fields exist in every structure and stay `""`. Every UI string carries both `data-ar` and `data-en`; the two counts must match.
- **Voice slots:** `shakir` = `ar-EG-ShakirNeural` (default), `story` = `ar-EG-SalmaNeural`. Both generated. `classic` = `ar-SA-HamedNeural` declared, not generated.
- **Reciter:** al-Husary default — `https://everyayah.com/data/Husary_128kbps/SSSAAA.mp3` (3-digit surah, 3-digit ayah). Verified 2026-09-11.
- **Cues come from `SentenceBoundary`**, never `WordBoundary` (edge-tts 7.2.8 emits none). Every beat must be whole sentences ending in `.` `؟` `!` or `:`.
- **No live TTS.** Never `speechSynthesis` or `translate_tts`. A missing clip shows "الصوت غير متاح" and stops.
- **Sourcing:** every `narr` beat needs non-empty `srcRefs`; only `classAr: "ثابت"` is accepted. `link` beats are exempt but constrained (≤80 chars, no digits, no `focus`/`pin`, never adjacent, never first or last).
- **Commits:** Conventional Commits. Bump `package.json` version and the `?v=` query in `index.html` for every changed `app/*.js`, `data/*.js`, `style.css`.
- **`python tools/check_release.py` must exit 0 before every commit.**

### A note on "tests" in this project

There is no test framework and none is being added — that is a deliberate spec decision. The gate scripts under `tools/` **are** the test suite. So TDD here means: **add the check to the gate first, run it and watch it fail, then write the code or data that makes it pass.** Every task below follows that cycle.

---

### Task 1: Repository skeleton and the release gate

**Files:**
- Create: `index.html`, `style.css`, `package.json`, `.gitignore`, `.nojekyll`
- Create: `app/state.js`, `app/main.js`
- Create: `tools/check_release.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `window.APP_STATE` — the single mutable state object, shape `{ SCREEN: 'intro'|'home'|'prophet', PROPHET: string|null, PHASE: number, LANG: 'ar'|'en', VOICE: string, RECITER: string }`. `window.setState(patch)` merges a partial and calls every function registered by `window.onStateChange(fn)`. `tools/check_release.py` exits 0 clean, 1 with a list of problems.

- [ ] **Step 1: Write the gate first, before any code it checks**

Create `tools/check_release.py`:

```python
#!/usr/bin/env python3
"""Release gate. Exit 0 = safe to commit; exit 1 = fix what it lists.

Every check here exists because its failure mode shipped live in ../Sera at
least once. Run before EVERY commit.
"""
import os, re, subprocess, sys

ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
problems = []


def read(rel):
    with open(os.path.join(ROOT, rel), encoding="utf-8") as f:
        return f.read()


def js_files():
    """Every checked-in .js file, relative to ROOT, sorted."""
    out = []
    for sub in ("app", "data"):
        d = os.path.join(ROOT, sub)
        if not os.path.isdir(d):
            continue
        for name in sorted(os.listdir(d)):
            if name.endswith(".js"):
                out.append(f"{sub}/{name}")
    return out


def check_js_syntax():
    for rel in js_files():
        r = subprocess.run(["node", "--check", os.path.join(ROOT, rel)],
                           capture_output=True, text=True)
        if r.returncode != 0:
            problems.append(f"SYNTAX  {rel}: {r.stderr.strip().splitlines()[0]}")


def check_section_balance():
    """A missing </section> nested the whole app inside one screen and blanked
    the site TWICE in ../Sera. The gate does not parse HTML, so count tags."""
    h = read("index.html")
    o = len(re.findall(r"<section\b", h))
    c = len(re.findall(r"</section>", h))
    if o != c:
        problems.append(f"HTML    <section> {o} != </section> {c}")


def check_bilingual_balance():
    h = read("index.html")
    ar = len(re.findall(r"data-ar=", h))
    en = len(re.findall(r"data-en=", h))
    if ar != en:
        problems.append(f"I18N    data-ar {ar} != data-en {en}")


def check_loader_version():
    """app/loader.js injects data and cue files with its own hardcoded VERSION.
    index.html's cache-bust check cannot see those, so drift here serves stale
    data silently."""
    ver = re.search(r'"version"\s*:\s*"([^"]+)"', read("package.json")).group(1)
    path = os.path.join(ROOT, "app", "loader.js")
    if not os.path.exists(path):
        return
    m = re.search(r'VERSION\s*=\s*"([^"]+)"', read("app/loader.js"))
    if not m:
        problems.append("CACHE   app/loader.js defines no VERSION")
    elif m.group(1) != ver:
        problems.append(f"CACHE   app/loader.js VERSION={m.group(1)}, package.json={ver}")


def check_cache_bust():
    """Every local app/ data/ .js and style.css referenced from index.html must
    carry ?v=<package version>. Stale-cache regressions shipped repeatedly."""
    h = read("index.html")
    ver = re.search(r'"version"\s*:\s*"([^"]+)"', read("package.json")).group(1)
    for m in re.finditer(r'(?:src|href)="((?:app|data)/[^"?]+\.js|style\.css)(\?v=([^"]*))?"', h):
        path, _, got = m.groups()
        if got != ver:
            problems.append(f"CACHE   {path} has ?v={got or '(none)'}, expected {ver}")


def main():
    check_js_syntax()
    check_section_balance()
    check_bilingual_balance()
    check_cache_bust()
    check_loader_version()
    if problems:
        print(f"\n{len(problems)} problem(s):\n")
        for p in problems:
            print("  " + p)
        return 1
    print("check_release: clean")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 2: Run the gate to verify it fails**

Run: `python tools/check_release.py`
Expected: FAIL — a traceback on missing `index.html`. That is the point: the gate is real before the code is.

- [ ] **Step 3: Create `package.json`, `.gitignore`, `.nojekyll`**

`package.json`:

```json
{
  "name": "prophet-stories",
  "version": "0.1.0",
  "private": true,
  "description": "Timeline of the stories of the Prophets — Qur'an and authenticated Sunnah only",
  "scripts": {
    "start": "npx --yes serve ."
  }
}
```

`.gitignore`:

```
node_modules/
.DS_Store
Thumbs.db
__pycache__/
```

`.nojekyll` — empty file. GitHub Pages otherwise ignores paths starting with underscore.

- [ ] **Step 4: Create `app/state.js`**

```js
/* The single state object. Nothing else writes to it directly. */
(function () {
  "use strict";

  window.APP_STATE = {
    SCREEN: "intro",   // 'intro' | 'home' | 'prophet'
    PROPHET: null,     // prophet key, e.g. 'nuh'
    PHASE: 0,
    LANG: "ar",
    VOICE: "shakir",
    RECITER: "Husary_128kbps"
  };

  var listeners = [];

  window.onStateChange = function (fn) {
    listeners.push(fn);
  };

  window.setState = function (patch) {
    var key;
    for (key in patch) {
      if (Object.prototype.hasOwnProperty.call(patch, key)) {
        window.APP_STATE[key] = patch[key];
      }
    }
    for (var i = 0; i < listeners.length; i++) {
      listeners[i](window.APP_STATE);
    }
  };
})();
```

- [ ] **Step 5: Create `app/main.js`**

```js
/* Wiring and init. Every show* path must set the ENTIRE state->DOM mapping —
   partial updates caused four separate live regressions in ../Sera. */
(function () {
  "use strict";

  function init() {
    document.documentElement.lang = window.APP_STATE.LANG;
    document.documentElement.dir = window.APP_STATE.LANG === "ar" ? "rtl" : "ltr";
  }

  if (document.readyState === "loading") {
    document.addEventListener("DOMContentLoaded", init);
  } else {
    init();
  }
})();
```

- [ ] **Step 6: Create `index.html` and `style.css`**

`index.html`:

```html
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>قصص الأنبياء — خط زمني موثّق</title>
<link rel="stylesheet" href="style.css?v=0.1.0">
</head>
<body>

<section id="intro-screen">
  <h1 data-ar="قصص الأنبياء" data-en="Stories of the Prophets">قصص الأنبياء</h1>
</section>

<script src="app/state.js?v=0.1.0"></script>
<script src="app/main.js?v=0.1.0"></script>
</body>
</html>
```

`style.css`:

```css
:root {
  --bg: #071a15;
  --ink: #f2ece0;
  --gold: #c5a059;
  --emerald: #063529;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  background: var(--bg);
  color: var(--ink);
  font-family: "Segoe UI", Tahoma, system-ui, sans-serif;
  line-height: 1.8;
}

/* Any fixed element with z-index > 100 and a background MUST default to
   display:none. A diagnostic badge that ignored this rendered a phantom green
   strip in ../Sera that survived ten false fixes. */
```

- [ ] **Step 7: Run the gate to verify it passes**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

Then confirm the gate actually bites — temporarily change `?v=0.1.0` to `?v=0.0.9` on the stylesheet, rerun, and expect `CACHE style.css has ?v=0.0.9, expected 0.1.0`. Restore it.

- [ ] **Step 8: Commit**

```bash
git add index.html style.css package.json .gitignore .nojekyll app/ tools/
git commit -m "feat: repository skeleton and release gate"
```

---

### Task 2: Intro screen

**Files:**
- Create: `data/intro.js`, `app/nav.js`
- Modify: `index.html`, `app/main.js`

**Interfaces:**
- Consumes: `window.APP_STATE`, `window.setState`, `window.onStateChange` (Task 1).
- Produces: `window.INTRO` — `{ titleAr, titleEn, beats: [{ kind, textAr, textEn }] }`. `window.showScreen(name)` sets `APP_STATE.SCREEN` and toggles every screen element; it is the only function allowed to change screen visibility.

- [ ] **Step 1: Add the screen-exhaustiveness check to the gate**

Add to `tools/check_release.py`, and call it from `main()`:

```python
SCREENS = ["intro-screen", "home-screen", "prophet-screen"]


def check_screens_exhaustive():
    """Every screen id must exist in index.html AND be handled in nav.js.
    A new screen that some show* path forgets is the single most repeated
    regression pattern in ../Sera (four separate live breakages)."""
    h = read("index.html")
    nav = read("app/nav.js") if os.path.exists(os.path.join(ROOT, "app/nav.js")) else ""
    for s in SCREENS:
        if f'id="{s}"' not in h:
            problems.append(f"SCREEN  #{s} missing from index.html")
        if s not in nav:
            problems.append(f"SCREEN  #{s} not handled in app/nav.js")
```

- [ ] **Step 2: Run the gate to verify it fails**

Run: `python tools/check_release.py`
Expected: FAIL, six lines — `#home-screen missing from index.html`, `#prophet-screen missing...`, and all three `not handled in app/nav.js`.

- [ ] **Step 3: Create `data/intro.js`**

Beats must be whole sentences. `kind` is `"link"` only for pure connectives.

```js
/* Intro screen content. No maps, no figures, no sources — this introduces the
   project, it does not narrate history. */
window.INTRO = {
  titleAr: "قصص الأنبياء",
  titleEn: "",
  subtitleAr: "خط زمني موثّق من القرآن والسنة الصحيحة",
  subtitleEn: "",
  ayah: "لَقَدْ كَانَ فِي قَصَصِهِمْ عِبْرَةٌ لِأُولِي الْأَلْبَابِ",
  ayahRef: "سورة يوسف، الآية ١١١",
  ayahEn: "",
  ayahRefEn: "Surah Yusuf (12), verse 111",
  beats: [
    { kind: "narr", textAr: "هذا خط زمني لقصص الأنبياء عليهم السلام، يعرضها بالترتيب الزمني نبيًّا بعد نبي. لكل نبي مراحل تبدأ بحال قومه قبل أن يُبعث إليهم.", textEn: "" },
    { kind: "narr", textAr: "لكل مرحلة سرد مسموع، وخريطة تتحرك مع السرد، وبطاقات لمن عاصره، وبطاقات للعبر المستفادة.", textEn: "" },
    { kind: "link", textAr: "وأما المنهج فهو أصل هذا العمل.", textEn: "" },
    { kind: "narr", textAr: "المصادر هي القرآن الكريم أولًا وحاكمًا، ثم ما صح من السنة، ثم تفسير ابن كثير وقصص الأنبياء له والبداية والنهاية. ولا يدخل هذا التطبيق شيء من الإسرائيليات.", textEn: "" }
  ]
};
```

- [ ] **Step 4: Create `app/nav.js`**

```js
/* The ONLY place screen visibility changes. Every call sets the complete
   state->DOM mapping for all screens — never a partial update. */
(function () {
  "use strict";

  var SCREENS = ["intro-screen", "home-screen", "prophet-screen"];

  window.showScreen = function (name) {
    var id = name + "-screen";
    for (var i = 0; i < SCREENS.length; i++) {
      var el = document.getElementById(SCREENS[i]);
      if (el) { el.hidden = (SCREENS[i] !== id); }
    }
    window.setState({ SCREEN: name });
  };

  window.renderIntro = function () {
    var d = window.INTRO;
    if (!d) { return; }
    document.getElementById("intro-title").textContent = d.titleAr;
    document.getElementById("intro-subtitle").textContent = d.subtitleAr;
    document.getElementById("intro-ayah").textContent = d.ayah;
    document.getElementById("intro-ayah-ref").textContent = d.ayahRef;

    var body = document.getElementById("intro-body");
    body.textContent = "";
    for (var i = 0; i < d.beats.length; i++) {
      var p = document.createElement("p");
      p.className = "beat beat-" + d.beats[i].kind;
      p.textContent = d.beats[i].textAr;
      body.appendChild(p);
    }
  };
})();
```

- [ ] **Step 5: Add the three screens to `index.html`**

Replace the existing `<section id="intro-screen">` block with:

```html
<section id="intro-screen">
  <h1 id="intro-title" data-ar="قصص الأنبياء" data-en="Stories of the Prophets"></h1>
  <p id="intro-subtitle" data-ar="خط زمني موثّق" data-en="A sourced timeline"></p>
  <blockquote class="ayah">
    <span id="intro-ayah"></span>
    <cite id="intro-ayah-ref"></cite>
  </blockquote>
  <div id="intro-body"></div>
  <button id="intro-enter" data-ar="ابدأ" data-en="Start">ابدأ</button>
</section>

<section id="home-screen" hidden>
  <h2 data-ar="الأنبياء" data-en="The Prophets"></h2>
  <ul id="prophet-list"></ul>
</section>

<section id="prophet-screen" hidden>
  <button id="btn-back-home" data-ar="رجوع" data-en="Back">رجوع</button>
  <h2 id="prophet-name"></h2>
  <div id="phase-body"></div>
</section>
```

Add before `app/main.js`:

```html
<script src="data/intro.js?v=0.1.0"></script>
<script src="app/nav.js?v=0.1.0"></script>
```

- [ ] **Step 6: Wire it in `app/main.js`**

Replace the `init` function body with:

```js
  function init() {
    document.documentElement.lang = window.APP_STATE.LANG;
    document.documentElement.dir = window.APP_STATE.LANG === "ar" ? "rtl" : "ltr";
    window.renderIntro();
    window.showScreen("intro");
    document.getElementById("intro-enter").addEventListener("click", function () {
      window.showScreen("home");
    });
  }
```

- [ ] **Step 7: Run the gate to verify it passes**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

Then open `index.html` directly from the filesystem (double-click — this proves the `file://` requirement). Expected: the intro renders in RTL, and "ابدأ" switches to an empty home screen.

- [ ] **Step 8: Commit**

```bash
git add index.html app/nav.js app/main.js data/intro.js tools/check_release.py
git commit -m "feat: intro screen and screen navigation"
```

---

### Task 3: Prophet index and on-demand loader

**Files:**
- Create: `data/index.js`, `app/loader.js`
- Modify: `index.html`, `app/main.js`, `app/nav.js`

**Interfaces:**
- Consumes: `window.showScreen` (Task 2).
- Produces: `window.PROPHET_INDEX` — an array of `{ key, nameAr, nameEn, order, ready }` sorted by `order`. `window.loadProphet(key, cb)` injects `data/<key>.js?v=<version>` once, then calls `cb(err, data)` where `data` is `window.PROPHETS[key]`. Repeat calls for a loaded key call back synchronously.

- [ ] **Step 1: Add the index/loader consistency check to the gate**

```python
def check_prophet_index():
    """Every index entry marked ready must have a data file, and every data file
    must appear in the index. A ready entry with no file is a dead card."""
    idx = read("data/index.js")
    keys = re.findall(r'key:\s*"([a-z0-9_-]+)"', idx)
    ready = re.findall(r'key:\s*"([a-z0-9_-]+)"[^}]*ready:\s*true', idx)
    for k in ready:
        if not os.path.exists(os.path.join(ROOT, "data", k + ".js")):
            problems.append(f"INDEX   '{k}' is ready:true but data/{k}.js is missing")
    for name in sorted(os.listdir(os.path.join(ROOT, "data"))):
        if name.endswith(".js") and name not in ("index.js", "intro.js"):
            k = name[:-3]
            if k not in keys:
                problems.append(f"INDEX   data/{name} exists but '{k}' is not in the index")
```

- [ ] **Step 2: Run the gate to verify it fails**

Run: `python tools/check_release.py`
Expected: FAIL with a traceback on missing `data/index.js`.

- [ ] **Step 3: Create `data/index.js`**

Chronological order. `ready: false` entries render as disabled cards — the timeline shows its own shape from day one.

```js
/* Lightweight index: loaded at startup so the home screen opens instantly no
   matter how many prophets exist. Prophet data itself loads on demand. */
window.PROPHET_INDEX = [
  { key: "adam",    nameAr: "آدم",   nameEn: "", order: 1,  ready: false },
  { key: "nuh",     nameAr: "نوح",   nameEn: "", order: 2,  ready: true  },
  { key: "hud",     nameAr: "هود",   nameEn: "", order: 3,  ready: false },
  { key: "salih",   nameAr: "صالح",  nameEn: "", order: 4,  ready: false },
  { key: "ibrahim", nameAr: "إبراهيم", nameEn: "", order: 5, ready: false }
];
```

- [ ] **Step 4: Create `app/loader.js`**

```js
/* On-demand prophet loading by classic <script> injection.
   NOT fetch — fetch is blocked on file:// by CORS, and file:// must work. */
(function () {
  "use strict";

  var VERSION = "0.1.0";
  var pending = {};

  window.PROPHETS = window.PROPHETS || {};

  window.loadProphet = function (key, cb) {
    if (window.PROPHETS[key]) { cb(null, window.PROPHETS[key]); return; }

    if (pending[key]) { pending[key].push(cb); return; }
    pending[key] = [cb];

    function done(err) {
      var queue = pending[key];
      delete pending[key];
      for (var i = 0; i < queue.length; i++) {
        queue[i](err, window.PROPHETS[key] || null);
      }
    }

    var s = document.createElement("script");
    s.src = "data/" + key + ".js?v=" + VERSION;
    s.onload = function () {
      done(window.PROPHETS[key] ? null : new Error("loaded but empty: " + key));
    };
    s.onerror = function () { done(new Error("failed to load " + key)); };
    document.head.appendChild(s);
  };
})();
```

- [ ] **Step 5: Render the home list in `app/nav.js`**

Append inside the IIFE:

```js
  window.renderHome = function () {
    var list = document.getElementById("prophet-list");
    list.textContent = "";
    var items = window.PROPHET_INDEX.slice().sort(function (a, b) {
      return a.order - b.order;
    });
    for (var i = 0; i < items.length; i++) {
      (function (p) {
        var li = document.createElement("li");
        var btn = document.createElement("button");
        btn.className = "prophet-card" + (p.ready ? "" : " is-pending");
        btn.textContent = p.nameAr;
        btn.disabled = !p.ready;
        if (!p.ready) { btn.title = "قيد الإعداد"; }
        btn.addEventListener("click", function () { window.openProphet(p.key); });
        li.appendChild(btn);
        list.appendChild(li);
      })(items[i]);
    }
  };

  window.openProphet = function (key) {
    window.loadProphet(key, function (err, data) {
      if (err) {
        var body = document.getElementById("phase-body");
        body.textContent = "تعذّر تحميل القصة.";
        var retry = document.createElement("button");
        retry.textContent = "إعادة المحاولة";
        retry.addEventListener("click", function () { window.openProphet(key); });
        body.appendChild(retry);
        window.showScreen("prophet");
        return;
      }
      window.setState({ PROPHET: key, PHASE: 0 });
      document.getElementById("prophet-name").textContent = data.nameAr;
      window.showScreen("prophet");
    });
  };
```

- [ ] **Step 6: Add the scripts and the back button**

In `index.html`, add before `app/nav.js`:

```html
<script src="data/index.js?v=0.1.0"></script>
<script src="app/loader.js?v=0.1.0"></script>
```

In `app/main.js` `init()`, after the existing `intro-enter` listener:

```js
    window.renderHome();
    document.getElementById("btn-back-home").addEventListener("click", function () {
      window.showScreen("home");
    });
```

- [ ] **Step 7: Create a stub `data/nuh.js` so the gate can pass**

```js
window.PROPHETS = window.PROPHETS || {};
window.PROPHETS.nuh = {
  key: "nuh",
  order: 2,
  nameAr: "نوح",
  nameEn: "",
  titleAr: "أول رسول إلى أهل الأرض",
  phases: []
};
```

- [ ] **Step 8: Run the gate and verify in a browser**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

Open `index.html` from `file://`. Expected: ابدأ → home lists five prophets with only نوح enabled; clicking نوح loads `data/nuh.js` and shows the prophet screen with the name. Confirm in DevTools Network that `nuh.js` is requested **only on click**, not at startup.

- [ ] **Step 9: Commit**

```bash
git add index.html app/loader.js app/nav.js app/main.js data/index.js data/nuh.js tools/check_release.py
git commit -m "feat: prophet index and on-demand data loading"
```

---

### Task 4: The sourcing gate

This task writes the gate **before** any narration content exists, so no unsourced prose can ever be written in the first place.

**Files:**
- Create: `tools/check_sources.py`, `tools/fixtures/bad_link_digits.js`, `tools/fixtures/bad_unsourced.js`, `tools/fixtures/bad_blacklist.js`
- Modify: `tools/check_release.py`

**Interfaces:**
- Consumes: the beat/source schema.
- Produces: `python tools/check_sources.py [--file PATH]` — exits 0 clean, 1 listing violations. `check_release.py` calls it for every `data/*.js`.

- [ ] **Step 1: Write three fixtures that MUST be rejected**

`tools/fixtures/bad_unsourced.js` — a narr beat with no sources:

```js
window.PROPHETS = window.PROPHETS || {};
window.PROPHETS.fixture = { key: "fixture", order: 99, nameAr: "اختبار", nameEn: "", phases: [
  { id: "state", kindAr: "اختبار", titleAr: "ع", titleEn: "", dateAr: "ع", dateEn: "",
    ayah: "ع", ayahRef: "ع", ayahEn: "", ayahRefEn: "Surah Nuh (71), verse 1",
    mtAr: "ع", mtEn: "", mdAr: "ع", mdEn: "", amb: "dawn",
    beats: [ { kind: "narr", textAr: "جملة بلا إسناد.", textEn: "", focus: {x:1,y:1,scale:1}, srcRefs: [] } ],
    charsAr: [], charsEn: [], lessonsAr: ["ع"], lessonsEn: [],
    srcs: [ { ref: "صحيح البخاري (١)", gradeAr: "صحيح", classAr: "ثابت", url: "" } ] }
]};
```

`tools/fixtures/bad_link_digits.js` — identical, but the beats array is:

```js
    beats: [
      { kind: "narr", textAr: "جملة أولى مسندة.", textEn: "", focus: {x:1,y:1,scale:1}, srcRefs: [0] },
      { kind: "link", textAr: "فركب معه ثمانون رجلا.", textEn: "" },
      { kind: "narr", textAr: "جملة ثالثة مسندة.", textEn: "", focus: {x:1,y:1,scale:1}, srcRefs: [0] }
    ],
```

`tools/fixtures/bad_blacklist.js` — identical to `bad_unsourced.js` but with `srcRefs: [0]` on the beat and:

```js
    srcs: [ { ref: "عرائس المجالس للثعلبي (٢/٤)", gradeAr: "", classAr: "ثابت", url: "" } ]
```

- [ ] **Step 2: Write `tools/check_sources.py`**

```python
#!/usr/bin/env python3
"""The sourcing gate.

Isra'iliyyat are not detected — they are made structurally impossible. No
automated test can classify a narration semantically; a keyword list passes a
reworded report and rejects a text that cites one in order to refute it. So the
guarantee is that narration prose cannot exist without an established chain.
"""
import argparse, json, os, re, subprocess, sys

ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

ACCEPTED_CLASS = "ثابت"

# Layer 3: books and transmitters whose presence is disqualifying.
BLACKLIST = [
    "الثعلبي", "الكسائي", "عرائس المجالس",
    "كعب الأحبار", "وهب بن منبه",
    "التوراة", "سفر التكوين", "العهد القديم",
    "الكافي", "بحار الأنوار", "الصحيفة السجادية",
]

# Layer 4: motifs that historically carried Isra'iliyyat into these stories.
# These STOP for a human; they do not auto-reject, because the text may be
# citing them in order to refute them and a machine cannot tell.
REVIEW_FLAGS = ["أوريا", "خاتم سليمان", "ركاب السفينة", "طول آدم", "هاروت وماروت"]

LINK_MAX_CHARS = 80
DIGITS = re.compile(r"[0-9٠-٩]")
SENTENCE_END = re.compile(r"[.؟!:]\s*$")

problems = []
notices = []


def load(path):
    """Evaluate a data file in Node and return its prophet object as JSON."""
    shim = (
        "global.window={};require(%s);"
        "const P=window.PROPHETS||{};const k=Object.keys(P)[0];"
        "process.stdout.write(JSON.stringify(P[k]||null))"
    ) % json.dumps(os.path.abspath(path).replace("\\", "/"))
    out = subprocess.check_output(["node", "-e", shim], cwd=ROOT)
    return json.loads(out)


def check_phase(pkey, pi, ph):
    where = f"{pkey}[{pi}]"
    srcs = ph.get("srcs") or []

    # Layer 2 — only established sources exist at all.
    for si, s in enumerate(srcs):
        cls = s.get("classAr", "")
        if cls != ACCEPTED_CLASS:
            problems.append(f"CLASS   {where}.srcs[{si}] classAr={cls!r}, only {ACCEPTED_CLASS!r} is accepted")
        # Layer 3 — blacklist.
        for bad in BLACKLIST:
            if bad in s.get("ref", "") or bad in (s.get("url") or ""):
                problems.append(f"BLACK   {where}.srcs[{si}] cites {bad!r}")

    beats = ph.get("beats") or []
    if not beats:
        problems.append(f"BEATS   {where} has no beats")
        return

    for bi, b in enumerate(beats):
        kind = b.get("kind", "narr")
        text = b.get("textAr", "")
        at = f"{where}.beats[{bi}]"

        if not SENTENCE_END.search(text):
            problems.append(f"SENT    {at} does not end in . ؟ ! or : — cues need whole sentences")

        if kind == "narr":
            # Layer 1 — the decisive one.
            refs = b.get("srcRefs") or []
            if not refs:
                problems.append(f"UNSRC   {at} is narration with no srcRefs")
            for r in refs:
                if not isinstance(r, int) or r < 0 or r >= len(srcs):
                    problems.append(f"UNSRC   {at} srcRefs {r} is out of range")
        elif kind == "link":
            # The narrow exception. "It's just a connective" is exactly the
            # cover a report would wear, so constrain it mechanically.
            if len(text) > LINK_MAX_CHARS:
                problems.append(f"LINK    {at} is {len(text)} chars, max {LINK_MAX_CHARS}")
            if DIGITS.search(text):
                problems.append(f"LINK    {at} contains a digit — a number is always a report")
            if b.get("focus") or b.get("pin"):
                problems.append(f"LINK    {at} has focus/pin — a connective does not move the listener")
            if b.get("srcRefs"):
                problems.append(f"LINK    {at} has srcRefs — mark it kind:'narr' instead")
            if bi == 0 or bi == len(beats) - 1:
                problems.append(f"LINK    {at} is first or last — a phase opens and closes on sourced material")
            if bi > 0 and beats[bi - 1].get("kind") == "link":
                problems.append(f"LINK    {at} follows another link beat — links may not be chained")
        else:
            problems.append(f"KIND    {at} kind={kind!r}, expected 'narr' or 'link'")

        for flag in REVIEW_FLAGS:
            if flag in text:
                notices.append(f"REVIEW  {at} mentions {flag!r} — confirm it is cited to refute, not to report")


def check_file(path):
    data = load(path)
    if not data:
        problems.append(f"LOAD    {path} defines no prophet")
        return
    for pi, ph in enumerate(data.get("phases") or []):
        check_phase(data.get("key", "?"), pi, ph)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--file", action="append", help="data file(s) to check; default all of data/")
    args = ap.parse_args()

    files = args.file
    if not files:
        d = os.path.join(ROOT, "data")
        files = [os.path.join(d, n) for n in sorted(os.listdir(d))
                 if n.endswith(".js") and n not in ("index.js", "intro.js")]

    for f in files:
        check_file(f)

    for n in notices:
        print("  " + n)
    if problems:
        print(f"\n{len(problems)} sourcing problem(s):\n")
        for p in problems:
            print("  " + p)
        return 1
    print("check_sources: clean")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 3: Run the gate against each fixture to verify each is rejected**

```bash
python tools/check_sources.py --file tools/fixtures/bad_unsourced.js
python tools/check_sources.py --file tools/fixtures/bad_link_digits.js
python tools/check_sources.py --file tools/fixtures/bad_blacklist.js
```

Expected, in order — each must exit 1:
1. `UNSRC   fixture[0].beats[0] is narration with no srcRefs`
2. `LINK    fixture[0].beats[1] contains a digit — a number is always a report`
3. `BLACK   fixture[0].srcs[0] cites 'الثعلبي'`

If any fixture exits 0, the gate is broken — fix it before continuing. A gate that does not reject is worse than no gate, because it is trusted.

- [ ] **Step 4: Run it against the real (empty) `data/nuh.js`**

Run: `python tools/check_sources.py`
Expected: `check_sources: clean` — `nuh` has `phases: []`, so nothing to violate yet.

- [ ] **Step 5: Call it from the release gate**

Add to `tools/check_release.py`, and call from `main()`:

```python
def check_sources():
    r = subprocess.run([sys.executable, os.path.join(ROOT, "tools", "check_sources.py")],
                       capture_output=True, text=True, cwd=ROOT)
    if r.returncode != 0:
        for line in r.stdout.strip().splitlines():
            if line.strip():
                problems.append("SOURCE  " + line.strip())
```

- [ ] **Step 6: Verify the whole gate still passes, then commit**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add tools/
git commit -m "feat: sourcing gate rejecting unsourced narration"
```

---

### Task 5: Nuh phase 1 — the state of his people

**Files:**
- Modify: `data/nuh.js`

**Interfaces:**
- Consumes: the schema enforced by Task 4.
- Produces: `window.PROPHETS.nuh.phases[0]` — a complete phase object with sentence-aligned beats.

**Content rule for this task:** every `narr` beat traces to the Qur'an, the Sahihayn, or Ibn Kathir. The five idols are named in Surah Nuh 71:23 directly; their history as righteous men whose images were later venerated is in **Sahih al-Bukhari 4920**, on the authority of Ibn Abbas — that is a sound marfu' chain, not an Isra'iliyyah.

- [ ] **Step 1: Write the phase**

Replace the `phases: []` in `data/nuh.js` with:

```js
  phases: [
    {
      id: "state",
      kindAr: "حال القوم قبل البعثة",
      kindEn: "",
      titleAr: "قوم نوح وأول شرك في الأرض",
      titleEn: "",
      dateAr: "قبل الطوفان",
      dateEn: "",

      ayah: "وَقَالُوا لَا تَذَرُنَّ آلِهَتَكُمْ وَلَا تَذَرُنَّ وَدًّا وَلَا سُوَاعًا وَلَا يَغُوثَ وَيَعُوقَ وَنَسْرًا",
      ayahRef: "سورة نوح، الآية ٢٣",
      ayahEn: "",
      ayahRefEn: "Surah Nuh (71), verse 23",

      mtAr: "أرض قوم نوح",
      mtEn: "",
      mdAr: "أول مجتمع وقع فيه الشرك بعد التوحيد",
      mdEn: "",
      amb: "dawn",

      beats: [
        {
          kind: "narr",
          textAr: "كان الناس قبل نوح عليه السلام على التوحيد، أمة واحدة لا تعبد إلا الله. ثم وقع فيهم الشرك، فكان أول شرك عُبد به غير الله في الأرض.",
          textEn: "",
          focus: { x: 400, y: 300, scale: 1.0 },
          srcRefs: [0, 1]
        },
        {
          kind: "narr",
          textAr: "وكان في قومه رجال صالحون، فلما ماتوا صوّروا صورهم ليتذكروا بها العبادة ويجتهدوا اجتهادهم. فلم تُعبد في زمانهم.",
          textEn: "",
          focus: { x: 380, y: 280, scale: 1.2 },
          srcRefs: [1]
        },
        { kind: "link", textAr: "ثم طال الأمد.", textEn: "" },
        {
          kind: "narr",
          textAr: "فلما تطاول الزمن ونُسي العلم، عُبدت تلك الصور من دون الله. وأسماؤها ودّ وسواع ويغوث ويعوق ونسر، سماها الله في كتابه.",
          textEn: "",
          focus: { x: 420, y: 310, scale: 1.4 },
          pin: "idols",
          srcRefs: [0, 1]
        }
      ],

      charsAr: [
        { i: "🕌", n: "ودّ وسواع ويغوث ويعوق ونسر", r: "أسماء رجال صالحين صارت أصنامًا تُعبد" }
      ],
      charsEn: [],

      lessonsAr: [
        "الشرك لا يبدأ جحودًا، بل يبدأ غلوًّا في الصالحين ثم يستقر عبادةً بعد طول الزمن.",
        "الأمة تفقد دينها حين تفقد علماءها، فمع نسيان العلم استقرت عبادة الصور."
      ],
      lessonsEn: [],

      srcs: [
        {
          ref: "سورة نوح، الآية ٢٣",
          gradeAr: "قرآن",
          classAr: "ثابت",
          url: ""
        },
        {
          ref: "صحيح البخاري (٤٩٢٠) عن ابن عباس رضي الله عنهما",
          gradeAr: "صحيح",
          classAr: "ثابت",
          url: "https://www.islamweb.net/ar/fatwa/"
        }
      ]
    }
  ]
```

- [ ] **Step 2: Run the sourcing gate**

Run: `python tools/check_sources.py`
Expected: `check_sources: clean` — no REVIEW notices, because nothing here touches a flagged motif.

- [ ] **Step 3: Deliberately break it, to confirm the gate bites on real content**

Temporarily change the `link` beat text to `"ثم طال الأمد مائة عام."` and rerun.
Expected: `LINK    nuh[0].beats[2] contains a digit` — wait, there are no digits, only the Arabic word مائة. **This is the honest limit of the digit check: a spelled-out number passes it.** Instead change it to `"ثم طال الأمد ٥٠ عاما."` and confirm the digit check fires, then restore the original text.

Record that limit in `docs/BUGS.md` (create it):

```markdown
# Known limits and fixed bugs

## Sourcing gate

- The `link` beat digit check catches numerals (`٥٠`, `50`) but **not** numbers
  spelled as words (`مائة`, `ثمانون`). A link beat is capped at 80 characters
  and may not be chained, which limits what a spelled-out number can smuggle,
  but the reviewer is the real check here. Do not treat the gate as complete.
```

- [ ] **Step 4: Run the full gate and commit**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add data/nuh.js docs/BUGS.md
git commit -m "content: Nuh phase 1 — the state of his people"
```

---

### Task 6: Phase renderer

**Files:**
- Create: `app/render.js`
- Modify: `index.html`, `app/nav.js`, `app/main.js`

**Interfaces:**
- Consumes: a phase object (Task 5), `window.APP_STATE`.
- Produces: `window.renderPhase(prophet, phaseIndex)` — fills `#phase-body` with the ayah, narration beats (each `<p>` carrying `data-beat="<i>"` so Task 10 can highlight it), figure cards, lesson cards, and source cards.

- [ ] **Step 1: Write `app/render.js`**

```js
/* Phase rendering. Stateless: takes a phase object, produces DOM. */
(function () {
  "use strict";

  function el(tag, cls, text) {
    var n = document.createElement(tag);
    if (cls) { n.className = cls; }
    if (text !== undefined) { n.textContent = text; }
    return n;
  }

  function cards(container, title, items, build) {
    if (!items || !items.length) { return; }
    var sec = el("div", "cards");
    sec.appendChild(el("h3", "cards-title", title));
    var grid = el("div", "cards-grid");
    for (var i = 0; i < items.length; i++) {
      grid.appendChild(build(items[i]));
    }
    sec.appendChild(grid);
    container.appendChild(sec);
  }

  window.renderPhase = function (prophet, phaseIndex) {
    var ph = prophet.phases[phaseIndex];
    var body = document.getElementById("phase-body");
    body.textContent = "";
    if (!ph) { body.appendChild(el("p", "", "لا توجد مراحل بعد.")); return; }

    body.appendChild(el("p", "phase-kind", ph.kindAr));
    body.appendChild(el("h3", "phase-title", ph.titleAr));
    body.appendChild(el("p", "phase-date", ph.dateAr));

    var q = el("blockquote", "ayah");
    q.appendChild(el("span", "", ph.ayah));
    q.appendChild(el("cite", "", ph.ayahRef));
    body.appendChild(q);

    var narr = el("div", "narration");
    for (var i = 0; i < ph.beats.length; i++) {
      var p = el("p", "beat beat-" + (ph.beats[i].kind || "narr"), ph.beats[i].textAr);
      p.setAttribute("data-beat", String(i));
      narr.appendChild(p);
    }
    body.appendChild(narr);

    cards(body, "من عاصره", ph.charsAr, function (c) {
      var card = el("div", "card");
      card.appendChild(el("span", "card-icon", c.i));
      card.appendChild(el("strong", "card-name", c.n));
      card.appendChild(el("span", "card-role", c.r));
      return card;
    });

    cards(body, "العبر المستفادة", ph.lessonsAr, function (t) {
      return el("div", "card card-lesson", t);
    });

    cards(body, "المصادر", ph.srcs, function (s) {
      var card = el("div", "card card-src");
      card.appendChild(el("span", "src-ref", s.ref));
      if (s.gradeAr) { card.appendChild(el("span", "src-grade", s.gradeAr)); }
      return card;
    });
  };
})();
```

- [ ] **Step 2: Call it from `openProphet` in `app/nav.js`**

Replace the two lines after `window.setState({ PROPHET: key, PHASE: 0 });` with:

```js
      document.getElementById("prophet-name").textContent =
        data.nameAr + " — " + data.titleAr;
      window.renderPhase(data, 0);
      window.showScreen("prophet");
```

- [ ] **Step 3: Add the script and styles**

In `index.html`, before `app/main.js`:

```html
<script src="app/render.js?v=0.1.0"></script>
```

Append to `style.css`:

```css
.ayah { border-inline-start: 3px solid var(--gold); margin: 1.2rem 0; padding: 0 1rem; }
.ayah cite { display: block; color: var(--gold); font-size: .85em; font-style: normal; }
.beat { margin: .8rem 0; }
.beat-link { opacity: .75; font-style: italic; }
.beat.is-active { background: rgba(197, 160, 89, .12); border-radius: .4rem; }
.cards-grid { display: grid; gap: .6rem; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); }
.card { background: rgba(255,255,255,.04); border: 1px solid rgba(197,160,89,.25); border-radius: .5rem; padding: .7rem; }
.card-name { display: block; }
.src-grade { color: var(--gold); font-size: .8em; }
.prophet-card { min-height: 44px; min-width: 44px; }
.prophet-card.is-pending { opacity: .4; }
```

- [ ] **Step 4: Verify in a browser**

Open `index.html` from `file://`, click نوح.
Expected: ayah with its reference, four narration paragraphs (the link beat visibly lighter and italic), one figure card, two lesson cards, two source cards with «صحيح» on the Bukhari one.

- [ ] **Step 5: Run the gate and commit**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add index.html style.css app/render.js app/nav.js
git commit -m "feat: phase renderer with figure, lesson and source cards"
```

---

### Task 7: Narration generation with sentence cues

**Files:**
- Create: `tools/gen_tts.py`, `tools/narration_ar.json`
- Modify: `tools/check_release.py`

**Interfaces:**
- Consumes: `data/*.js` beats; `tools/narration_ar.json` keyed `<prophet>_<phase>_<beat>`.
- Produces: `audio/<slot>/<prophet>_<phase>_ar.mp3` and `audio/<slot>/<prophet>_<phase>_ar.cues.js` setting `window.CUES["<slot>/<prophet>_<phase>_ar"] = [seconds...]`, one entry per beat. Plus `audio/manifest.json`.

- [ ] **Step 1: Write the diacritized sidecar**

`tools/narration_ar.json` — keys are `<prophet>_<phase>_<beat>`. The consonant skeleton must match `textAr` exactly; only harakat are added. Target density is 80%+; anything under 40% is rejected as bare.

```json
{
  "_comment": "Build-time only. The app never loads this. On-screen text stays bare.",
  "nuh_0_0": "كَانَ النَّاسُ قَبْلَ نُوحٍ عَلَيْهِ السَّلَامُ عَلَى التَّوْحِيدِ، أُمَّةً وَاحِدَةً لَا تَعْبُدُ إِلَّا اللَّهَ. ثُمَّ وَقَعَ فِيهِمُ الشِّرْكُ، فَكَانَ أَوَّلَ شِرْكٍ عُبِدَ بِهِ غَيْرُ اللَّهِ فِي الْأَرْضِ.",
  "nuh_0_1": "وَكَانَ فِي قَوْمِهِ رِجَالٌ صَالِحُونَ، فَلَمَّا مَاتُوا صَوَّرُوا صُوَرَهُمْ لِيَتَذَكَّرُوا بِهَا الْعِبَادَةَ وَيَجْتَهِدُوا اجْتِهَادَهُمْ. فَلَمْ تُعْبَدْ فِي زَمَانِهِمْ.",
  "nuh_0_2": "ثُمَّ طَالَ الْأَمَدُ.",
  "nuh_0_3": "فَلَمَّا تَطَاوَلَ الزَّمَنُ وَنُسِيَ الْعِلْمُ، عُبِدَتْ تِلْكَ الصُّوَرُ مِنْ دُونِ اللَّهِ. وَأَسْمَاؤُهَا وَدٌّ وَسُوَاعٌ وَيَغُوثُ وَيَعُوقُ وَنَسْرٌ، سَمَّاهَا اللَّهُ فِي كِتَابِهِ."
}
```

- [ ] **Step 2: Write `tools/gen_tts.py`**

```python
#!/usr/bin/env python3
"""Generate narration MP3s and sentence-cue sidecars with edge-tts.

Cues come from SentenceBoundary events. Verified 2026-09-11 on edge-tts 7.2.8:
WordBoundary is NEVER emitted, for any voice or language. SentenceBoundary is,
one event per sentence with offset/duration/text — which is more precise for
beat boundaries than accumulating word counts would have been.
"""
import argparse, asyncio, json, os, re, subprocess, sys
import edge_tts

ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
VOC_PATH = os.path.join(ROOT, "tools", "narration_ar.json")
AUDIO = os.path.join(ROOT, "audio")

SLOTS = {
    "shakir": {"ar": "ar-EG-ShakirNeural", "labelAr": "شاكر", "generate": True},
    "story":  {"ar": "ar-EG-SalmaNeural",  "labelAr": "سلمى", "generate": True},
    "classic": {"ar": "ar-SA-HamedNeural", "labelAr": "حامد", "generate": False},
}
RATE = "-8%"
CONCURRENCY = 4  # be gentle on the free endpoint

SENT_SPLIT = re.compile(r"(?<=[.؟!:])\s+")


def load_prophets():
    """{ key: prophetObject } for every data/*.js except index and intro."""
    d = os.path.join(ROOT, "data")
    files = [n for n in sorted(os.listdir(d))
             if n.endswith(".js") and n not in ("index.js", "intro.js")]
    reqs = ";".join("require('./data/%s')" % n for n in files)
    shim = f"global.window={{}};{reqs};process.stdout.write(JSON.stringify(window.PROPHETS||{{}}))"
    return json.loads(subprocess.check_output(["node", "-e", shim], cwd=ROOT))


def load_voc():
    with open(VOC_PATH, encoding="utf-8") as f:
        return {k: v for k, v in json.load(f).items() if k != "_comment"}


def sentences(text):
    return [s for s in SENT_SPLIT.split(text.strip()) if s]


def phase_text(pkey, pi, phase, voc):
    """Returns (full_text, beat_sentence_counts). Prefers the vocalized sidecar
    per beat; falls back to bare textAr with a warning."""
    parts, counts = [], []
    for bi, b in enumerate(phase["beats"]):
        key = f"{pkey}_{pi}_{bi}"
        t = voc.get(key)
        if not t:
            print(f"  WARN no vocalization for {key} — TTS will guess the vowels")
            t = b["textAr"]
        parts.append(t.strip())
        counts.append(len(sentences(t)))
    return " ".join(parts), counts


async def synth(sem, voice, text, counts, mp3_path, cues_path, cue_key, force):
    if os.path.exists(mp3_path) and os.path.getsize(mp3_path) > 0 and not force:
        return "skip"
    async with sem:
        comm = edge_tts.Communicate(text, voice, rate=RATE)
        audio = bytearray()
        starts = []
        async for ch in comm.stream():
            if ch["type"] == "audio":
                audio += ch["data"]
            elif ch["type"] == "SentenceBoundary":
                starts.append(ch["offset"] / 1e7)

    total = sum(counts)
    if len(starts) != total:
        return f"MISMATCH {len(starts)} sentence events != {total} sentences in beats"
    if not audio:
        return "EMPTY audio"

    # A beat's cue is its first sentence's offset.
    cues, at = [], 0
    for c in counts:
        cues.append(round(starts[at], 2))
        at += c

    os.makedirs(os.path.dirname(mp3_path), exist_ok=True)
    with open(mp3_path, "wb") as f:
        f.write(audio)
    with open(cues_path, "w", encoding="utf-8", newline="\n") as f:
        f.write("window.CUES = window.CUES || {};\n")
        f.write(f'window.CUES[{json.dumps(cue_key, ensure_ascii=False)}] = {json.dumps(cues)};\n')
    return "ok"


async def main_async(args):
    prophets = load_prophets()
    voc = load_voc()
    slots = args.slots or [s for s in SLOTS if SLOTS[s]["generate"]]
    sem = asyncio.Semaphore(CONCURRENCY)
    jobs, labels = [], []

    for pkey, p in prophets.items():
        if args.prophet and pkey not in args.prophet:
            continue
        for pi, phase in enumerate(p.get("phases") or []):
            text, counts = phase_text(pkey, pi, phase, voc)
            for slot in slots:
                base = f"{pkey}_{pi}_ar"
                mp3 = os.path.join(AUDIO, slot, base + ".mp3")
                cues = os.path.join(AUDIO, slot, base + ".cues.js")
                jobs.append(synth(sem, SLOTS[slot]["ar"], text, counts,
                                  mp3, cues, f"{slot}/{base}", args.force))
                labels.append(f"{slot}/{base}")

    results = await asyncio.gather(*jobs)
    bad = 0
    for label, r in zip(labels, results):
        if r not in ("ok", "skip"):
            print(f"  FAIL {label}: {r}")
            bad += 1
        else:
            print(f"  {r:4} {label}")

    manifest = {"slots": {s: {"ar": SLOTS[s]["ar"], "labelAr": SLOTS[s]["labelAr"]}
                          for s in slots}}
    os.makedirs(AUDIO, exist_ok=True)
    with open(os.path.join(AUDIO, "manifest.json"), "w", encoding="utf-8", newline="\n") as f:
        json.dump(manifest, f, ensure_ascii=False, indent=2)

    return 1 if bad else 0


def main():
    ap = argparse.ArgumentParser(description="Generate narration MP3s + sentence cues.")
    ap.add_argument("--prophet", action="append", help="subset of prophet keys")
    ap.add_argument("--slots", nargs="*", choices=list(SLOTS), help="subset of voice slots")
    ap.add_argument("--force", action="store_true", help="regenerate even if the file exists")
    return asyncio.run(main_async(ap.parse_args()))


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 3: Generate Nuh phase 1 for both voices**

Run: `python tools/gen_tts.py --prophet nuh`
Expected:

```
  ok   shakir/nuh_0_ar
  ok   story/nuh_0_ar
```

If you see `MISMATCH n != m`, the sidecar's sentence count differs from the beats' — a punctuation mark was dropped or added when vocalizing. Fix the sidecar; do not adjust the tolerance.

- [ ] **Step 4: Inspect the generated cues**

Run: `cat audio/shakir/nuh_0_ar.cues.js`
Expected: four numbers, ascending, starting near `0.1`, e.g.
`window.CUES["shakir/nuh_0_ar"] = [0.1, 11.3, 19.8, 22.4];`

Four entries for four beats. If there are three, a beat produced no sentence.

- [ ] **Step 5: Add audio coverage to the release gate**

```python
GEN_SLOTS = ["shakir", "story"]


def check_audio_coverage():
    """Every phase x generated slot needs a non-empty MP3 and a cues file whose
    entry count equals that phase's beat count. In ../Sera a new voice slot
    shipped missing a whole era, and failed generations left 0-byte MP3s that
    the 'exists' check happily skipped."""
    d = os.path.join(ROOT, "data")
    for name in sorted(os.listdir(d)):
        if not name.endswith(".js") or name in ("index.js", "intro.js"):
            continue
        key = name[:-3]
        shim = ("global.window={};require('./data/%s');"
                "const p=window.PROPHETS['%s'];"
                "process.stdout.write(JSON.stringify((p.phases||[]).map(x=>x.beats.length)))" % (name, key))
        counts = json.loads(subprocess.check_output(["node", "-e", shim], cwd=ROOT))
        for pi, nbeats in enumerate(counts):
            for slot in GEN_SLOTS:
                base = f"{key}_{pi}_ar"
                mp3 = os.path.join(ROOT, "audio", slot, base + ".mp3")
                cue = os.path.join(ROOT, "audio", slot, base + ".cues.js")
                if not os.path.exists(mp3) or os.path.getsize(mp3) == 0:
                    problems.append(f"AUDIO   {slot}/{base}.mp3 missing or empty")
                if not os.path.exists(cue):
                    problems.append(f"AUDIO   {slot}/{base}.cues.js missing")
                    continue
                got = len(json.loads(re.search(r"=\s*(\[[^\]]*\]);", read(f"audio/{slot}/{base}.cues.js")).group(1)))
                if got != nbeats:
                    problems.append(f"AUDIO   {slot}/{base}.cues.js has {got} cues, {nbeats} beats")
```

Add `import json` to the top of `check_release.py` and call `check_audio_coverage()` from `main()`.

- [ ] **Step 6: Run the gate and commit the audio**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add tools/gen_tts.py tools/narration_ar.json tools/check_release.py audio/
git commit -m "feat: narration generation with SentenceBoundary cues"
```

---

### Task 8: Vocalization gate

**Files:**
- Create: `tools/check_voc.py`
- Modify: `tools/check_release.py`

**Interfaces:**
- Consumes: `data/*.js` beats and `tools/narration_ar.json`.
- Produces: `python tools/check_voc.py` — exits 0 clean, 1 listing MISSING / EXTRA / MISMATCH / BARE / FAKE / FUSION / FOREIGN.

- [ ] **Step 1: Write `tools/check_voc.py`**

Ported from `../Sera/tools/check_voc.py`, rekeyed to beats. Every check exists because its failure mode shipped live.

```python
#!/usr/bin/env python3
"""Verify the diacritized narration sidecar.

Density alone proves NOTHING: 28 Ottoman entries in ../Sera scored 91-96% while
being mechanically fatha-stamped garbage. Hence the FAKE and FUSION checks.
"""
import json, os, re, subprocess, sys

ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
VOC_PATH = os.path.join(ROOT, "tools", "narration_ar.json")

_HARAKAT = re.compile(r"[ؐ-ًؚ-ٰٟۖ-ۭـ]")
_AR_LETTER = re.compile(r"[ء-ي]")
_VOWEL_MARKS = re.compile(r"[ً-ْٰ]")
_MARKS_ALL = re.compile(r"[ً-ْ]")
_ALIF_MARKED = re.compile(r"ا[َُِّْ]")
_FOREIGN = re.compile(r"[^؀-ۿݐ-ݿﭐ-﷿ﹰ-﻿"
                      r"0-9\s\.,;:!\?\'\"«»\(\)\[\]\{\}—–\-_%&\*\+=/\\#@٪؟،؛]")

MIN_DENSITY = 40.0   # below this the entry is BARE, not vocalized
problems = []


def skeleton(s):
    s = _HARAKAT.sub("", s)
    s = re.sub(r"[آأإٱ]", "ا", s)
    s = s.replace("ؤ", "و").replace("ئ", "ي")
    s = s.replace("ى", "ي").replace("ء", "")
    return re.sub(r"[^ا-ي0-9٠-٩]", "", s)


def density(s):
    letters = len(_AR_LETTER.findall(s))
    return 100.0 if not letters else 100.0 * len(_VOWEL_MARKS.findall(s)) / letters


def fake_signals(s):
    """Mechanical stamping: almost all marks are fatha, with no sukun or shadda,
    and fathas land on plain alif where they never belong."""
    marks = _MARKS_ALL.findall(s)
    if len(marks) < 20:
        return None
    fatha = marks.count("َ")
    sukun = marks.count("ْ")
    shadda = marks.count("ّ")
    if fatha / len(marks) > 0.85 and (sukun + shadda) / len(marks) < 0.03:
        return f"{100*fatha//len(marks)}% fatha, {sukun} sukun, {shadda} shadda"
    if len(_ALIF_MARKED.findall(s)) > 3:
        return f"{len(_ALIF_MARKED.findall(s))} marked plain-alifs"
    return None


def load_beats():
    """{ '<prophet>_<phase>_<beat>': textAr }"""
    d = os.path.join(ROOT, "data")
    files = [n for n in sorted(os.listdir(d))
             if n.endswith(".js") and n not in ("index.js", "intro.js")]
    reqs = ";".join("require('./data/%s')" % n for n in files)
    shim = (f"global.window={{}};{reqs};const P=window.PROPHETS||{{}};const o={{}};"
            "for(const k of Object.keys(P))(P[k].phases||[]).forEach((ph,i)=>"
            "ph.beats.forEach((b,j)=>o[k+'_'+i+'_'+j]=b.textAr||''));"
            "process.stdout.write(JSON.stringify(o))")
    return json.loads(subprocess.check_output(["node", "-e", shim], cwd=ROOT))


def main():
    beats = load_beats()
    with open(VOC_PATH, encoding="utf-8") as f:
        voc = {k: v for k, v in json.load(f).items() if k != "_comment"}

    for k in sorted(set(voc) - set(beats)):
        problems.append(f"EXTRA    {k} in sidecar but no such beat — stale after an edit?")

    for k in sorted(beats):
        bare = beats[k]
        for ch in sorted(_FOREIGN.findall(bare)):
            problems.append(f"FOREIGN  {k} on-screen text contains {ch!r}")
        if k not in voc:
            problems.append(f"MISSING  {k} has no vocalization — TTS will guess every vowel")
            continue
        v = voc[k]
        if skeleton(v) != skeleton(bare):
            problems.append(f"MISMATCH {k} consonant skeleton differs from textAr")
        d = density(v)
        if d < MIN_DENSITY:
            problems.append(f"BARE     {k} density {d:.1f}% — not actually vocalized")
        fake = fake_signals(v)
        if fake:
            problems.append(f"FAKE     {k} mechanically stamped: {fake}")
        for ch in sorted(_FOREIGN.findall(v)):
            problems.append(f"FOREIGN  {k} sidecar contains {ch!r}")
        if re.search(r"[ء-ي]{28,}", _HARAKAT.sub("", v)):
            problems.append(f"FUSION   {k} has a 28+ letter run — a separator was dropped")

    if problems:
        print(f"\n{len(problems)} vocalization problem(s):\n")
        for p in problems:
            print("  " + p)
        return 1
    print("check_voc: clean")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 2: Run it against the real sidecar**

Run: `python tools/check_voc.py`
Expected: `check_voc: clean`

- [ ] **Step 3: Prove each gate bites**

Test all three failure modes, restoring the sidecar after each:

1. Copy `nuh_0_2`'s bare `textAr` (`ثم طال الأمد.`) over its vocalized value → expect `BARE nuh_0_2 density ...%`.
2. Change one letter in `nuh_0_0`'s sidecar (e.g. `كَانَ` → `كَالَ`) → expect `MISMATCH nuh_0_0`.
3. Delete the `nuh_0_1` key → expect `MISSING nuh_0_1`.

If any of these passes silently the gate is broken; fix it before continuing.

- [ ] **Step 4: Call it from the release gate**

Add to `check_release.py`, mirroring `check_sources`:

```python
def check_voc():
    r = subprocess.run([sys.executable, os.path.join(ROOT, "tools", "check_voc.py")],
                       capture_output=True, text=True, cwd=ROOT)
    if r.returncode != 0:
        for line in r.stdout.strip().splitlines():
            if line.strip():
                problems.append("VOC     " + line.strip())
```

- [ ] **Step 5: Run the gate and commit**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add tools/check_voc.py tools/check_release.py
git commit -m "feat: vocalization gate catching bare, fake and fused sidecars"
```

---

### Task 9: Narration playback and voice picker

**Files:**
- Create: `app/audio.js`
- Modify: `index.html`, `app/render.js`, `app/main.js`

**Interfaces:**
- Consumes: `window.APP_STATE.VOICE`, the generated MP3s.
- Produces: `window.playPhase(prophetKey, phaseIndex)` returns the `HTMLAudioElement` (or `null` if unavailable), `window.stopPhase()`, and `window.onPhaseAudio(fn)` which calls `fn(audioEl, cueKey)` whenever playback starts — Task 10 uses this to attach map sync.

- [ ] **Step 1: Write `app/audio.js`**

```js
/* Narration playback. Pre-recorded only — NEVER speechSynthesis. It was removed
   from ../Sera for sounding robotic and ignoring the chosen voice. */
(function () {
  "use strict";

  var current = null;
  var listeners = [];

  window.onPhaseAudio = function (fn) { listeners.push(fn); };

  function notice(msg) {
    var n = document.getElementById("audio-notice");
    if (n) { n.textContent = msg; n.hidden = !msg; }
  }

  window.stopPhase = function () {
    if (current) { current.pause(); current.src = ""; current = null; }
  };

  window.playPhase = function (prophetKey, phaseIndex) {
    window.stopPhase();
    notice("");

    var slot = window.APP_STATE.VOICE;
    var base = prophetKey + "_" + phaseIndex + "_ar";
    var cueKey = slot + "/" + base;

    var a = new Audio("audio/" + slot + "/" + base + ".mp3");
    a.addEventListener("error", function () {
      notice("الصوت غير متاح");
      current = null;
    });

    current = a;
    for (var i = 0; i < listeners.length; i++) { listeners[i](a, cueKey); }
    a.play().catch(function () { notice("الصوت غير متاح"); });
    return a;
  };
})();
```

- [ ] **Step 2: Load every cues file for the phase**

Cues are `.js`, so they inject like data. Append to `app/loader.js` inside the IIFE:

```js
  var cuesLoaded = {};

  window.loadCues = function (cueKey, cb) {
    window.CUES = window.CUES || {};
    if (window.CUES[cueKey] || cuesLoaded[cueKey]) { cb(window.CUES[cueKey] || null); return; }
    cuesLoaded[cueKey] = true;
    var s = document.createElement("script");
    s.src = "audio/" + cueKey + ".cues.js?v=" + VERSION;
    s.onload = function () { cb(window.CUES[cueKey] || null); };
    s.onerror = function () { cb(null); };
    document.head.appendChild(s);
  };
```

- [ ] **Step 3: Add the controls to `index.html`**

Inside `#prophet-screen`, after `#prophet-name`:

```html
<div class="controls">
  <button id="btn-play" data-ar="استمع" data-en="Listen">استمع</button>
  <label for="voice-pick" data-ar="الصوت" data-en="Voice">الصوت</label>
  <select id="voice-pick">
    <option value="shakir">شاكر</option>
    <option value="story">سلمى</option>
  </select>
  <p id="audio-notice" hidden></p>
</div>
```

Add before `app/main.js`:

```html
<script src="app/audio.js?v=0.1.0"></script>
```

- [ ] **Step 4: Wire the controls in `app/main.js` `init()`**

```js
    document.getElementById("btn-play").addEventListener("click", function () {
      var s = window.APP_STATE;
      if (s.PROPHET) { window.playPhase(s.PROPHET, s.PHASE); }
    });
    document.getElementById("voice-pick").addEventListener("change", function (e) {
      window.setState({ VOICE: e.target.value });
      window.stopPhase();
    });
```

- [ ] **Step 5: Verify in a browser, including the failure path**

Serve it (`npm start`) and open the local URL. Click نوح → استمع.
Expected: Shakir narrates the phase. Switch to سلمى and press استمع: a different voice.

Then rename `audio/shakir/nuh_0_ar.mp3` temporarily and press استمع.
Expected: «الصوت غير متاح» appears and **nothing speaks** — no synthetic fallback. Restore the file.

- [ ] **Step 6: Run the gate and commit**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add index.html app/audio.js app/loader.js app/main.js
git commit -m "feat: narration playback with voice picker"
```

---

### Task 10: Map and audio-driven focus

**Files:**
- Create: `app/mapsync.js`
- Modify: `index.html`, `style.css`, `app/main.js`

**Interfaces:**
- Consumes: `window.onPhaseAudio` (Task 9), `window.loadCues` (Task 9), the phase's `beats[].focus`.
- Produces: `window.attachMapSync()` — registered once at init; advances `#svg-nuh`'s `viewBox` and highlights the active `[data-beat]` paragraph as the narration crosses each cue.

- [ ] **Step 1: Add the inline SVG map**

Inside `#prophet-screen`, after the controls. Every `<text>` needs both `data-ar` and `data-en` or the bilingual gate fails.

```html
<div class="map-wrap">
  <svg id="svg-nuh" class="map-svg" viewBox="0 0 800 600" xmlns="http://www.w3.org/2000/svg">
    <rect width="800" height="600" fill="#0a2620"/>
    <path d="M120 380 C260 300 420 320 560 260 C660 220 720 240 760 210"
          fill="none" stroke="#3c6f62" stroke-width="26" stroke-linecap="round"/>
    <circle cx="400" cy="300" r="10" fill="#c5a059"/>
    <text x="400" y="278" fill="#f2ece0" font-size="20" text-anchor="middle"
          data-ar="أرض قوم نوح" data-en="The land of Nuh's people">أرض قوم نوح</text>
    <circle id="map-marker" cx="400" cy="300" r="16" fill="none"
            stroke="#c5a059" stroke-width="3" opacity="0"/>
  </svg>
</div>
```

- [ ] **Step 2: Write `app/mapsync.js`**

```js
/* Drives the map focus from narration cues.
   Uses 'timeupdate', NOT requestAnimationFrame — browsers pause rAF in hidden
   tabs, which is how an imam map in ../Sera never positioned itself. */
(function () {
  "use strict";

  var BASE = { w: 800, h: 600 };

  function applyFocus(svg, focus) {
    if (!svg || !focus) { return; }
    var scale = focus.scale || 1;
    var w = BASE.w / scale;
    var h = BASE.h / scale;
    svg.setAttribute("viewBox",
      (focus.x - w / 2) + " " + (focus.y - h / 2) + " " + w + " " + h);
    var marker = document.getElementById("map-marker");
    if (marker) {
      marker.setAttribute("cx", focus.x);
      marker.setAttribute("cy", focus.y);
      marker.setAttribute("opacity", "1");
    }
  }

  function highlight(index) {
    var all = document.querySelectorAll("#phase-body [data-beat]");
    for (var i = 0; i < all.length; i++) {
      all[i].classList.toggle("is-active", Number(all[i].getAttribute("data-beat")) === index);
    }
  }

  window.attachMapSync = function () {
    window.onPhaseAudio(function (audio, cueKey) {
      var state = window.APP_STATE;
      var prophet = window.PROPHETS[state.PROPHET];
      if (!prophet) { return; }
      var phase = prophet.phases[state.PHASE];
      var svg = document.getElementById("svg-" + state.PROPHET);

      window.loadCues(cueKey, function (cues) {
        var last = -1;

        // Degrade gracefully: no cues means a static map, not a broken page.
        if (!cues) {
          applyFocus(svg, phase.beats[0].focus);
          return;
        }

        audio.addEventListener("timeupdate", function () {
          var t = audio.currentTime;
          var i = 0;
          while (i + 1 < cues.length && cues[i + 1] <= t) { i++; }
          if (i === last) { return; }
          last = i;
          highlight(i);
          // link beats carry no focus — keep the previous one.
          for (var j = i; j >= 0; j--) {
            if (phase.beats[j] && phase.beats[j].focus) {
              applyFocus(svg, phase.beats[j].focus);
              break;
            }
          }
        });

        audio.addEventListener("ended", function () { highlight(-1); });
      });
    });
  };
})();
```

- [ ] **Step 3: Register it and add styles**

In `index.html` before `app/main.js`:

```html
<script src="app/mapsync.js?v=0.1.0"></script>
```

In `app/main.js` `init()`, add `window.attachMapSync();` as the last line.

Append to `style.css`:

```css
.map-wrap { margin: 1rem 0; }
.map-svg { width: 100%; max-height: 50vh; display: block; transition: none; }
#map-marker { transition: cx .4s ease, cy .4s ease; }
@media (max-width: 720px) { .map-svg { max-height: 40vh; } }
```

- [ ] **Step 4: Verify the sync in a browser**

Serve and open. Click نوح → استمع, and watch through the whole clip.
Expected: the first paragraph highlights immediately; at roughly 11s the second highlights and the map zooms slightly; the link beat highlights without moving the map; the final beat zooms to the idols focus. Highlight clears at the end.

Confirm the cue file loaded once in the Network tab, and that switching to سلمى loads a *different* cues file with different numbers.

- [ ] **Step 5: Run the gate and commit**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add index.html style.css app/mapsync.js app/main.js
git commit -m "feat: audio-driven map focus and beat highlighting"
```

---

### Task 11: Verse recitation

**Files:**
- Create: `app/recite.js`
- Modify: `index.html`, `app/main.js`, `tools/check_release.py`

**Interfaces:**
- Consumes: `phase.ayahRefEn`.
- Produces: `window.parseAyahRef(refEn)` → `{ surah, ayah }` or `null`; `window.reciteURL(ref, reciter)` → the everyayah URL; `window.playRecitation(refEn)` plays it or shows a notice.

- [ ] **Step 1: Add the ayah-reference check to the gate**

A non-parseable reference means a phase silently loses its recitation button. This shipped in `../Sera` as generically-paired verses.

```python
AYAH_RE = re.compile(r"Surah\s+.+?\((\d+)\),\s+verses?\s+(\d+)")


def check_ayah_refs():
    d = os.path.join(ROOT, "data")
    for name in sorted(os.listdir(d)):
        if not name.endswith(".js") or name in ("index.js", "intro.js"):
            continue
        key = name[:-3]
        shim = ("global.window={};require('./data/%s');"
                "process.stdout.write(JSON.stringify((window.PROPHETS['%s'].phases||[])"
                ".map(p=>p.ayahRefEn||'')))" % (name, key))
        refs = json.loads(subprocess.check_output(["node", "-e", shim], cwd=ROOT))
        for i, r in enumerate(refs):
            m = AYAH_RE.search(r or "")
            if not m:
                problems.append(f"AYAH    {key}[{i}] ayahRefEn {r!r} is not parseable")
            elif not (1 <= int(m.group(1)) <= 114):
                problems.append(f"AYAH    {key}[{i}] surah {m.group(1)} out of range")
```

- [ ] **Step 2: Run the gate to verify it passes on real data**

Run: `python tools/check_release.py`
Expected: clean — `"Surah Nuh (71), verse 23"` parses to surah 71, ayah 23.

Then temporarily change `ayahRefEn` to `"Bukhari 4920"` and rerun.
Expected: `AYAH    nuh[0] ayahRefEn 'Bukhari 4920' is not parseable`. Restore it.

- [ ] **Step 3: Write `app/recite.js`**

```js
/* Verse recitation, streamed from everyayah.com.
   Needs network — unlike narration, which is committed. If it fails, narration
   keeps working and recitation simply goes silent. */
(function () {
  "use strict";

  var RE = /Surah\s+.+?\((\d+)\),\s+verses?\s+(\d+)/;
  var current = null;

  window.parseAyahRef = function (refEn) {
    var m = RE.exec(refEn || "");
    if (!m) { return null; }
    return { surah: Number(m[1]), ayah: Number(m[2]) };
  };

  function pad(n) { return ("00" + n).slice(-3); }

  window.reciteURL = function (ref, reciter) {
    return "https://everyayah.com/data/" + reciter + "/" +
           pad(ref.surah) + pad(ref.ayah) + ".mp3";
  };

  window.stopRecitation = function () {
    if (current) { current.pause(); current.src = ""; current = null; }
  };

  window.playRecitation = function (refEn) {
    window.stopRecitation();
    var notice = document.getElementById("recite-notice");
    var ref = window.parseAyahRef(refEn);
    if (!ref) { notice.textContent = "لا تلاوة لهذه المرحلة"; notice.hidden = false; return; }

    notice.hidden = true;
    var a = new Audio(window.reciteURL(ref, window.APP_STATE.RECITER));
    a.addEventListener("error", function () {
      notice.textContent = "تعذّر تحميل التلاوة — تحقق من الاتصال";
      notice.hidden = false;
      current = null;
    });
    current = a;
    a.play().catch(function () {
      notice.textContent = "تعذّر تحميل التلاوة — تحقق من الاتصال";
      notice.hidden = false;
    });
  };
})();
```

- [ ] **Step 4: Add the controls and wire them**

Inside the `.controls` div in `index.html`:

```html
<button id="btn-recite" data-ar="تلاوة الآية" data-en="Recite the verse">تلاوة الآية</button>
<label for="reciter-pick" data-ar="القارئ" data-en="Reciter">القارئ</label>
<select id="reciter-pick">
  <option value="Husary_128kbps">الحصري</option>
  <option value="Alafasy_128kbps">العفاسي</option>
  <option value="Abdul_Basit_Murattal_192kbps">عبد الباسط</option>
</select>
<p id="recite-notice" hidden></p>
```

Add `<script src="app/recite.js?v=0.1.0"></script>` before `app/main.js`, and in `init()`:

```js
    document.getElementById("btn-recite").addEventListener("click", function () {
      var s = window.APP_STATE;
      var p = window.PROPHETS[s.PROPHET];
      if (p) { window.playRecitation(p.phases[s.PHASE].ayahRefEn); }
    });
    document.getElementById("reciter-pick").addEventListener("change", function (e) {
      window.setState({ RECITER: e.target.value });
      window.stopRecitation();
    });
```

- [ ] **Step 5: Verify both paths**

Serve and open. Click نوح → تلاوة الآية.
Expected: al-Husary recites Surah Nuh 71:23.

Then set DevTools Network to Offline and press it again.
Expected: «تعذّر تحميل التلاوة — تحقق من الاتصال». Press استمع while still offline — narration **still plays**, because it is committed. That contrast is the design working.

- [ ] **Step 6: Run the gate and commit**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add index.html app/recite.js app/main.js tools/check_release.py
git commit -m "feat: verse recitation with al-Husary default"
```

---

### Task 12: Remaining phases of Nuh, and phase navigation

**Files:**
- Modify: `data/nuh.js`, `tools/narration_ar.json`, `index.html`, `app/nav.js`, `app/main.js`

**Interfaces:**
- Consumes: everything above.
- Produces: `window.goPhase(delta)` — moves `APP_STATE.PHASE`, re-renders, stops audio.

- [ ] **Step 1: Write phases 2–5**

Append to `phases` in `data/nuh.js`, following the exact shape of phase 1. Each needs whole-sentence beats, `srcRefs` on every `narr` beat, and `classAr: "ثابت"` on every source.

| # | `id` | `titleAr` | Anchor verse | Core sources |
|---|---|---|---|---|
| 1 | `dawah` | الدعوة ألف سنة إلا خمسين عامًا | نوح ٧١:٥–٩ | القرآن، ابن كثير |
| 2 | `rejection` | تكذيب القوم واستكبارهم | هود ١١:٢٧ | القرآن، ابن كثير |
| 3 | `ark` | أمر الله ببناء السفينة | هود ١١:٣٧–٣٨ | القرآن، ابن كثير |
| 4 | `flood` | الطوفان ونجاة المؤمنين | هود ١١:٤٢–٤٣ | القرآن، ابن كثير |

**Content discipline for this task — do not skip:** the ark's dimensions, the number and names of its passengers, and the identity of Nuh's drowned son are **classic Isra'iliyyat entry points**. The Qur'an gives none of them. Write only what the verses and Ibn Kathir's graded material state. `check_sources.py` will raise a REVIEW notice if «ركاب السفينة» appears — treat that as a stop, not a formality.

- [ ] **Step 2: Run the sourcing gate before generating any audio**

Run: `python tools/check_sources.py`
Expected: `check_sources: clean`. Fix every violation before spending generation time on text that will change.

- [ ] **Step 3: Add the vocalized sidecar entries**

Add `nuh_1_0` … `nuh_4_n` to `tools/narration_ar.json`, one per beat, keeping each skeleton identical to its `textAr`.

Run: `python tools/check_voc.py`
Expected: `check_voc: clean`. A `MISSING` line means a beat has no entry; a `MISMATCH` means the skeleton drifted.

- [ ] **Step 4: Generate the audio**

Run: `python tools/gen_tts.py --prophet nuh`
Expected: `ok` for every `shakir/nuh_<i>_ar` and `story/nuh_<i>_ar`. Any `MISMATCH n != m` means a sidecar entry's sentence count differs from its beat's — fix the punctuation, do not loosen the check.

- [ ] **Step 5: Add phase navigation**

In `index.html`, inside `#prophet-screen` after `#phase-body`:

```html
<nav class="phase-nav">
  <button id="btn-prev" data-ar="السابق" data-en="Previous">السابق</button>
  <span id="phase-counter"></span>
  <button id="btn-next" data-ar="التالي" data-en="Next">التالي</button>
</nav>
```

Append to `app/nav.js` inside the IIFE:

```js
  window.goPhase = function (delta) {
    var s = window.APP_STATE;
    var p = window.PROPHETS[s.PROPHET];
    if (!p) { return; }
    var next = s.PHASE + delta;
    if (next < 0 || next >= p.phases.length) { return; }
    window.stopPhase();
    window.stopRecitation();
    window.setState({ PHASE: next });
    window.renderPhase(p, next);
    window.updatePhaseNav();
  };

  window.updatePhaseNav = function () {
    var s = window.APP_STATE;
    var p = window.PROPHETS[s.PROPHET];
    if (!p) { return; }
    document.getElementById("phase-counter").textContent =
      (s.PHASE + 1) + " / " + p.phases.length;
    document.getElementById("btn-prev").disabled = s.PHASE === 0;
    document.getElementById("btn-next").disabled = s.PHASE === p.phases.length - 1;
  };
```

Call `window.updatePhaseNav();` at the end of `openProphet`'s success path, and wire the buttons in `app/main.js` `init()`:

```js
    document.getElementById("btn-prev").addEventListener("click", function () { window.goPhase(-1); });
    document.getElementById("btn-next").addEventListener("click", function () { window.goPhase(1); });
```

- [ ] **Step 6: Verify the full story end to end**

Serve and open. Walk all five phases: listen to each, watch the map track, play a recitation, go back and forward.
Expected: the counter reads `1 / 5` … `5 / 5`; السابق is disabled on the first and التالي on the last; changing phase stops any playing audio rather than layering two voices.

- [ ] **Step 7: Run the gate and commit**

Run: `python tools/check_release.py`
Expected: `check_release: clean`

```bash
git add data/nuh.js tools/narration_ar.json index.html app/nav.js app/main.js audio/
git commit -m "content: complete the story of Nuh with phase navigation"
```

---

### Task 13: Release polish

**Files:**
- Modify: `index.html`, `style.css`, `package.json`, `README.md`
- Create: `docs/SOURCES.md`, `docs/ARCHITECTURE.md`, `docs/DATA_SCHEMA.md`

**Interfaces:**
- Consumes: everything.
- Produces: a shippable v0.2.0.

- [ ] **Step 1: Work the mobile checklist**

Open the page at 360×740 in device emulation and fix what fails:

- [ ] Every button has a ≥44px tap target
- [ ] The map caps at 40vh with no horizontal scroll
- [ ] The ayah stays readable (≥1.05rem)
- [ ] Controls wrap instead of overflowing
- [ ] `env(safe-area-inset-bottom)` respected on the phase nav

- [ ] **Step 2: Audit every fixed-position element**

Run: `grep -n "position: *fixed" style.css`

For each hit with `z-index > 100` and a background colour, confirm it defaults to `display: none`. A badge that ignored this rendered a phantom green strip in `../Sera` that survived ten false fixes.

- [ ] **Step 3: Write the three docs**

`docs/SOURCES.md` — the accepted canon, the cautions on al-Tabari, the rejected books and transmitters, and how `classAr` is assigned. `docs/DATA_SCHEMA.md` — every field of a phase and a beat, with the `link`-beat constraints. `docs/ARCHITECTURE.md` — module boundaries, the `file://` constraints, and §7 on the two audio paths and the cue mechanism.

- [ ] **Step 4: Rewrite `README.md`**

Cover what the project is, the sourcing method, how to run it locally, how to add a prophet, and the gate commands. State plainly that recitation needs network and narration does not.

- [ ] **Step 5: Bump the version everywhere**

Set `"version": "0.2.0"` in `package.json`, then update **every** `?v=` in `index.html` to `0.2.0` and `VERSION` in `app/loader.js`.

Run: `python tools/check_release.py`
Expected: clean. If you missed one, the CACHE check names the file — that check exists because three separate stale-cache incidents shipped in `../Sera`.

- [ ] **Step 6: Final verification from `file://`**

Close the dev server. Open `index.html` by double-clicking it.
Expected: the whole app works — intro, home, نوح, all five phases, narration, map sync. Only recitation requires the network. If anything fails here, a `fetch` or a module script crept in.

- [ ] **Step 7: Commit and tag**

```bash
git add -A
git commit -m "chore: release v0.2.0 — intro and the story of Nuh"
git tag v0.2.0
```

---

## Self-Review

**Spec coverage:** Intro screen → Task 2. Per-prophet on-demand data → Task 3. Variable phase counts → Tasks 5, 12 (nothing assumes a fixed count). Beats with cue-driven map sync → Tasks 7, 10. Sourcing gate with the four layers → Task 4. `link`-beat constraints → Task 4. Two generated voices with shakir default → Tasks 7, 9. al-Husary recitation → Task 11. Vocalization gates → Task 8. Error handling table → Tasks 3 (load failure), 9 (missing clip), 10 (missing cues), 11 (offline, unparseable ref). Mobile checklist → Task 13. Adam is correctly **out of scope** per the spec.

**Known gaps, stated rather than hidden:**

1. **`data/index.js` has five entries, not twenty-five.** Adding the remaining twenty is data entry against a fixed shape; it does not change any code. Do it when the second prophet lands.
2. **`check_sources.py` cannot read Arabic semantically.** Layer 4 raises notices for a human; the digit check catches `٥٠` but not `خمسون`. This is recorded in `docs/BUGS.md` in Task 5 Step 3 rather than papered over.
3. **The map SVG in Task 10 is schematic, not cartographic.** It carries the right *structure* — viewBox focus, a marker, bilingual labels. Replacing it with accurate geography is a content task that touches no JS.

**Type consistency:** `window.playPhase(prophetKey, phaseIndex)` (Task 9) matches its call in Task 12's `goPhase`. `window.loadCues(cueKey, cb)` (Task 9 Step 2) matches its use in Task 10. `window.renderPhase(prophet, phaseIndex)` takes the prophet *object*, not the key — consistent in Tasks 6 and 12. `cueKey` is `"<slot>/<prophet>_<phase>_ar"` in `gen_tts.py` (Task 7), `audio.js` (Task 9) and `loadCues` (Task 9) alike.
