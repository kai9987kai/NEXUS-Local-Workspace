# NEXUS — Local Intelligence Workspace

NEXUS is a **single-file, local-first browser workspace** for managing actions, technical projects, career opportunities, study readiness, experiments, portfolio evidence, contextual knowledge and focused work.

The application runs entirely in the browser from one HTML file. It has no package manager, build step, server-side database, analytics SDK or external runtime dependency. Its interface uses keyed partial DOM updates, while computationally heavier ranking, fuzzy search, simulation and benchmarking operations run inside an embedded Web Worker.

> **Privacy warning**
>
> The distributed HTML contains initial seed records inside its source code. Remove or replace those records before publishing the file publicly. Runtime data stored in `localStorage` and IndexedDB is not encrypted at rest. The privacy-blur control is visual concealment, not cryptographic protection.

---

## Contents

- [Core capabilities](#core-capabilities)
- [Technical architecture](#technical-architecture)
- [Decision engine](#decision-engine)
- [Persistence model](#persistence-model)
- [Backup formats](#backup-formats)
- [Experimental browser APIs](#experimental-browser-apis)
- [Running the application](#running-the-application)
- [Workspace modules](#workspace-modules)
- [Keyboard and interaction controls](#keyboard-and-interaction-controls)
- [Data model](#data-model)
- [Customisation](#customisation)
- [Privacy and security](#privacy-and-security)
- [Browser compatibility](#browser-compatibility)
- [Known limitations](#known-limitations)
- [Troubleshooting](#troubleshooting)
- [Development checklist](#development-checklist)
- [Suggested future work](#suggested-future-work)
- [Licence](#licence)

---

## Core capabilities

### Adaptive action planning

NEXUS scores active actions against the current operating state:

- Energy level from 1 to 5
- Available focus time
- Working mode: balanced, deep work, admin, creative or low-friction
- Priority and expected impact
- Due-date pressure
- Effort and energy requirements
- Portfolio value
- Dependency state

The highest-scoring action becomes the current recommendation. A dynamic day plan is then assembled from feasible actions.

### Technical project intelligence

The Project Lab tracks:

- Delivery status
- Build progress
- Impact
- Novelty
- Evidence quality
- Delivery risk
- Technology tags
- Highest-value next evidence action

Each project receives a derived proof score. NEXUS can also export a project-specific Markdown proof pack containing a problem statement, novelty summary, current evidence state, limitations and a suggested portfolio structure.

### Career opportunity pipeline

Career Radar provides a four-stage pipeline:

1. Discovered
2. Applied
3. Interview
4. Offer

Each opportunity receives a fit percentage derived from flexibility, technical alignment and portfolio alignment.

### Study re-entry model

The Study Re-entry area combines:

- Module-level readiness
- Confidence ratings
- Open study actions
- Practical return constraints
- An aggregate readiness percentage

### Experiment design

The Experiment Lab supports:

- Hypothesis-driven project definitions
- Explicit success metrics
- Test methods
- Progress and state tracking
- Local concept synthesis
- Weighted decision matrices

Generated experiments are created from a domain, system form and desired outcome, then converted into a measurable hypothesis rather than remaining as an unstructured idea.

### Local contextual knowledge

The Knowledge Vault stores editable context used by the workspace. Entries can be categorised and marked sensitive. Sensitive cards are blurred by default when privacy mode is enabled.

### Interactive systems graph

The Systems Graph visualises relationships among:

- Projects
- Interests
- Responsibilities

The graph uses a canvas-based force simulation. Nodes can be dragged and pinned, with their positions saved locally.

### Adaptive focus mode

Focus mode provides:

- Configurable countdown duration
- Action title, category and notes
- Pause and resume
- Completion workflow
- Screen Wake Lock where supported
- Document Picture-in-Picture mini timer where supported
- Completion notification where permission is available

---

## Technical architecture

NEXUS is intentionally implemented as a self-contained HTML document:

```text
nexus-local-intelligence.html
├── Semantic HTML application shell
├── Embedded experimental CSS
├── Reactive state store
├── Keyed partial DOM reconciler
├── Embedded Blob-backed Web Worker
├── localStorage snapshot layer
├── IndexedDB event journal and snapshots
├── BroadcastChannel tab synchronisation
├── Web Crypto backup encryption
└── Import/export and capability detection
```

### Reactive store

`ReactiveStore` owns the authoritative in-memory state and exposes channel-based subscriptions. Mutations are applied through transactions:

```javascript
store.transaction(
  state => {
    // Mutate a specific state branch.
  },
  ['actions', 'dashboard'],
  { type: 'update-action', entityId: '...' }
);
```

A transaction:

1. Mutates the state.
2. Updates mutation metadata.
3. Adds an activity entry.
4. Schedules persistence.
5. Emits only the relevant subscription channels.
6. Appends an event to IndexedDB.
7. Periodically creates a recovery snapshot.

### Dynamic partial updates

The interface does not rely on a framework or rebuild the entire document after every change. Repeated collections use a keyed reconciler:

```javascript
keyedPatch(container, items, keyOf, create, update);
```

For each render pass, the reconciler:

1. Maps existing children by `data-key`.
2. Reuses matching elements.
3. Creates only missing elements.
4. Updates retained elements in place.
5. Removes obsolete elements.
6. Reorders nodes using a `DocumentFragment`.

Small scalar values use targeted `patchText()` and `patchAttr()` operations to avoid unnecessary writes.

### Worker-based computation

A Web Worker is generated from an embedded JavaScript string and loaded through a Blob URL. The worker handles:

- Action ranking
- Day-plan construction
- Trigram fuzzy search
- Monte Carlo day simulation
- Synthetic computation benchmarking

This keeps the main thread available for input, navigation and animation.

### Navigation transitions

View changes use the View Transitions API when available and immediately fall back to normal DOM state changes when unsupported.

---

## Decision engine

The action score is a deterministic heuristic calculated inside the Web Worker.

Conceptually:

```text
score =
    priority × 8
  + impact × 6
  + urgency
  + energy fit
  + time fit
  + working-mode fit
  + portfolio bonus
  + dependency penalty
  - effort × 2
```

### Urgency component

| Due state | Urgency points |
|---|---:|
| Overdue or due today | 30 |
| Due tomorrow | 24 |
| Due within 3 days | 18 |
| Due within 7 days | 10 |
| Later or no immediate pressure | 3 |

### Energy fit

Energy fit rewards actions whose required energy is close to the current energy level:

```text
12 - abs(actionEnergy - currentEnergy) × 3
```

### Time fit

An action receives the maximum time-fit reward when its duration fits inside the selected focus window. Longer actions receive a progressively smaller value and can eventually receive a negative time-fit score.

### Mode fit

The current working mode provides a category bonus:

| Mode | Favoured categories |
|---|---|
| Deep work | AI research, Portfolio, Study |
| Admin | Career, Life admin |
| Creative | Portfolio, AI research |
| Low-friction | Life admin, Career |
| Balanced | No category-specific bonus |

### Dependency handling

An action receives a strong negative penalty when one of its referenced dependencies exists and is incomplete. The current editor does not expose dependency selection, but the data model and ranking logic support dependency IDs.

### Day-plan construction

The worker sorts actions by score and selects up to five feasible actions. The planning horizon is three times the selected focus window. Actions with negative scores or durations larger than the remaining planning capacity are skipped.

### Monte Carlo simulation

The day simulator runs thousands of randomised scheduling trials in the worker. It reports:

- Mean completed-action count
- Mean accumulated decision value
- Number of simulation runs
- Worker execution duration

This is a heuristic planning experiment, not a predictive model or guarantee of future productivity.

---

## Persistence model

NEXUS uses two browser persistence layers.

### 1. Fast snapshot: `localStorage`

The complete current state is serialised under:

```text
nexus.local.intelligence.v3
```

Writes are debounced to reduce repeated synchronous storage operations.

### 2. Journal and snapshots: IndexedDB

Database name:

```text
nexus-local-journal
```

Object stores:

| Store | Purpose |
|---|---|
| `events` | Append-only mutation records |
| `snapshots` | Full cloned recovery states |

A new IndexedDB snapshot is automatically requested after every twentieth mutation. A snapshot can also be created manually in Systems Lab.

### Cross-tab propagation

When supported and enabled, `BroadcastChannel` publishes the complete state to other open NEXUS tabs through:

```text
nexus-workspace-sync
```

The current implementation uses whole-state propagation and effectively follows last-write-wins behaviour. It is not a CRDT and does not merge simultaneous conflicting edits.

### Legacy migration

At startup, NEXUS checks for its current schema. It can also scan older local storage records containing compatible task and project arrays, then map them into the current action, project, opportunity, memory, experiment and study structures.

---

## Backup formats

Systems Lab exposes four data operations.

### Plain JSON

Extension:

```text
.json
```

Contains formatted, human-readable workspace state.

### Gzip-compressed JSON

Extension:

```text
.json.gz
```

Uses `CompressionStream('gzip')`. When Compression Streams are unavailable, the app falls back to plain JSON export.

### Encrypted NEXUS backup

Extension:

```text
.nexus
```

Encryption design:

| Component | Implementation |
|---|---|
| Password derivation | PBKDF2 |
| Hash | SHA-256 |
| Iterations | 250,000 |
| Salt | 16 random bytes |
| Cipher | AES-GCM |
| Key length | 256 bits |
| IV | 12 random bytes |
| File header | `NEXUS3` |

Binary layout:

```text
[6-byte header][16-byte salt][12-byte IV][AES-GCM ciphertext]
```

The passphrase is never stored. An encrypted backup cannot be recovered if its passphrase is lost.

### Import

The importer accepts:

- `.json`
- `.json.gz`
- `.nexus`

Imported state is normalised to the current schema before being persisted and rendered.

---

## Experimental browser APIs

NEXUS detects capabilities at runtime and displays the result in Systems Lab.

| API or CSS feature | Usage | Fallback |
|---|---|---|
| View Transitions | Animated workspace navigation | Immediate view change |
| WebGPU | Capability detection for future compute features | No current compute dependency |
| Web Worker | Ranking, search, simulation, benchmark | Core modern-browser requirement in this build |
| IndexedDB | Event journal and snapshots | `localStorage` remains active |
| BroadcastChannel | Cross-tab state propagation | Tabs operate independently |
| Scheduler API | Background-priority work | `requestIdleCallback` or `setTimeout` |
| Compression Streams | Gzip backup | Plain JSON backup |
| File System Access API | Native save-file picker | Anchor-based browser download |
| Wake Lock | Prevent screen sleep during focus | Timer continues without lock |
| Document Picture-in-Picture | Floating focus timer | Main focus dialog |
| Web Crypto | Encrypted backup | Encrypted export unavailable |
| Speech Recognition | Voice action capture | Manual action editor |
| Notification API | Focus-completion notification | In-app toast |
| CSS Container Queries | Component-responsive cards | Media-query layout still applies |
| CSS OKLCH and `color-mix()` | Perceptual theme system | Requires a modern CSS engine |
| Typed CSS custom properties | Animated rings and scan position | Static or reduced animation |
| Scroll-driven animation | Hero scroll response | Hero remains static |
| `field-sizing: content` | Adaptive textarea sizing | Standard resizable textarea |

Availability varies by browser, platform, security context and permission state.

---

## Running the application

### Option 1: open the file directly

Open:

```text
nexus-local-intelligence.html
```

This is sufficient for the core workspace in browsers that allow local-file storage and worker creation.

Some browser APIs can be restricted under `file://`. For more consistent behaviour, use a local HTTP server.

### Option 2: Python local server

From the directory containing the HTML file:

```bash
python -m http.server 8080
```

Then open:

```text
http://localhost:8080/nexus-local-intelligence.html
```

### Option 3: Node.js local server

```bash
npx serve .
```

Open the local URL printed by the command.

### Hosting

The file can be hosted on any static host, including GitHub Pages. HTTPS is recommended because several browser capabilities require a secure context.

Before public deployment:

1. Remove private seed records from `seedState()`.
2. Add an appropriate Content Security Policy.
3. Test Blob Worker compatibility with that policy.
4. Add a licence.
5. Review accessibility and keyboard behaviour.
6. Confirm that imported data is validated to the required standard.

---

## Workspace modules

### Command Deck

Provides:

- Current highest-value action
- Energy, time and working-mode controls
- Momentum score
- Recent completion streak
- Dynamic day plan
- Portfolio signal cards
- Day simulation

### Action Graph

Provides:

- Create, edit, complete, reopen and delete operations
- Full-text filtering
- Category and completion filters
- Sorting by score, due date or creation time
- Focus-session launch
- Overdue indication

### Project Lab

Provides:

- Project creation and editing
- Status filtering
- Leverage, progress and risk sorting
- Proof-readiness calculation
- Markdown proof-pack export

### Career Radar

Provides:

- Opportunity creation and editing
- Stage counts
- Fit scoring
- One-click stage advancement
- Kanban-style visual layout

### Study Re-entry

Provides:

- Aggregate readiness score
- Module confidence and readiness controls
- Study-specific actions
- Private constraint context

### Experiment Lab

Provides:

- Experiment records
- Concept synthesis
- Weighted decision matrix
- Progress and status tracking
- Measurable hypothesis structure

### Knowledge Vault

Provides:

- Editable contextual records
- Categories
- Search
- Sensitive-item marking
- Privacy blur

### Systems Graph

Provides:

- Canvas force simulation
- Project, interest and responsibility nodes
- Dragging and position persistence
- Recentring

### Systems Lab

Provides:

- Capability matrix
- Storage usage estimates
- IndexedDB event and snapshot counts
- JSON, gzip and encrypted backups
- Import
- Worker benchmark
- Theme and accent configuration
- Focus duration configuration
- Cross-tab control
- Notification permission request
- Workspace reset

---

## Keyboard and interaction controls

| Input | Action |
|---|---|
| `Ctrl + K` | Open the command palette |
| `Cmd + K` | Open the command palette on macOS |
| `Arrow Up` / `Arrow Down` | Move through command results |
| `Enter` | Run the selected command |
| Drag graph node | Reposition and persist the node |
| Energy buttons | Update the shared energy state |
| Privacy button | Blur or reveal sensitive context |
| Theme button | Toggle dark and light themes |

The global search field opens the command palette. Search results are generated with trigram similarity inside the Web Worker.

---

## Data model

Top-level state:

```javascript
{
  schema,
  settings,
  actions,
  projects,
  opportunities,
  modules,
  experiments,
  memories,
  interests,
  responsibilities,
  decisionMatrix,
  graphPositions,
  activity,
  meta
}
```

### Action

```javascript
{
  id,
  title,
  category,
  priority,
  impact,
  effort,
  energy,
  minutes,
  due,
  done,
  notes,
  portfolio,
  dependencies,
  created,
  completedAt
}
```

### Project

```javascript
{
  id,
  name,
  description,
  status,
  progress,
  impact,
  novelty,
  evidence,
  risk,
  tags,
  next
}
```

### Opportunity

```javascript
{
  id,
  company,
  role,
  stage,
  date,
  location,
  flexibility,
  technical,
  portfolioFit,
  notes
}
```

### Experiment

```javascript
{
  id,
  title,
  hypothesis,
  metric,
  method,
  status,
  progress,
  tags
}
```

### Context record

```javascript
{
  id,
  category,
  text,
  sensitive
}
```

---

## Customisation

### Change the initial data

Edit `seedState()` inside the HTML file. This function defines the default:

- Actions
- Projects
- Opportunities
- Modules
- Experiments
- Context records
- Interests
- Responsibilities
- Settings

Changing seed data does not overwrite an existing saved workspace. Use **Reset workspace** after changing the source, or change the storage key/schema during development.

### Change storage identifiers

The main constants are:

```javascript
const APP_KEY = 'nexus.local.intelligence.v3';
const DB_NAME = 'nexus-local-journal';
const DB_VERSION = 1;
const SCHEMA = 3;
```

Increment `SCHEMA` when introducing incompatible state changes and update the migration logic accordingly.

### Change the scoring model

Edit `priorityScore()` inside `workerSource`. Because the function runs in the worker, changes remain isolated from the main rendering thread.

Useful future scoring dimensions include:

- Estimated interruption risk
- Context-switching cost
- Required location or device
- Hard deadlines versus soft targets
- Project dependencies
- Expected evidence value
- Application-closing dates
- Recovery requirements
- Confidence or uncertainty

### Change visual design

The primary design tokens are declared under `:root`:

```css
--hue
--bg
--bg-2
--surface
--surface-2
--text
--muted
--accent
--accent-2
--accent-3
--warn
--danger
--good
--radius-xl
--radius
--shadow
```

The accent-hue range is configurable from Systems Lab.

### Add a new module

A typical module requires:

1. A navigation button with `data-view`.
2. A `<section class="view" id="view-...">` container.
3. A state collection or derived selector.
4. A render function.
5. A subscription channel.
6. Create/edit/delete event handlers.
7. Backup and migration consideration.

---

## Privacy and security

### What remains local

The current application source does not perform network requests. Workspace mutations are stored in the browser profile and exported only when the user explicitly invokes a backup operation.

### What is not encrypted

The following are unencrypted at rest:

- `localStorage` state
- IndexedDB event records
- IndexedDB snapshots
- Seed data embedded in the HTML source
- Plain JSON backups
- Gzip backups

Anyone with access to the browser profile, exported files or HTML source may be able to inspect this information.

### Privacy blur

Privacy mode applies a CSS blur and disables selection for sensitive context cards. It protects against casual shoulder-surfing only. The underlying text remains in the DOM and browser storage.

### Encrypted backups

Encrypted `.nexus` exports provide authenticated encryption for the exported file. They do not encrypt live browser storage.

### Imported data

Imported JSON becomes trusted application state after parsing and normalisation. Displayed text is escaped before insertion in most dynamic templates, which reduces script-injection risk. Stronger schema validation should be added before accepting backups from untrusted sources.

### Content Security Policy

The single-file design uses:

- Inline CSS
- Inline JavaScript
- A Blob-backed Worker

A strict hosted Content Security Policy must explicitly account for these implementation choices. A production deployment should preferably move scripts and styles into hashed external assets or use CSP hashes/nonces.

---

## Browser compatibility

A recent Chromium-based browser provides the broadest feature coverage. Core layout, forms, local storage and standard IndexedDB behaviour should work in other modern browsers, but experimental APIs can be unavailable.

NEXUS performs runtime capability checks rather than assuming support. Unsupported optional features display fallbacks where implemented.

For the most complete environment, use:

- A modern desktop browser
- HTTPS or `http://localhost`
- Storage permission enabled
- JavaScript enabled
- Pop-ups/downloads permitted when exporting
- Notification, microphone or wake-lock permission only when those features are needed

---

## Known limitations

1. **No server synchronisation:** data is tied to the current browser profile unless exported.
2. **Last-write-wins cross-tab behaviour:** concurrent edits are not merged semantically.
3. **No snapshot restore interface:** IndexedDB snapshots are created and counted, but this build does not expose a UI for selecting and restoring one.
4. **No background scheduler:** notification permission is requested, but the application does not run a service worker or deliver scheduled due-date alerts while closed.
5. **Focus completion only:** notifications are currently used for completed focus intervals, not persistent background reminders.
6. **Dependency editing is absent:** the action model supports dependencies, but the action editor currently stores an empty dependency array.
7. **No drag-and-drop career movement:** opportunity stages advance through a button rather than native Kanban dragging.
8. **Graph relationships are heuristic:** edges are generated by position in the project, interest and responsibility arrays rather than user-authored semantic links.
9. **Worker fallback is limited:** the current build assumes Web Worker support for ranking and search.
10. **No automated test suite is embedded:** behaviour should be tested after structural edits.
11. **No installable PWA layer:** the file is offline-capable because it is self-contained, but it does not include a manifest or service worker.
12. **Seed records are source-visible:** deleting records in the UI does not remove the original default values from the HTML source.
13. **Simulation is heuristic:** Monte Carlo output is an exploratory estimate, not a statistically calibrated forecast.
14. **No repository or calendar integration:** project, job and study data must currently be entered or imported manually.

---

## Troubleshooting

### Data does not persist

Possible causes:

- Private browsing restrictions
- Browser storage disabled
- Storage quota exhausted
- Local-file restrictions
- Security software clearing site data

Run the app from `http://localhost` and check the Systems Lab storage status.

### The app shows “Memory-only mode”

`localStorage.setItem()` failed. Export any important current data immediately and reopen the app in a context where persistent storage is available.

### Encrypted backup import fails

Check that:

- The file has a `.nexus` extension.
- The header is `NEXUS3`.
- The exact passphrase is used.
- Web Crypto is available.
- The file has not been modified or truncated.

AES-GCM authentication intentionally rejects incorrect passwords and altered ciphertext.

### Gzip import fails

The browser may not support `DecompressionStream`. Import the same workspace from a plain JSON export, or decompress the file externally before import.

### Speech capture does not start

Speech Recognition availability is browser-dependent and can require:

- HTTPS or localhost
- Microphone permission
- A supported browser engine
- An online browser speech service, depending on implementation

Manual action capture remains available.

### Picture-in-Picture is unavailable

Document Picture-in-Picture is experimental. Use the main focus dialog instead.

### The worker is blocked by a hosted CSP

Allow an appropriate `worker-src` policy for Blob URLs, or move the worker code into a separate JavaScript file and load it from an approved origin.

### Reset does not use newly edited seed data

The browser is probably loading previously persisted state. Use **Reset workspace**, clear site storage, or change `APP_KEY`/`SCHEMA` during development.

---

## Development checklist

Before releasing a modified build:

- [ ] Validate HTML structure.
- [ ] Run JavaScript syntax checks.
- [ ] Test first launch with empty storage.
- [ ] Test migration from an older state.
- [ ] Test create, edit, complete and delete operations.
- [ ] Test worker ranking and fuzzy search.
- [ ] Test import/export for JSON, gzip and encrypted files.
- [ ] Test incorrect encrypted-backup passwords.
- [ ] Test storage failure behaviour.
- [ ] Test dark, light and system themes.
- [ ] Test privacy blur.
- [ ] Test keyboard navigation and `Ctrl/Cmd + K`.
- [ ] Test mobile navigation.
- [ ] Test reduced-motion mode.
- [ ] Test cross-tab updates.
- [ ] Test graph dragging and resizing.
- [ ] Test focus pause, completion and wake-lock release.
- [ ] Remove private seed records before public distribution.
- [ ] Add a suitable licence.

---

## Suggested future work

### Architecture

- Add JSON Schema validation and versioned migrations.
- Add a proper IndexedDB snapshot browser and restore workflow.
- Introduce CRDT-based cross-tab and multi-device merge semantics.
- Provide a main-thread fallback when Worker creation fails.
- Split optional modules into dynamically loaded components while preserving an exportable single-file build.
- Add deterministic unit tests for scoring and migration.

### Planning intelligence

- Add editable action dependencies.
- Add deadline confidence and uncertainty ranges.
- Add calendar-aware capacity.
- Add interruption-cost modelling.
- Add recurring actions and routines.
- Add explainable score decomposition per action.
- Add Pareto-front analysis for project selection.
- Add deterministic seeded simulations for reproducibility.

### Portfolio and career

- Import repository metadata from GitHub exports or an explicitly authorised integration.
- Generate evidence matrices from screenshots, commits and benchmarks.
- Track application deadlines, contacts and follow-up dates.
- Produce role-specific project ordering and proof packs.
- Add CSV import/export for opportunities.

### Study

- Add module deadlines and dependency maps.
- Track software installation and licence readiness.
- Add revision sessions and spaced-repetition prompts.
- Add timetable import without exposing private data to a server.

### Platform

- Add an optional PWA manifest and service worker.
- Add installable shortcuts.
- Add local notification scheduling where platform support permits it.
- Add OPFS storage for larger local evidence files.
- Add File System Access directory linking for project artefacts.
- Add WebGPU-accelerated graph layout or simulation experiments.

---

## Licence

No software licence is included with the current file. Add an explicit licence before redistribution or public collaboration. Until then, normal copyright restrictions apply.
