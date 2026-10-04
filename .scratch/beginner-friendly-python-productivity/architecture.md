# Tracker architecture and event flow

This is the proposed first-version architecture. Boxes labelled as modules and commands describe future application components; they have not been implemented. Authoritative external setup facts are linked from the existing hardware and publishing guides.

## System context

```mermaid
flowchart TD
    Student["Student"] -->|"Presses"| Button["AIY Voice HAT V1 button"]
    subgraph Desk["Desk: Raspberry Pi 3 Model B+"]
        App["Python tracker process"]
        DB[("SQLite on microSD: authoritative history")]
        WAV["Python-generated tone WAV files"]
        Speaker["Voice HAT V1 speaker"]
        Output["Complete static dashboard snapshot"]
        Publisher["Python publisher and local Wrangler"]
        App --> DB
        App --> WAV
        WAV --> Speaker
        App --> Output
        Output --> Publisher
    end
    Button --> App
    Speaker -->|"Action feedback"| Student
    Publisher -->|"Outbound HTTPS Direct Upload"| Pages["Cloudflare Pages"]
    Pages --> Browser["iPhone Safari or laptop browser"]
    Student -->|"Views snapshot"| Browser
    Output -.->|"If Pi publishing is incompatible"| Laptop["Laptop with pinned Wrangler"]
    Laptop -.->|"Same Pages project"| Pages
```

Only the Pi stores the authoritative database. Pages stores published browser assets, with the intended history embedded in them. Browser requests go to Cloudflare, not to the Pi. The laptop path is a publication fallback, with tracking still on the Pi.

## Module boundaries

```mermaid
flowchart TD
    CLI["cli.py: run, simulate, recover, publish, backup, export"] --> Config["config.py: validate non-secret settings"]
    CLI --> Input["button.py: GPIO or simulated input"]
    Input --> Gestures["gestures.py: debounce and monotonic tap recognition"]
    Gestures --> Session["session.py: exact state rules and elapsed timing"]
    Session --> Storage["storage.py: SQLite transactions and checkpoints"]
    Storage --> DB[("Local SQLite")]
    Session --> Audio["audio.py: ordered cue queue and ALSA playback"]
    Session --> PublishRequest["Persist request for a newer publication"]
    PublishRequest --> DB
    DB --> Dashboard["dashboard.py: consistent read, daily splitting, HTML/CSS/SVG"]
    Dashboard --> Snapshot["Immutable complete output directory"]
    Snapshot --> Publisher["publishing.py: one worker, retries, Wrangler subprocess"]
    Publisher --> DB
    Publisher --> Pages["Cloudflare Pages"]
    Config -.-> Input
    Config -.-> Gestures
    Config -.-> Audio
    Config -.-> Dashboard
    Config -.-> Publisher
```

These names describe responsibilities, not a rigid requirement to create one file per box. Keep small functions and simple interfaces. Hardware adapters provide input facts; they do not decide which tap starts or ends a session. Simulation exercises the same gesture and session logic.

## Runtime ownership and responsiveness

One tracking loop owns the session state and orders confirmed gestures. GPIO callbacks enqueue timestamped edges quickly. They do not play audio, render pages, or start uploads. Gesture scheduling uses monotonic time and never waits for an audio or deployment worker.

SQLite writes are short, immediate transactions. Commit the accepted action before updating the visible in-memory state or enqueueing the confirmation cue. If storage fails, do not signal a saved action or silently advance state: report the fault and stop accepting new session mutations until persistence is available. Audio and deployment failures cannot roll back a committed action.

Use one ordered audio worker and one publication worker. The latter serializes dashboard generation and uploads and can read a consistent database snapshot without holding a write transaction through rendering or network calls. Avoid a worker or connection per tap. Each SQLite connection belongs to its owner thread; use short transactions and busy timeouts rather than sharing a connection unsafely.

## Session state machine

```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> IDLE: single tap / ignore
    IDLE --> STUDYING: double tap / start
    STUDYING --> ON_BREAK: single tap / pause
    ON_BREAK --> STUDYING: single tap / resume
    STUDYING --> IDLE: double tap / end
    ON_BREAK --> IDLE: double tap / close break and end
```

Recovery is an operational gate, separate from these three user states. An unresolved interrupted session prevents a new start; it does not introduce a fourth button state. [Recovery lifecycle](data-and-recovery.md#startup-and-recovery) explains how the process reaches a trustworthy state before enabling gestures.

## Gesture recognition

```mermaid
flowchart TD
    Edge["Timestamped GPIO edge or simulated edge"] --> Debounce["Apply configurable debounce; require press and release"]
    Debounce --> Valid{"New accepted press?"}
    Valid -->|"No"| Ignore["Ignore bounce or held-button repeat"]
    Valid -->|"Yes"| Pending{"First tap pending?"}
    Pending -->|"No"| Arm["Store first press time; arm 400 ms deadline"]
    Arm --> Wait["Continue processing input; no action yet"]
    Pending -->|"Yes"| Inside{"Second accepted press within deadline?"}
    Inside -->|"Yes"| Double["Cancel pending single; emit exactly one double tap"]
    Inside -->|"No"| SingleThenArm["Emit expired single; start a new pending tap"]
    Wait --> Expiry{"Deadline reached with no second press?"}
    Expiry -->|"Yes"| Single["Emit exactly one single tap"]
    Expiry -->|"No"| Wait
```

Press timestamps use the monotonic clock. Define the boundary as a second accepted press at or before the deadline producing a double tap. Process queued edges through that deadline before resolving its timer, so event-loop scheduling does not change recognition. Events later than the deadline start another tap after the expired single is dispatched. A confirmed double consumes its two taps; a third tap starts a new candidate.

The recognizer must see release before another press is eligible. Debounce and double-tap detection are separate concerns. A configurable initial debounce proposal is 50 ms; the double-tap default is the required 400 ms. Automated boundary and bounce tests use a fake clock.

## What a button action does

```mermaid
sequenceDiagram
    actor Student
    participant GPIO as Button adapter
    participant G as Gesture recognizer
    participant S as Session engine
    participant DB as SQLite
    participant A as Audio worker
    participant P as Publication worker
    Student->>GPIO: Press and release button
    GPIO->>G: Debounced input with monotonic timestamp
    G->>G: Resolve single or double after window rules
    G->>S: One confirmed gesture
    S->>S: Validate transition and calculate known elapsed time
    S->>DB: Transaction: event, interval, session totals, revision
    opt End action
        S->>DB: Same transaction: request publication of new revision
    end
    DB-->>S: Commit successful
    S->>S: Adopt committed state
    S->>A: Enqueue action cue immediately
    A->>A: Play in order, log failure without undoing event
    opt End action
        S->>P: Wake background publication worker
        P->>DB: Read consistent latest snapshot
        P->>P: Generate full output and deploy serially
    end
    Note over GPIO,S: Further input remains responsive during audio and upload
```

The ignored IDLE single tap exits before any transaction or cue. For end-on-break, the transaction closes the break and session together. The end cue is scheduled before rendering/upload work begins; the loop does not wait for the speaker to finish.

## Audio design

```mermaid
flowchart TD
    Commit["Committed start, pause, resume, or end"] --> Cue["Map action to distinct tone pattern"]
    Cue --> Queue["One FIFO audio queue"]
    Queue --> Worker["One background playback worker"]
    Worker --> Device["aplay subprocess with configured ALSA device"]
    Device --> Done{"Playback completes within timeout?"}
    Done -->|"Yes"| Next["Play next queued cue"]
    Done -->|"No"| Fault["Stop failed playback; record error; continue queue"]
    Fault --> Next
```

Generate reusable WAV files at setup with Python's standard library and a configurable amplitude. Use a V1-compatible output format and explicitly selected ALSA device. No two cues play together. Playback has a finite timeout so a hung subprocess cannot block all later cues. Tracking data remains valid if the queue or device fails; cues do not survive a reboot as a promise of new actions. [V1 audio checks](voice-hat.md) document the hardware-specific evidence.

## Dashboard structure

```mermaid
flowchart TD
    DB[("Consistent SQLite read at revision R")] --> Known["Extract known study and break intervals"]
    Known --> Split["Split at Asia/Kolkata midnight boundaries"]
    Split --> Daily["Per-day study, break, and session totals"]
    Daily --> View["Selected day summary and session timeline"]
    Daily --> Seven["Past seven calendar days, including zero days"]
    Known --> History["Complete retained session history and status labels"]
    Meta["Snapshot time and last confirmed publication metadata"] --> HTML["Self-contained HTML, CSS, SVG, minimal JavaScript"]
    View --> HTML
    Seven --> HTML
    History --> HTML
    HTML --> Dist["Complete immutable output directory"]
```

Default to the latest local day in the snapshot; allow selecting other available days. Show study and break durations distinctly. A session spanning midnight appears in each touched day's detail, with its full session endpoints clearly distinguished from that day's clipped contribution. Proposed daily session count is the number of distinct sessions with a known interval touching that day; the history table still counts each session once.

Uncertain recovery gaps are visibly separate and contribute to neither total. If manual publication includes an ongoing session, render only durations persisted through its checkpoint and label them incomplete/as-of that checkpoint. Never use a browser timer to make a static snapshot appear live. Include a clear snapshot notice and distinguish generation time from successful publication time as described in [publication architecture](publication-architecture.md#publication-timestamps).

Keep assets local, text readable without zooming, controls touch-friendly, and tables scrollable at narrow widths. Use semantic HTML and accessible labels for SVG charts; colour must not be the only distinction between study, break, and interruption. Escape any configurable text inserted into HTML. Do not embed tokens or database files.

## Verification plan

| Area | Required evidence |
| --- | --- |
| State rules | Every transition, ignored IDLE single, end while studying and on break |
| Gestures | Delayed single, only one action per double, exact window boundary, third tap, bounce, held press |
| Durations | Fake monotonic clock, active/break sums, multiple daily sessions, wall-clock changes |
| Calendar | Asia/Kolkata midnight splits, break crossing midnight, zero-activity dates |
| Persistence | Immediate event commit, atomic end-on-break, transaction failure, checkpoint durability |
| Recovery | Unfinished detection, close at checkpoint, resume with gap excluded, no reboot clock reuse |
| Audio | Ordered non-overlap, configured device/volume, failure and timeout isolation |
| Publication | Failed upload retains history/pending state, bounded retries, restart retry, coalescing, one uploader |
| Dashboard | Representative ended-on-break and midnight samples; iPhone-sized layout; credentials absent |
| Operations | Non-root Pi service, safe backup/export, real button/audio checks and ARM64 tooling checks |

Unit tests use fake clocks and mock input, audio, and upload calls, with no real sleeps, Pi, or credentials. Later browser checks inspect a generated sample snapshot. Pi and Cloudflare acceptance checks are recorded separately from mocks. [Current verification record](verification.md) lists only checks already performed.
