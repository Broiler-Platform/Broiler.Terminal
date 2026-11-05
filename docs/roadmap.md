# Broiler.Terminal roadmap

**Status:** Proposed. No runtime implementation exists yet.  
**Planning rule:** Phases have evidence-based exit gates, not calendar promises.

This roadmap delivers a terminal-output host for Broiler.UI first. Child-process/remote
terminal emulation follows as a separate product slice after the attached-terminal model
has proved itself.

## Phase 0 — decisions, fixtures, and repository scaffold

### Deliverables

- Approve the split between attached TUI endpoint and future emulator connection.
- Record ADRs for dependency/package topology, cell/Unicode model, lifecycle and
  ownership, internal driver/session actor, VT/capability policy, security limits, and
  Broiler.UI/Input integration.
- Resolve the open decisions in the [design draft](design-draft.md#open-decisions-for-phase-0):
  package/type names, Testing publication, exact platform matrix/budgets, internal friend
  SPI enforcement, integration type names, and attached-presenter threat model.
- Scaffold:
  - `Broiler.Terminal`;
  - `Broiler.Terminal.Windows`;
  - `Broiler.Terminal.Linux`;
  - neutral/Windows/Linux test runners;
  - Windows and Linux diagnostic demos;
  - solution configurations, packaging metadata, test script, CI, publish dry run, and
    revision-scoped `HUMAN_REVIEW.md`.
- Commit protocol fixture format, frame snapshot format, diagnostics schema, support
  matrix schema, requirement-applicability table, and a public feature/conformance table.
- Adopt the normative [Preview 1 profile](profiles/preview-1.md), pin generated Unicode data,
  and document how profile/data revisions are hashed, updated, and reviewed.
- Freeze exact mandatory versus exploratory Windows/Linux/frontend/SSH rows, representative
  pairwise coverage, artifact/skip policy, and support duration; placeholders such as
  “current,” “common,” and “selected” do not pass this phase.
- Freeze workload profiles and budgets for frame/diff allocation and latency, input-to-
  present latency, write-queue pressure, diagnostic pressure, cleanup deadlines, and soak.
- Record the cross-repository migration ADR: awaited UI presentation, typed Input proxy
  devices and paste event, host-focus/caret paths, metrics/theme leases, Standard theme
  package ownership, sensitive-content responsibility, and projector decision gate.
- Establish a clean sibling baseline before projector work. Broiler.UI MUST pin a Graphics
  revision containing every command its checked-in controls use, including `FillTriangle`.
- Complete the attached-presenter threat model and review the closed diagnostics schema.

Initial budgets are measured after warm-up on the recorded CI reference runner; Phase 0
may tighten them by ADR but may not leave them qualitative:

| Workload | Blocking budget |
| --- | --- |
| 80×24 unchanged interactive plan | p95 ≤ 1 ms, ≤ 16 KiB allocation, zero display bytes |
| 200×60 full ideal render + physical plan | p95 ≤ 16 ms, ≤ 32 MiB transient allocation |
| 1,000×1,000 limit frame | completes ≤ 2 s, process working-set increase ≤ 512 MiB |
| 120×40 input-to-present under an idle local driver | p95 ≤ 33 ms excluding frontend display time |
| Slow writer | at most two pending presentations; producer observes backpressure; memory remains within configured frame/update limits |
| Cleanup with one blocked driver step | step ≤ 2 s and total cleanup ≤ 10 s |
| 12-hour resize/input/present soak | no handle/fd growth; post-warm-up managed growth ≤ 2 MiB/hour |

### Exit gate

- Every dependency arrow for projects present in Phase 0 is enforced by architecture tests;
  conditional Testing and future Input/UI arrows are added to the gate when those projects
  are introduced.
- Core compiles as safe, trimming/AOT-compatible `net10.0` with no native imports.
- All provisional public types have an owner, lifecycle, thread model, and limits.
- The application-facing API exposes no raw active-session byte writer, and model tests pin
  the complete session state/generation transition table.
- No package or README claims generic Unix, SSH-client, xterm, accessibility, or emulator
  support merely from the planned architecture.

## Phase 1 — deterministic cell core and oracle

### Deliverables

- Integer cell geometry, default/indexed/RGB colors, styles, link identities, cursor, and
  surface/frame descriptors.
- Pinned extended-grapheme segmentation and terminal-width profile, including explicit
  ambiguous and emoji width policy.
- Cell storage with wide-leading/continuation invariants and bounded grapheme interning.
- Single-writer `TerminalRenderListBuilder` and immutable, atomically frozen/validated
  `TerminalRenderList`.
- `TerminalGridRenderer` reference replay into immutable `TerminalFrame` snapshots.
- Ideal-frame oracle and canonical snapshot serialization/profile hashes.
- Scripted frame inputs; no endpoint, physical diff, or VT frontend oracle in this phase.
- Typed faults, limits, sanitized diagnostics, and injected monotonic clock.

### Verification

- Golden frames cover fill, text, clipping, cursor, overflow, style, and resize.
- Unicode corpus covers combining marks, emoji/ZWJ, flags, variation selectors, CJK,
  ambiguous width, bidi preservation, ill-formed UTF-16 policy, and pathological clusters.
- Property tests cannot create an orphan wide cell under arbitrary write/clip/resize.
- Same inputs/profile always serialize to the same ideal snapshot.
- Memory is bounded by aggregate command/text/grapheme/style/link/frame/metadata limits,
  not only visible-cell and per-item limits.
- Phase 0 frame/allocation budgets pass for the representative size/style workloads.

### Exit gate

Every ideal-frame behavior is testable without Windows, Linux, a real terminal, byte I/O,
or wall-clock sleeps. The grid oracle is the sole normative reference for ideal cells; it
is not called the VT/frontend oracle.

## Phase 2 — VT presentation and frontend input protocol

### Deliverables

- Capability-aware full/sparse physical diff planner with the profile's deterministic byte
  cost/tie rules and one owner for committed/unknown physical state.
- UTF-8 VT output encoder for the enumerated ECMA-48/DEC/xterm baseline.
- Deterministic color/style/cursor/erase downgrade by capability profile.
- Correct autowrap and bottom-right-cell strategy.
- Independent virtual frontend oracle that replays encoded updates without reusing the diff
  planner's state-transition implementation.
- Incremental Linux/stream frontend-input decoder for named keys, committed text, focus,
  SGR mouse, bracketed paste, and bounded unknown sequences.
- Escape ambiguity handled by configurable timeout and virtual clock.
- Typed capability resolver with explicit override, conservative environment/terminfo
  fact interfaces, optional bounded probes disabled by default, unsafe assumption
  diagnostics, and application policy as the final clamp. Linux fact collection remains a
  Phase 4 backend responsibility.
- Preview 1 `Alternate` mode plus mandatory Plain renderer; `Inline` remains unsupported
  until Phase 6.
- Structural separation of user text from control/title/link operations.
- Internal scripted driver, independent read/write pumps, nonblocking session actor,
  generation/stale-frame handling, cleanup deadlines, and the single ordered session-event
  stream. No application-facing raw driver is exposed.

### Verification

- Replay every encoded update in the independent frontend oracle and compare the final
  physical model with the downgraded target frame.
- Split every timestamped input fixture at every byte boundary and get identical events;
  timing advances and EOF flush are explicit fixture dimensions.
- Fuzz UTF-8, CSI/OSC-like input, timeouts, paste, control injection, limits, cancellation,
  and partial writes.
- Fault injection after any output prefix marks physical state unknown; recovery begins
  with a reset/full repaint.
- Plain default output contains only valid UTF-8 graphic text and LF, emits each requested
  complete snapshot, and contains no other C0/C1/control bytes.
- Paste overflow discards safely to its delimiter; link/title metadata cannot inject an OSC
  terminator; public APIs cannot obtain a raw active-session writer.
- Phase 0 protocol, latency, allocation, queue, and cleanup budgets pass.

### Exit gate

The protocol layer is safe and deterministic over scripted streams. No platform backend
is permitted to bypass it with its own layout/diff semantics. The attached-presenter threat
model receives its required post-protocol security review before platform Preview work.

## Phase 3 — attached Windows backend

### Deliverables

- Detect console, VT-capable stream, and redirected handle cases.
- Scoped capture/change/restoration of output/input modes and any encoding state.
- VT-first output shared by Windows Terminal and Console Host.
- Native console input records where available, preserving resize, mouse, key transition,
  repeat, scan code, and committed character fidelity through the internal typed ingress
  union rather than a lossy UTF-8 endpoint.
- Cancellation-safe native read/write loops, viewport-size events, suspend/resume, and
  sanitized runtime diagnostics.
- Windows diagnostic TUI with frame, input, resize, color, width, cleanup, and plain-mode
  probes.

### Matrix and exit gate

- Every mandatory exact Windows/runtime/x64/Arm64 row frozen by Phase 0.
- The exact Windows Terminal and Console Host builds named in that manifest.
- Launch from cmd and PowerShell.
- Windows OpenSSH session with a PTY and without a PTY.
- Resize/selection and externally induced suspend/resume storms, Ctrl+C policy, every mixed
  redirected stdin/stdout case, broken pipe,
  cancellation at every open/close step, and repeated open/close soak.
- No leaked handles, hung input thread, unexpected code-page/mode change, or uncontrolled
  output after disposal.

ConPTY child-process support is not part of this gate.

## Phase 4 — attached Linux backend

### Deliverables

- `isatty`, scoped `termios` raw-mode lease, UTF-8 policy, and exact restoration.
- `winsize`/`SIGWINCH` integration without unsafe managed signal callbacks.
- Linux terminfo/environment capability resolver and conservative missing/corrupt fallback.
- Linux collects device/locale/terminfo facts; Core owns typed precedence, unsafe
  assumptions, policy intersection, and diagnostics.
- Async read/write/cancellation strategy with bounded queues and diagnostics.
- Linux diagnostic TUI matching the Windows probe surface.

### Matrix and exit gate

- Every mandatory exact glibc/musl distribution/runtime/x64/Arm64 row frozen by Phase 0.
- The exact Linux virtual-console and terminal-emulator builds named in that manifest.
- `tmux` and GNU screen, with the capability profile each actually exposes.
- Linux OpenSSH session with a PTY and without a PTY.
- Resize/signal/suspend storms, EOF, redirected handles, malformed terminfo, non-UTF-8
  locale policy, cancellation at every open/close step, and repeated open/close soak.
- No leaked file descriptors, handler registrations, changed `termios`, or unbounded queue.

This gate earns “Linux,” not “Unix,” support.

Phases 3 and 4 can be implemented in parallel once Phase 2 is stable.
Backend “suspend/resume” here means detecting externally imposed OS/job-control state,
invalidating the generation, and restoring safely on resume. A library command that
intentionally suspends to a shell remains Phase 6.

## Phase 5 — Broiler.Input and Broiler.UI integration; Preview 1

This phase changes the Broiler.Input, Broiler.Graphics, and Broiler.UI repositories as
integration work; it adds none of those dependencies to Terminal core. Phase 5A may start
after Phase 2 and one attached backend pass. Phase 5B may develop against that backend, but
the Preview 1 exit gate requires both Phase 3 and Phase 4.

### Phase 5A — narrow bridge and mandatory decision

- Add `Broiler.Input.Terminal` as a coordinator over existing Keyboard, Mouse, and Text
  packages. It exposes typed non-owning proxy devices sharing one Terminal session; it does
  not add a universal input-device base class.
- Add a marked paste event to Broiler.Input.Text and a direct UI-host activation path for
  terminal focus. Preserve session timestamps/sequence numbers, stable proxy device IDs,
  separate key/text paths, and untrusted paste provenance.
- Convert legacy terminal keypresses into documented synthetic balanced transitions when
  key-up is unavailable. Retain real transitions only from native Windows records in
  Preview 1; do not claim an unnamed enhanced keyboard protocol.
- Add an awaited Broiler.UI render/present path. Existing synchronous GUI hosts remain
  supported; the Terminal host never blocks a dispatcher on async I/O and never
  fire-and-forgets presentation.
- Add `Broiler.UI.Terminal` for the host/projector and
  `Broiler.UI.Terminal.Standard` for the optional grid-aligned Standard theme.
- Add exclusive Graphics text-provider and UI.Standard theme leases that capture/restore
  prior state and reject overlapping incompatible leases. Use one fixed logical cell
  rectangle for measurement, viewport, rendering, caret, and pointer conversion.
- Implement `IUiTextInputHost` caret-to-Terminal-cursor mapping and injected clipboard
  policy. OSC 52 stays disabled. UI masks/redacts sensitive values before render/semantic/
  plain snapshots.
- Project complete UI frames; `TerminalSession`, not the UI adapter, exclusively owns
  physical downgrade, diff, commit, and recovery.

The Phase 5A fixture is Label, Button, CheckBox, Edit, ListView, ScrollView, Menu, and
Dialog. Its decision ADR accepts the projector only if all fixture snapshots/caret positions
match, no render command is silently dropped, the projector contains zero control-type/name
special cases, and every unsupported command has one general visible fallback. Otherwise
the ADR selects a presentation-neutral/cell-native UI render contract. Phase 5B cannot begin
until this ADR is approved.

### Phase 5B — selected bridge and control matrix

- Implement the selected rendering contract and deterministic tab traversal, focus-visible
  behavior, modal trapping, and terminal keyboard conventions required for keyboard use.
- Mandatory supported rows: Label; Button/ToggleButton/CheckBox/RadioButton; Edit for ASCII
  and approved grapheme/emoji/CJK cases; Panel/ScrollView/ListView/TreeView/Splitter/TabView;
  Menu/Tooltip/Dialog/logical Window; ProgressBar/Slider/ComboBox/SpinBox/Toolbar.
- Mandatory explicit limitation rows: ImageView (`[image]` until an alt-text side channel),
  CodeEditor, RichEdit, FormatCodeView, FileDialog, and FontDialog. Each row is classified
  supported, visible fallback, or blocked—never silently omitted from the published matrix.
- Audit AltGr, Ctrl+Alt, Escape timeout, signal policy, repeat, unavailable native fields,
  arrow-name aliases, paste confirmation, focus, wheel/mouse, resize generation, and
  UTF-16-code-unit caret/hit-test defects without inventing fidelity.

### Preview 1 exit gate

- The same mandatory UI fixture is usable by keyboard in every exact Windows/Linux/local/
  SSH matrix row frozen in Phase 0; optional native fidelity may differ, but normalized
  application behavior matches.
- Rendered ideal snapshots and Terminal-owned physical plans match the two distinct oracles
  under monochrome, 16, 256, and RGB profiles.
- Awaited presentation propagates backpressure, cancellation, stale-generation results,
  write faults, and disposal; no second physical diff model exists.
- No printable key is inserted twice; marked paste reaches policy before insertion; controls
  activating on key-up work with synthetic legacy transitions.
- The metrics/theme leases restore their predecessors and reject a second incompatible host. The
  Preview 1 limitation to one heterogeneous host policy at a time is explicit.
- Every mandatory control row passes and every limitation row has the declared visible or
  blocked behavior. The projector/render-contract ADR and all cross-repository pins identify
  exact revisions.

## Phase 6 — robust multi-host UI and terminal UX

### Broiler.UI prerequisites

- Make text measurement session/host-scoped instead of process-global.
- Make Standard theme/token resolution session/host-scoped.
- Complete and stabilize the rendering contract selected by the Phase 5A ADR; do not reopen
  the decision implicitly through control-specific patches.
- Do not create a parallel terminal implementation of every control unless the evidence
  shows a genuine behavioral need.
- Extend the Preview 1 tab/focus/modal conformance to multi-host and dynamic-host cases.

### Terminal UX

- Specify and add `Inline`: region reservation/origin, scrolling, resize/reflow, external
  writer invalidation, minimum dimensions, and final cursor placement are normative.
- Add safe suspend-to-shell/re-entry only with exact mode restoration.
- Add copy/selection conventions, local-selection override while mouse reporting is
  active, and bounded paste policy hooks.
- Add deterministic reduced-color/high-contrast themes, ASCII border fallback, and
  resize layouts down to documented minimum dimensions.
- Define a logging/status integration that does not corrupt a full-screen session.

### Accessibility gate

- Test stable cursor/edit behavior, keyboard-only use, high contrast, reduced repaint
  noise, and optional linear mode with named screen readers and terminal frontends.
- Record limitations of alternate-screen applications and the impossibility of sending a
  rich semantic tree through generic VT.
- Redact private controls from every plain/diagnostic/accessibility path.

### Exit gate

- Simultaneous GUI and terminal UI sessions use different metrics/themes without global
  state collision.
- Every selected rendering-contract migration and multi-host tab/focus/modal test passes.
- `Inline` reservation, resize, external-output recovery, and final-cursor fixtures pass.
- Suspend-to-shell restores every mode under injected failures and re-entry forces a new
  generation/full frame.
- Mouse-selection override, paste bounds, reduced-color/high-contrast/ASCII fallback,
  documented minimum dimensions, and full-screen-safe logging each have observable tests.
- Every accessibility statement is backed by named revision-scoped frontend/screen-reader
  evidence rather than inference.

## Phase 7 — SSH-hosted and compatibility hardening

Running under SSH is already exercised in the backend phases; this phase widens and
stresses the claim rather than adding an SSH dependency.

### Deliverables

- Ephemeral Linux and Windows OpenSSH test servers in an isolated CI/integration setup.
- PTY/no-PTY, resize propagation, high latency/jitter, Unicode, color downgrade, signal,
  abrupt disconnect, reconnect-as-new-session, and plain-output cases.
- Terminal/frontend matrix expanded with the most important DEC/xterm profile variants,
  `tmux`, screen, and terminal capability lies/quirks.
- Published feature table and compatibility deviations tied to exact versions.
- Parser/encoder fuzz corpus, dependency/native scan, handle/fd/memory leak runs, and
  long-duration input/resize/presentation soak.

### Exit gate

- SSH execution needs no special application package and produces the same ideal frames
  and normalized portable application behavior as local execution under the same effective
  profile. Native transitions, scan codes, repeat detail, and byte-arrival timing are
  explicitly excluded from equality.
- Disconnect and no-PTY behavior are explicit and bounded.
- The release does not advertise xterm/ANSI/SSH/Unix compatibility beyond enumerated
  evidence.

## Phase 8 — first stable attached-TUI release

### Deliverables and gate

- Freeze public names after Terminal, UI, Input, and at least one application consumer
  review.
- Add package validation against the last published preview/stable baseline; when no prior
  package exists, record the exact bootstrap API baseline used by future comparisons.
- Validate clean NuGet consumption, trimming, Native AOT where claimed, symbols, XML docs,
  SourceLink, deterministic build, license/provenance, and package dependency graph.
- Complete security review of untrusted bytes, capability spoofing, control injection,
  paste, native interop, cleanup, diagnostics, and resource limits.
- Complete attributable human review for the exact revision and archive its artifacts.
- Every mandatory stable-claim matrix row MUST pass. Exploratory/deferred rows may be
  explicitly documented but cannot satisfy or weaken a mandatory claim.

The result is a stable attached-terminal presentation component and Broiler.UI host. It
does not silently acquire an embedded-emulator claim.

## Future scope outline — embedded TerminalView (separate approval)

This is not an active evidence-gated phase. Begin only after a dedicated specification and
threat model are approved; that specification MUST replace this outline with separate gates
for emulator core, each local connection, UI/accessibility, remote connections, and final
compatibility/security hardening.

### Neutral emulator core

- Separate `ITerminalConnection`, serialized emulator session, always-draining reader,
  streaming application-output parser, screen/history model, stable logical line IDs,
  soft wraps, versioned snapshots, and coalesced damage.
- Enumerated ECMA-48/DEC/xterm modes, including main/alternate buffers, margins, tabs,
  autowrap, style/colors, cursor, application input modes, mouse, focus, bracketed paste,
  title, and OSC 8 links.
- Explicit limits for scrollback rows/bytes, queues, sequence payloads, links, titles,
  replies, and action rate.
- Default-denied clipboard, notification, image, shell-integration, file-transfer, and URL
  side effects.

### Local connections

- Windows ConPTY connection with synchronous-pipe pumps, exact handle order, resize,
  output drain, multi-client/EOF lifecycle, and Win32-input-mode fidelity.
- Linux PTY connection with an audited native spawn/session/`execve` shim; no managed
  post-`fork()` child path.
- macOS/BSD implementations only after their independent ABI and behavior matrices.

### Broiler.UI.TerminalView

- One abstraction assembly and one `.Standard` implementation, following UI topology.
- Cell rendering through Broiler.Graphics, selection, search, scrollback, links, cursor,
  IME committed-text path, clipboard host ports, resize/reflow policy, and local-selection
  override.
- Bounded versioned accessibility text ranges and named UIA/AT-SPI/screen-reader evidence;
  never one accessibility object per cell.

### Remote connections

- First integrate the system `ssh` executable through the local PTY/ConPTY path so user
  known-hosts, agent, credentials, and configuration remain authoritative.
- Consider an in-process `Broiler.Terminal.Ssh` only after dependency, cryptography, AOT,
  host-key, credential, forwarding, and maintenance review.
- Model RFC 4254 PTY request, terminal modes, window change, EOF, exit status/signal, and
  disconnect distinctly.

### Eventual approval gates

- Byte-split, parser-fuzz, vttest/ncurses/editor/shell, ConPTY/PTY lifecycle, SSH trust,
  scrollback bounds, selection/search, accessibility, leak/deadlock, and soak matrices all
  pass for the exact claimed profile.
- The embedded emulator ships under an explicit compatibility profile; it is not described
  simply as “xterm compatible.”

## Cross-cutting risk register

| Risk | Leading indicator | Required response |
| --- | --- | --- |
| Unicode width drift | Oracle differs from real cursor position | Pin/update profile, record host quirk, full repaint; never font-measure cells |
| Capability overclaim | Unsupported sequence appears literally or input mode stalls | Downgrade typed profile, add matrix fixture, keep active probes optional |
| Partial output desync | Sparse updates corrupt later frames | Mark unknown and reset/full repaint after any incomplete write |
| Public raw-driver escape | Application writes unvalidated control bytes | Keep driver friend-internal; application receives only `TerminalSession` |
| Mode leakage | Cursor hidden/raw mode after exit | Fault injection at every acquisition, reverse cleanup, backend soak |
| UI projection heuristics spread | Per-control special cases grow | Fail Phase 5A projector gate and select the neutral/cell-native contract |
| Sync UI/async terminal mismatch | Blocked dispatcher or lost write fault | Awaited UI present seam with bounded backpressure and disposal tests |
| Global UI provider collision | GUI metrics/theme change when TUI starts | Session-scope providers before heterogeneous-host claim |
| Key/text duplication | One printable press inserts twice | Separate event paths and adapter conformance tests |
| Paste/focus provenance lost | Paste bypasses policy or focus mutates element focus | Marked paste type and distinct host-activation path |
| Parser/control injection | User text changes title/clipboard/modes | Structural text/control APIs, limits, fuzzing, side effects denied |
| SSH scope creep | Host-key/credential code appears in core | Keep launched-under-SSH path; system ssh first; separate reviewed adapter |
| Unix overclaim | Linux struct/ioctl copied to another OS | Separate backend and evidence per OS/ABI |
| Slow embedded UI deadlocks child | PTY output stops when render stalls | Always-draining reader and bounded damage coalescing |
| Accessibility overclaim | No named terminal/screen-reader evidence | Publish limitation, add explicit matrix before claim |

## Definitions of done

- **Decision/ADR:** alternatives and evidence are recorded; owner, consequences, migration,
  compatibility, security, and supersession rule are explicit; affected specifications and
  gates are updated.
- **Implementation:** behavior plus negative/error paths, ownership, cancellation, limits,
  diagnostics, XML/API documentation, deterministic tests, and applicable performance/
  platform evidence are complete.
- **Fixture/evidence:** format/version/provenance and expected result are pinned; artifacts
  are reproducible, linked to an exact revision, and classified mandatory or exploratory.
- **Release artifact:** clean-consumer/package, API baseline, provenance/license, matrix,
  threat/security/human review, and support claims all refer to the exact shipped revision.

A demo that works once is discovery evidence, not an exit gate.
