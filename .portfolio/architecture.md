# Architecture

## System Diagram

```mermaid
flowchart TD
    subgraph Popup["Popup (popup/)"]
        UI[menu.html / menu.js]
        CSV[csvParser.js — GFAFCsv]
    end

    subgraph SW["Service Worker"]
        BG[background.js — popupSession port]
    end

    subgraph Content["Content Script (docs.google.com/forms/*)"]
        STORE[storage.js — GFAFStorage]
        FILL[GoogleForm.js — match + fill + observer]
    end

    DB[(chrome.storage.local)]
    DOM[Google Forms DOM]

    UI -->|edit / import| STORE
    CSV --> STORE
    UI -->|stream latest on edit| BG
    BG -->|flush on popup close| DB
    STORE <--> DB
    FILL -->|read answers + settings| STORE
    FILL -->|fill fields| DOM
    DOM -->|mutations re-trigger| FILL
    STORE -. storage.onChanged .-> FILL
```

## Component Descriptions

### Storage helper (`GFAFStorage`)
- **Purpose**: Single source of truth for persistence, shared by the popup and the content script.
- **Location**: `scripts/storage.js`
- **Key responsibilities**: Wrap `chrome.storage.local` get/set for form data and settings; check `chrome.runtime.lastError` on every write; run a one-time, idempotent `chrome.storage.sync → local` migration guarded by a `_migrated` marker.

### Content script (matching + filling)
- **Purpose**: Detect form fields, match them to saved answers, and fill them.
- **Location**: `scripts/GoogleForm.js`
- **Key responsibilities**: Per input type, select fields and read each field's title; fuzzy-match the title to a saved key via Levenshtein similarity; write the value and dispatch `input`/`change`; observe the form for late-rendered sections.

### CSV parser (`GFAFCsv`)
- **Purpose**: Turn an uploaded CSV into key/value answers.
- **Location**: `scripts/csvParser.js`
- **Key responsibilities**: Single-pass, quote-aware parsing; merge into stored data honoring an overwrite flag; report read/write failures instead of silently losing data.

### Popup UI
- **Purpose**: Where the user manages answers and settings.
- **Location**: `popup/menu.html`, `popup/menu.js`, `popup/styles.css`
- **Key responsibilities**: Editable key/value rows, CSV import, date-format selector; persist through `GFAFStorage`; trigger a re-fill of the active tab.

### Service worker
- **Purpose**: Guarantee the last edit survives the popup closing.
- **Location**: `background.js`
- **Key responsibilities**: Accept a `popupSession` port, buffer the latest snapshot, and flush it to storage synchronously on disconnect.

## Data Flow

1. The user adds answers in the popup (or imports a CSV); each change writes to `chrome.storage.local` via `GFAFStorage` and is streamed to the service worker.
2. On a Google Forms page, the content script reads the saved answers and the date-format setting.
3. For each input, it reads the field title, fuzzy-matches it to a saved key, and — if a match clears the similarity threshold — writes the value and dispatches `input`/`change`.
4. A `MutationObserver` (disconnected during writes, debounced afterward) re-fills sections Google Forms renders lazily.
5. A `storage.onChanged` listener invalidates the match cache so live popup edits re-fill immediately.

## External Integrations

| Service | Purpose | Notes |
|---------|---------|-------|
| Google Forms (`docs.google.com/forms/*`) | The page whose fields are filled | No API — the content script reads/writes the live DOM and dispatches synthetic input events |
| `chrome.storage.local` | Answer + settings persistence | No per-item quota cap; data is device-local |

## Key Architectural Decisions

### `chrome.storage.local` over `chrome.storage.sync`
- **Context**: Answers and bulk CSV imports can exceed `sync`'s ~8KB-per-item / ~100KB-total quotas, and quota failures are easy to drop silently.
- **Decision**: Persist to `local`, with a one-time migration that copies existing `sync` data forward.
- **Rationale**: `local` removes the quota ceiling that made large imports unreliable. The trade-off is losing cross-device sync; for a fill-from-saved-answers tool, reliable local storage matters more than roaming. The migration is marker-guarded so it never clobbers newer local data and bails out on a read error.

### Disconnect-and-debounce around programmatic writes
- **Context**: Filling a field dispatches `input`/`change`, which Google Forms handles by mutating the DOM — which fires the `MutationObserver` that triggers the fill, creating a feedback loop.
- **Decision**: Hoist the observer, `disconnect()` before the write pass and reconnect in a `finally`, and debounce observer-driven fills (~120ms).
- **Rationale**: The observer is still needed for genuinely lazy-loaded sections, so removing it wasn't an option. Pausing it only around self-induced mutations breaks the loop while preserving late-fill behavior.

### Fuzzy matching by normalized Levenshtein similarity
- **Context**: Form authors phrase the same field many ways (*E-mail*, *email*, *Email Address*); exact-key matching would force users to pre-register every variant.
- **Decision**: Compute `1 - distance/maxLen` between the lowercased field title and each saved key, and take the best match above a similarity threshold.
- **Rationale**: One saved key covers many phrasings. Results are memoized per field title (keyed on the saved-key set) so the O(n·m) distance isn't recomputed on every observer tick.

### Shared, namespaced modules instead of ES modules
- **Context**: The same parsing/storage logic is needed in both the popup and the content script, but MV3 content scripts and popup `<script>` tags don't share an ES-module graph cleanly.
- **Decision**: Expose `GFAFStorage` and `GFAFCsv` as IIFE-guarded globals, injected in load order via the manifest and the popup HTML.
- **Rationale**: Code is shared with no bundler and no `web_accessible_resources` plumbing. Double-definition guards make re-injection safe.

### Port-based save-on-close
- **Context**: The popup writes on each change, but it can be dismissed before an async write settles.
- **Decision**: Open a `popupSession` port; the popup streams the latest snapshot, and the service worker flushes it synchronously on disconnect.
- **Rationale**: A belt-and-suspenders guarantee for the last edit, using the connection lifecycle rather than fragile timers, with no extra permissions.
