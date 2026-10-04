# Dashboard generation and publication architecture

This design keeps tracking local and makes Cloudflare Pages a published snapshot. No dashboard or Cloudflare deployment exists yet. See [publishing prerequisites](publishing.md) for verified CLI flags, proposed tool versions, and setup sources.

## Deployment topology

```mermaid
flowchart TD
    subgraph Pi["Pi: non-root tracker service"]
        DB[("SQLite history and publication state")]
        Worker["Python publication worker"]
        Snapshot["Complete immutable static snapshot"]
        Wrangler["Locally pinned Wrangler CLI"]
        DB --> Worker
        Worker --> Snapshot
        Snapshot --> Wrangler
        Env["Protected process environment: token and account ID"] --> Wrangler
    end
    Wrangler -->|"Outbound HTTPS upload"| CF["Cloudflare Pages production project"]
    CF --> Browser["Browser downloads HTML, CSS, SVG, and intended history"]
    Snapshot -.->|"Local-network copy when needed"| Laptop["Supported laptop: Python orchestration and pinned Wrangler"]
    Laptop -.-> CF
    Access["Cloudflare Access if private access is chosen and verified"] -.-> CF
```

Credentials are process inputs, never snapshot data. The browser needs no token and no connection to the Pi. The public GitHub repository contains source and documentation, not real session history. A private dashboard would require protecting every accessible hostname and history asset, not just its visible landing page.

## Durable publication state

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Pending: end action or manual publish request
    Pending --> Rendering: worker owns publication lock
    Rendering --> Uploading: complete snapshot ready
    Rendering --> RetryWait: generation failure
    Uploading --> RetryWait: upload failure or timeout
    RetryWait --> Pending: retry deadline reached
    Uploading --> Idle: acknowledged success and no newer requested revision
    Uploading --> Pending: acknowledged success with newer updates waiting
```

Persist requested and published dataset revisions, retry count/deadline, last confirmed success, and a sanitised error summary in SQLite. Every normal end transaction requests a publication revision before commit. A manual request updates the same durable state and wakes the same worker.

Only one process holds the publication lock. Normal tracking and CLI requests may enqueue updates concurrently, but cannot launch another upload. A standalone manual publisher may own that lock when the service is absent; it must refuse or just queue the request if another uploader owns it. Process-owned locks release after a crash; durable pending intent does not.

Metadata updates do not increment the dataset revision or trigger a fresh upload by themselves. Otherwise recording an upload success could create an endless publication loop.

## Coalescing and snapshot consistency

```mermaid
sequenceDiagram
    participant T as Tracking loop
    participant DB as SQLite
    participant P as Single publication worker
    participant D as Dashboard generator
    participant W as Wrangler
    participant CF as Cloudflare Pages
    T->>DB: Commit end and request revision 12
    T->>P: Wake worker without waiting
    P->>DB: Read latest consistent dataset and requested revision
    DB-->>P: Immutable input for revision 12
    P->>D: Generate complete history snapshot 12
    D-->>P: Immutable output directory 12
    P->>W: Start upload 12
    T->>DB: Commit another session and request revision 14
    Note over P,W: Do not change files used by upload 12
    W->>CF: Direct Upload complete snapshot 12
    CF-->>W: Successful deployment acknowledgement
    W-->>P: Success and deployment metadata
    P->>DB: Record published revision 12, revision 14 stays pending
    P->>DB: Read newest complete input
    DB-->>P: Revision 14 or newer
    P->>D: Generate newest full snapshot
    Note over P,CF: One upload at a time, intermediate revisions may be skipped
```

Render from a consistent SQLite read into a new output directory. Complete it before publishing. Do not mutate files while Wrangler reads them, or reuse an old partial output after failure. Expose the finished directory as local `dist/` if useful, but give each in-flight upload an immutable directory.

Every snapshot contains all retained intended history, including previously published dates. Coalescing skips obsolete requests rather than skipping historic data. If a new session is ongoing when the latest snapshot is read, include only checkpointed known time with an incomplete label.

## Failure, restart, and retry

```mermaid
flowchart TD
    Request["Durable pending publication exists"] --> Lock{"Publication lock available?"}
    Lock -->|"No"| Queue["Leave request queued; do not launch another process"]
    Lock -->|"Yes"| Build["Generate latest complete snapshot"]
    Build --> Valid{"Generation valid?"}
    Valid -->|"No"| Fail["Keep pending; log sanitised failure"]
    Valid -->|"Yes"| Upload["Run Wrangler with timeout and explicit project/branch"]
    Upload --> Success{"Success acknowledged?"}
    Success -->|"No"| Fail
    Success -->|"Yes"| Save["Persist successful revision and deployment metadata"]
    Save --> More{"Newer requested revision exists?"}
    More -->|"Yes"| Build
    More -->|"No"| Idle["Release lock and await request"]
    Fail --> Retry["Persist capped backoff and next attempt"]
    Retry --> Release["Release resources; tracker keeps accepting input"]
    Release --> Due["Wake when due; retry latest snapshot"]
    Due --> Request
    Restart["Process restart"] --> Recover["Treat abandoned in-flight work as pending"]
    Recover --> Request
```

Proposed automatic retry delays are 5, 10, 20, 40, 80, 160, then 300 seconds, capped at 300 thereafter. Respect server retry instructions when present and avoid shortening a rate-limit wait. A bounded delay, one worker, and coalescing keep offline retries controlled. Manual retry uses the same lock and must not circumvent an outstanding server rate-limit deadline.

Use monotonic deadlines during a running process. Persist UTC retry metadata for restart, but clamp the recovered delay when wall time is uncertain so a bad clock does not create a rapid loop or indefinite wait. Reset backoff after success. Credentials/configuration errors remain pending with a clear diagnostic and bounded retries; they never discard study data.

Limit the subprocess runtime, stop timed-out child processes, and collect only sanitised logs. Do not print the child environment or token-bearing request headers. When the Pi is offline, existing Pages content continues to serve. When the upload succeeds but the Pi crashes before recording success, repeat the newest complete snapshot after restart; do not mark it successful merely because an attempt started.

The laptop fallback can generate/copy and deploy the same output using the same CLI flags. A manual laptop upload is not a Pi-worker success until a verified receipt is reconciled with the local publication metadata. Without that reconciliation, leave the Pi request pending; a later repeat upload is safer than silently losing intent. Automatic post-session publication depends on verified Pi tooling or a separately designed laptop worker.

## Publication timestamps

A static artifact cannot know its own future upload completion time while it is being generated. Do not label a guessed start time or generation time as a successful publication timestamp.

The proposed first-version snapshot shows two explicitly different values: **snapshot generated at**, and **last confirmed successful publication before this snapshot**. After acknowledgement, SQLite stores the actual latest successful publication timestamp/deployment ID; a local status command can show it immediately. The next snapshot carries that verified result.

This honest display can lag the upload currently serving the page. Showing the exact latest success timestamp inside that same completely static page remains an open presentation decision. A follow-up upload introduces its own new completion time; it does not eliminate the circularity. Confirm the label/semantics before implementation instead of silently weakening the original dashboard requirement or adding a cloud database/live API.

## Visibility and release gate

```mermaid
flowchart TD
    Local["Generate sample-data local dashboard preview"] --> Review["Review iPhone/laptop layout and exported contents"]
    Review --> Choice{"User confirms dashboard visibility"}
    Choice -->|"Public"| Public["Confirm intended study history can be public"]
    Choice -->|"Private"| Private["Verify Cloudflare Access across production and deployment URLs"]
    Private --> Test["Test unauthenticated HTML and data requests are denied"]
    Test --> Protected{"All exposed paths protected?"}
    Protected -->|"No"| Fix["Fix access configuration before uploading real history"]
    Fix --> Private
    Protected -->|"Yes"| Publish["Publish to explicit Pages production project"]
    Public --> Publish
    Publish --> Verify["Verify deployed snapshot and record successful publication"]
```

Preview links themselves may be externally accessible; first review sample data locally. Do not assume that Cloudflare's preview protection covers the production hostname, a custom domain, or every deployment alias. Private access requires current official configuration research and real unauthenticated tests before uploading personal data. The decision to make the source repository public has already been made and is separate from this gate.

## Operating limits

Keep the dashboard below the verified Pages per-file/site limits documented in [publishing prerequisites](publishing.md#limits-to-design-around). When history grows, split self-contained history assets or pages while including all retained history in every upload. Do not silently truncate it. Backup and retention choices remain explicit future decisions.

Do not add a Pages Function, API, or cloud database to solve static rendering concerns without revisiting the first-version scope. The default architecture uses complete static assets and outgoing Direct Upload from Python orchestration.
