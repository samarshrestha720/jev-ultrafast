# How Jev Ultrafast Works (Simple Explanation)

> Personal note — not part of the official docs in `docs/`. Written to understand the repo.

There are 2 different models with 2 different jobs:

1. **Jev / TypeSafe** — decides *what to do next, and where*.
2. **Text LLM** — decides *what string to type* (only when needed).

Think of it like: Jev = dispatcher pointing at a button/field, Text LLM = typist, Python code = hands that actually click/type.

## The Loop

```
goal -> [1. look at page] -> [2. Jev picks operation+target] -> [3. maybe Text LLM writes] -> [4. execute] -> repeat
```

### 1. Look — `jev_ultrafast/snapshot.js`

Before each step, code in the browser reads the visible page in one atomic snapshot and builds an **indexed element table**:

```text
[1] button    Change ticket type · Round trip
[2] combobox  Where from?        · San Francisco
[3] combobox  Where to?          · empty
[4] textbox   Departure          · empty
```

It records for each control: role, label, current value, checked/selected state, and visible page text (max 6000 chars). One node = one index, even if it supports both clicking and typing. No screenshots are sent to the model by default.

### 2. Jev decides — `jev_ultrafast/model.py` `choose()`

Jev is called via `https://api.typesafe.ai/v1/systemone` with model `jev-latest`. It receives:

- the original goal (kept unchanged the whole run)
- the page URL/title + visible text
- the element table + recent actions (last 10)

Jev never outputs a selector, coordinate, or JavaScript. It only picks from a menu — **one `operation` head + speculative `target` heads**, all in **one network request**:

```text
operation:
  CLICK       or TYPE_TEXT or SELECT or SCROLL_UP/DOWN or WAIT or DONE or BLOCKED

click_target:      [1], [5], "Open [2]", ...
type_text_target:  [2], [3], [4], ...
select_target:     [2]:2, [2]:3, ...  (dropdown options have code-owned indices)
```

That is the "speculative fan-out": the server evaluates operation *and* all applicable targets in parallel. The executor only uses the target matching the chosen operation, which avoids incompatible choices and saves one round-trip. Code validates the answer (probabilities sum to 1, choice exists, etc.) or aborts — it never executes an invalid choice.

Instructions in `jev_ultrafast/questions.py` tell Jev to use current field values, *not* repeat satisfied steps, don't toggle a checkbox already in the right state, click Search when fields are ready, only `DONE` when all requirements are visibly satisfied.

### 3. Text LLM writes — only for `TYPE_TEXT` — `jev_ultrafast/model.py` `field_text()`

If Jev chose `TYPE_TEXT` on, say, `[3] Where to?`, *then* a second, small, cheap LLM is called (e.g. `inception/mercury-2.5` via OpenRouter, configured by `TEXT_MODEL*` env vars in `.env.example`).

It gets:

- the original goal
- the selected field (label, role, current value)
- visible page context + recent actions (last 6)

It must return **exactly**:

```json
{"text": "London"}
```

No commentary, no code. Code strictly parses this JSON, rejects empty >2000 chars, or `{"text": null}` if the value isn't inferable. Only *that* string is typed.

If the page goes stale while generating and the context is *byte-identical*, the generated value is cached and reused — `pending_text` in `agent.py`. Any change discards it. No retries after mutation.

### 4. Execute — `jev_ultrafast/browser.py` + `agent.py`

The index `[3]` is resolved back to the *actual DOM node* via a code-owned `WeakMap` ID (set in `snapshot.js`). The model never invented that ID.

Before clicking, Python re-checks freshness:

- page URL/document/viewport/form values haven't changed
- target is still visible, enabled, not covered, geometry is re-read fresh

Then:

- `CLICK` -> CDP mouse press at center of node
- `TYPE_TEXT` -> select-all + `Input.insertText`
- `SELECT` -> set `value` + fire input/change events
- `SCROLL` -> wheel event, `WAIT` -> 100ms

Execution is logged **before** the next observation, so a navigation can't erase the fact an action happened. Nothing is retried blindly. After acting, code waits at most 2 animation frames / 50ms (or up to 200ms if typing in a combobox to let autocomplete suggestions appear), then takes a new snapshot.

Loop runs until Jev picks `DONE`, `BLOCKED`, 3 identical no-change actions, or 60 steps. `DONE` from the model is **not trusted** — examples like `examples/flights.py` independently verify the final page (Zürich, London, date, visible flight options).

## Concrete example: Zurich -> London

Goal: *"Find one-way flights from Zurich to London on Sept 20, 2026..."*

1. Snapshot shows `Where from? empty`, `Where to? empty`, `Departure empty`
2. Jev: `operation=TYPE_TEXT`, `type_text_target=[2]` (speculatively also computed click/select targets, ignored)
3. Text LLM: `{"text":"Zurich"}` -> typed into `[2]`
4. Wait for suggestions -> Snapshot
5. Jev: `TYPE_TEXT [3]` -> Text LLM: `{"text":"London"}` -> typed
6. Jev: `CLICK` on autocomplete suggestion `[8] Zurich, Switzerland`
7. Jev: `CLICK` on date, picks Sept 20, confirms
8. Jev: `CLICK [Search]`, page loads
9. Jev: `DONE` — then Python's independent checker verifies route/date/results.

Why it's fast: 1 Jev round-trip per step (not operation-then-target), 1 DOM snapshot per step, no screenshots in the model loop, minimal waits, and background focus emulation so a hidden tab still animates.

## File map

| File | Job |
| --- | --- |
| `agent.py` | The loop: predict -> (maybe write text) -> act -> observe |
| `model.py` | Builds element table, calls Jev, calls Text LLM |
| `questions.py` | Prompts for Jev + Text LLM |
| `snapshot.js` | Builds element table + freshness guards in browser |
| `browser.py` | CDP connection, geometry/click checks, execution |

## Limits to keep in mind

- `DONE` from Jev is not proof — independent verification is required.
- DOM reader handles common HTML/ARIA controls, not the full accessibility spec.
- Shadow roots, frames, canvas, uploads, pop-ups, nested scrolling, and complex keyboard widgets can block progress.
- A valid action can still be wrong.
