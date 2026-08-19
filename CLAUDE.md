# Auto Škola — Project Context for AI Agents

## What this is

A single-page web app for learning Serbian driving theory questions.
Single HTML file (`index.html`) — no build step, no framework, no bundler.
Deployed on Netlify, source on GitHub at `https://github.com/MirkoM98/auto-skola-pitanja`.

---

## Tech stack

- Vanilla JS, no framework
- Tailwind CSS via CDN
- FontAwesome via CDN
- Cloudinary for image hosting (cloud name: `dnqbsacmr`)
- Giphy API for Office GIFs in simulation results (key in `GIPHY_KEY` const)
- Data: `autoskola_baza.json` — 828 questions across categories/subcategories

---

## File structure

```
index.html          — entire app (HTML + embedded JS, ~70 KB)
autoskola_baza.json — question database
CLAUDE.md           — this file
README.md           — user-facing project description
.gitignore          — excludes images/, *.py, .DS_Store
```

---

## Data shape

`autoskola_baza.json` is an array of **categories**, each with **subcategories**, each with **questions**:

```json
[{
  "categoryId": 1,
  "categoryName": "...",
  "subcategories": [{
    "subcategoryId": "1.1",
    "subcategoryName": "...",
    "questions": [{
      "questionId": 9355,
      "text": "Pitanje...",
      "points": 2,
      "choicesRequired": 1,
      "imagePath": "https://res.cloudinary.com/dnqbsacmr/image/upload/10853.jpg",
      "choices": [{
        "choiceId": 101,
        "text": "Odgovor A",
        "isCorrect": true
      }]
    }]
  }]
}]
```

On load each question gets enriched with: `_subId`, `_subName`, `_catId`, `_catName`.

- `points`: 2 or 3 (occasionally 1) — used for simulation scoring
- `choicesRequired > 1` means multi-choice question
- `imagePath`: Cloudinary URL or null. Format: `https://res.cloudinary.com/dnqbsacmr/image/upload/{number}.jpg`
- Questions without `imagePath` (or empty string) have no image

---

## Global state object `S`

All runtime state lives here. Never stored in component state.

```js
const S = {
  raw: [],              // enriched categories (loaded once from JSON)
  flagged: new Set(),   // questionIds — persisted in localStorage
  answered: {},         // {qid: {chosenIds:[], correct:bool}} — session only, reset on reload
  pending: {},          // {qid: Set<choiceId>} — multi-choice mid-selection

  openCats: new Set(),  // accordion state
  openSubs: new Set(),
  hideEmpty: true,      // hide subcategories with no matching questions
  onlyFlagged: false,
  query: '',            // current search string (lowercased)

  // Focus modal
  fqs: [],              // current subcategory's filtered questions
  fIdx: 0,              // current index in fqs
  fCurrentSubId: null,
  focusPositions: {},   // {subId: lastIdx} — restored when reopening
  fForceReveal: false,  // show answers without clicking

  // Flashcard (Znaci tab quiz)
  fcqs: [], fcIdx: 0,
  fcAns: null,          // {chosenIds[], correct} | null
  fcPend: new Set(),
  fcScore: {ok:0, fail:0},

  // Signs tab
  signsShowAll: false,  // false = only traffic signs, true = all image questions

  // Simulation
  simQs: [],            // selected questions for current simulation
  simIdx: 0,
  simAnswers: {},       // {qid: {chosenIds[], correct}}
  simPend: {},          // multi-choice in-progress
  simDone: false,
  simCheated: false,    // set by cheatSim() easter egg
  repeatQs: false,      // toggle: allow repeating used questions
  usedQids: new Set(),  // qids answered correctly in previous sims — persisted
};
```

---

## localStorage keys

| Key | Content |
|-----|---------|
| `autoskola_flagged_v2` | JSON array of flagged questionIds |
| `autoskola_sim_repeat` | `"true"` or `"false"` for repeatQs toggle |
| `autoskola_sim_used` | JSON array of correctly-answered questionIds |

---

## Constants

```js
const LS_KEY        = 'autoskola_flagged_v2';
const LS_REPEAT     = 'autoskola_sim_repeat';
const LS_USED       = 'autoskola_sim_used';

const SIM_QUESTIONS = 41;          // questions per simulation
const SIM_PASS_PCT  = 0.85;        // need >= 85% of total points to pass
const GIPHY_KEY     = '7mqmxcBufeOh2foGORCkkiNuL9e1zvUt';
const TEST_MODE     = new URLSearchParams(location.search).get('test-mode') === 'true';

const SIGN_PREFIX   = 'saobraćajni znak';  // questions starting with this are traffic signs
```

---

## CSS classes (custom, defined in `<style>`)

### Card states
| Class | Meaning |
|-------|---------|
| `q-flagged` | Yellow border — highest visual priority, overrides correct/wrong |
| `q-correct` | Green border — question answered correctly |
| `q-wrong` | Red border — question answered incorrectly |
| (none) | White — unanswered, unflagged |

Priority rule: flagged > correct > wrong > white. A flagged+wrong card shows yellow.

### Choice button states
| Class | Meaning |
|-------|---------|
| `cb-idle` | Unanswered, not selected |
| `cb-pending` | Multi-choice: selected but not confirmed |
| `cb-correct` | Correct answer, user chose it |
| `cb-missed` | Correct answer, user did NOT choose it |
| `cb-wrong` | Wrong answer, user chose it |
| `cb-neutral` | Wrong answer, user did not choose it |

### Other
| Class | Notes |
|-------|-------|
| `toggle-track` | Custom toggle track (blue=on, grey=off) |
| `toggle-thumb` | Custom toggle thumb |
| `thin-scroll` | Thin scrollbar styling |
| `zoomable` | Cursor zoom on images (opens lightbox on click) |

---

## Key functions

### Rendering

**`renderQ(q)`** — returns HTML string for one question card. Does NOT touch the DOM.
Answer order is always shuffled via `seededShuffle(q.choices, q.questionId)` — stable per question, different every render without storing the shuffled order.

**`reRenderCard(qid)`** — in-place DOM replacement of one question card.
CRITICAL: uses `tmp.firstElementChild` (NOT `tmp.firstChild`). `renderQ()` returns `\n<div...>` so `firstChild` is a whitespace text node and breaks silently.

```js
function reRenderCard(qid) {
  const tmp = document.createElement('div');
  tmp.innerHTML = renderQ(q);
  card.replaceWith(tmp.firstElementChild); // NOT firstChild
}
```

**`renderAll()`** — full re-render of the accordion list. Expensive but simple.
**`renderAllKeepScroll()`** — saves/restores `window.scrollY` around `renderAll()`.

### Answer flow

**Single choice**: `answerSingle(qid, cid, isCorrect)` → sets `S.answered[qid]` → `reRenderCard(qid)`

**Multi-choice**:
1. `togglePending(qid, cid)` — adds/removes from `S.pending[qid]` (Set)
2. `checkMulti(qid)` — validates, sets `S.answered[qid]`, deletes `S.pending[qid]`

Both flows use `reRenderCard` for in-place update.

### Search

`onSearch(v)` sets `S.query`. `filteredData()` applies it.
Search by questionId: `String(q.questionId) === S.query` is the first condition — safe because IDs don't appear in answer text.

### Focus modal

`openFocus(subId, catName, subName, startIdx)` — opens modal at `startIdx`.
`openFocusAtQ(subId, qid)` — finds the question's index in the sub's filtered list and calls `openFocus`.
`S.focusPositions[subId]` — saved on close, restored on next open of same subcategory.
Modal dimensions: `height: min(92vh, 700px)` with `flex flex-col`. Body is `flex-1 overflow-y-auto`. Footer is `flex-shrink-0` — prevents footer jumping on content change.

### Traffic signs (`isTrafficSign`)

```js
function isTrafficSign(q) {
  return q.text.toLowerCase().startsWith(SIGN_PREFIX); // 'saobraćajni znak'
}
```
257 out of 828 image questions qualify as traffic signs.
`signsShowAll: false` = show only traffic signs (default). `true` = show all image questions.

---

## Simulation logic

### Question pool selection (`selectSimQuestions`)

1. All questions from `S.raw` go into `pool`
2. If `S.repeatQs === false`: filter out `S.usedQids` → `available`. If available is empty, fall back to full pool.
3. Shuffle, take up to `SIM_QUESTIONS` (41)

### Saving used questions (`finishSim`)

ONLY correctly answered questions are added to `S.usedQids`.
Wrong/unanswered questions are NEVER added — they always re-enter the pool regardless of the repeat toggle.

```js
S.simQs.forEach(q => {
  if (S.simAnswers[q.questionId]?.correct) S.usedQids.add(q.questionId);
});
```

### Scoring (`calcSimScore`)

```js
// Returns: { totalPossible, scored, lost, answered, wrongCount, passed }
passed: totalPossible > 0 && scored / totalPossible >= SIM_PASS_PCT  // 85%
```

Points matter, not question count. Questions have 2–3 points each. A typical 41-question sim has ~90 total points; threshold is 76+ points scored.

### Results screen

- Pass: "POLOŽEN!" + "Bravo!" + happy Office GIF
- Fail: "NIJE POLOŽEN" + "Stolica nedovoljno zagrejana." + disappointed Office GIF
- After stats: full review section of all wrong/unanswered questions with correct answers highlighted
- Chip in header: "POLOŽEN" (green) or "NIJE POLOŽEN" (red)

### Easter egg: cheat button

A nearly-invisible `·` after "Simulacija ispita" in the modal header (`opacity-5`).
Click → `cheatSim()` → `S.simCheated = true` → special "VARALICA!" screen with caught Office GIF.

### Test mode

`?test-mode=true` in URL shows `✓ pass` / `✗ fail` buttons in the simulation footer.
- `forceSimPass()` — marks all questions correct, goes to results
- `forceSimFail()` — clears all answers (0%), goes to results
Neither saves to `usedQids`.

---

## Giphy integration

`fetchOfficeGif(mode)` where `mode` is `'pass'`, `'fail'`, or `'cheat'`.

| Mode | Search query |
|------|-------------|
| `pass` | `"the office happy celebrate"` |
| `fail` | `"the office disappointed"` |
| `cheat` | `"the office caught busted"` |

Returns a random URL from 25 results (`images.downsized.url`). Injected into `#sim-gif-area` after results render — emoji shows as fallback while loading.

---

## Known gotchas

1. **`tmp.firstElementChild` not `tmp.firstChild`** in `reRenderCard` — `renderQ()` returns a string starting with `\n` so `firstChild` is a text node.

2. **`seededShuffle` uses questionId as seed** — same order every render, no stored shuffle state needed.

3. **`S.answered` is session-only** — not persisted. Only `S.flagged` and sim data persist.

4. **Focus modal footer** uses `flex-shrink-0` — critical so it doesn't move when question content changes height. Modal itself uses `height: min(92vh, 700px)` not `max-height`.

5. **Signs tab default**: `signsShowAll: false` = only traffic signs shown. Toggle is grey (off) by default.

6. **Multi-choice detection**: `q.choicesRequired > 1`, not a boolean flag.

7. **Image errors**: `onerror="this.parentElement.remove()"` removes the wrapping div silently.

8. **`h(s)`** — HTML escape utility. `jsq(s)` — escapes for inline JS string attributes. Always use both for user content.

9. **Simulation score chip is hidden during sim** — only shown on results screen. Do not add logic to show it during questions.

10. **`S.simCheated` must be reset to `false` in `startSimulation()`** — otherwise subsequent sims inherit the cheat state.
