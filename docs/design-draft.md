# Broiler.Terminal design draft

**Status:** Proposed architecture  
**Draft:** 0.2  
**Date:** 2026-09-03

## Executive decision

Broiler.Terminal should be a sibling of Broiler.Graphics in architecture, not a clone of
its public API. The repository should own a deterministic, cell-native model and the
protocol/platform machinery required to present that model in an attached terminal.
Native and host-specific behavior remains in explicit backend packages.

The first supported product is **Broiler UI running in an existing terminal**. A later
**terminal emulator embedded in Broiler UI** is deliberately separated because its byte
flow, lifecycle, security boundary, scrollback model, and accessibility surface are the
opposite of the TUI presenter.

This decision gives the requested local Windows, local Linux, and SSH-hosted coverage
without making an SSH client, child-process host, or xterm emulator a prerequisite for a
usable Preview 1.

## What was learned from the sibling components

At the design date:

- Broiler.Graphics uses a safe `net10.0` core, pure ordered render data, a deterministic
  CPU oracle, backend-owned presentation, explicit package selection, and no meta-package.
- Its neutral surface and renderer contracts do not expose native handles.
- Broiler.UI references only the neutral Graphics and Input contracts. Its host owns the
  surface and presentation.
- Broiler.UI's current `IUiHost` is nevertheless graphics-specific: it creates and
  presents a `BRenderList`, while standard controls measure through Graphics text APIs.
- Broiler.Input owns normalized application input. Terminal escape decoding is a protocol
  concern, but mapping decoded terminal events into application input should not create a
  second UI input hierarchy.

The Terminal design copies the good boundaries and makes lifecycle/threading rules more
explicit. It does not copy Graphics' legacy Windows input/window ownership or its
currently inconsistent surface-disposal assumptions.

## Two product directions

### Direction A — TUI presenter (first)

```text
application / Broiler.UI.TerminalHost
            |
            v
TerminalRenderList --validate/replay--> TerminalFrame (ideal cell state)
            |                                  |
            |                         compare with committed frame
            |                                  v
session events <--- decoder <--- internal driver ---> VT diff encoder
                                                   |
                                                   v
                                     user's existing terminal frontend
```

The process is a terminal client. It receives user input and emits display operations.
Windows Terminal, Console Host, a Linux terminal, `tmux`, and an SSH client's terminal
are frontends around the process. The library does not render fonts or own pixels.

### Direction B — embedded emulator (later)

```text
child/remote process
  -> ConPTY / Unix PTY / SSH channel
  -> streaming application-output VT parser
  -> screen + bounded scrollback model
  -> immutable snapshot/damage
  -> Broiler.UI.TerminalView
```

This direction needs process/session control, an application-output parser, terminal
replies, scrollback and reflow, selection/search, and hostile-sequence policy. It may
reuse value types, Unicode data, limits, and test infrastructure from the core, but it
must not reuse the frontend-input decoder as though the two grammars had identical
semantics.

## Graphics equivalence

The intended equivalence is:

| Graphics concept | Terminal concept | Preserved property |
| --- | --- | --- |
| `BRenderList` and closed commands | `TerminalRenderList` and closed cell commands | Pure ordered data, validation, exhaustive replay |
| `BFrameContext` | `TerminalFrameContext` | Immutable per-frame policy and monotonic index |
| `IBroilerSurface` | `TerminalSession` over one internal driver | Explicit ownership, resize, disposal |
| `BImageRenderer` | `TerminalGridRenderer` plus virtual terminal oracle | Deterministic reference behavior |
| Backend renderer | Attached-console/TTY backend and VT presenter | Host-specific I/O outside the core model |
| CPU fallback | Conservative full repaint or explicit plain output | Correct fallback over silent feature invention |
| Device-loss signal | Broken pipe, disconnect, partial write, unknown state | Caller-observable recovery boundary |
| Driver diagnostics | Endpoint/profile/mode diagnostics | Actual path is observable |

The terminal equivalent is **not** a pixel surface. Coordinates are integer columns and
rows; cells have no alpha channel; font family/size and arbitrary transforms are not
portable terminal concepts.

## Layering and dependency direction

### This repository

```text
Broiler.Terminal.Windows -----> Broiler.Terminal
Broiler.Terminal.Linux  ------> Broiler.Terminal
Broiler.Terminal.Testing -----> Broiler.Terminal       (only if published)

Broiler.Terminal -------------> .NET base libraries only
```

The core must remain safe managed code, trimming/AOT compatible, and free of P/Invoke.
Backends may use narrow native interop. There is no runtime reflection-based backend
loader: the application chooses and constructs a backend explicitly.

The initial Linux name is intentional. A single “Unix” package would invite one Linux
`termios`/`ioctl` ABI layout to masquerade as portable. macOS and BSD support should be
added only with dedicated implementation and evidence; shared source can still be used.

### Cross-component integration

The proposed eventual dependency graph is:

```text
Broiler.Input.Terminal ---> Broiler.Input + Keyboard + Mouse + Text + Broiler.Terminal

Broiler.UI.Terminal ---> Broiler.UI (and its neutral Graphics dependency)
                     -> Broiler.Input.Terminal
                     -> Broiler.Terminal

Broiler.UI.Terminal.Standard ---> Broiler.UI.Terminal + Broiler.UI.Standard
```

`Broiler.Input.Terminal` is a coordinator over the existing typed Keyboard, Mouse, and
Text packages; it exposes typed proxy devices rather than a universal device hierarchy.
Terminal-frontend focus remains a UI host event. `Broiler.UI.Terminal.Standard` is optional
and owns the grid-aligned Standard theme; the base host does not silently depend on the
Standard control implementation.

The precise package names are provisional, but the arrows are not: the Terminal core
must not reference Graphics, Input, or UI. The Graphics-render-list projector is private
implementation inside the UI terminal host unless reuse evidence later justifies a
separate package; this avoids adding a Terminal dependency to Broiler.Graphics merely to
host Broiler.UI.

## Key design decisions

### Cell-native canonical state

The canonical rendered artifact is an immutable rectangular frame. Each visible glyph is
one extended grapheme cluster in a leading cell, followed by zero or one continuation
cell according to a session width profile. Style and hyperlink identity are cell data;
cursor is frame metadata and title is separately serialized session metadata.

This makes snapshot tests, diffing, selection, and future accessibility deterministic.
The physical terminal remains an eventually consistent, capability-downgraded projection
of the last committed ideal frame. Oracle equality covers cells and planned terminal
state, not font shaping, bidi display, or pixel-identical frontend rendering.

### Recorded commands plus an oracle

Higher layers record a small closed command set—clear/fill, text, clipping, and cursor.
The safe managed `TerminalGridRenderer` validates and replays it into a frame. Backends do
not reinterpret layout. This matches Graphics' render-list/oracle pattern and prevents
Windows and Linux from acquiring different cell semantics.

The presenter then plans a capability-aware diff from the committed physical model.
Planning and encoding are separately testable: a byte stream is fed to an independent
virtual terminal model and must produce the planned cell state.

### VT-first output, platform-appropriate input

VT/ECMA-48-style output is the common presentation language. Microsoft recommends VT
sequences over classic console output for cross-platform command-line applications, so
Windows Terminal and Console Host are variants of one Windows backend rather than two
packages.

Input is not forced through one lowest-common-denominator byte path. On an attached
Windows console, native input records retain resize, mouse, key-up/down, repeat, and scan
code information that a byte stream can lose. On Linux, raw TTY bytes are decoded through
the terminal input profile. Both become Terminal protocol events and are adapted to
Broiler.Input at the integration boundary.

The raw attached driver is an internal friend-assembly SPI, not a public endpoint. Its one
ordered ingress union carries byte chunks or neutralized native records plus resize, EOF,
and faults. The application sees only `TerminalSession`, so it cannot retain a raw writer
after ownership transfer or bypass text/control policy.

The common contract promises only information that terminal protocols can represent.
Richer Windows fields are optional fidelity, never required for portable UI behavior.

### Typed capabilities, conservative negotiation

Capabilities are an immutable typed set: color level, alternate screen, cursor controls,
mouse protocol, focus reporting, bracketed paste, synchronized output, hyperlink support,
and so on. Environment values and terminfo entries are inputs to a resolver, not the
public contract.

Resolution order is:

1. backend facts about the attached handle/device;
2. terminfo and conservative environment hints;
3. optional, bounded active-probe results;
4. typed `Auto`/`ForceOff`/explicit unsafe-assumption overrides; and
5. application feature permission as the final clamp.

Active probes are off by default because replies share the user-input stream and can be
spoofed or mistaken for keystrokes. A capability is enabled only when both the resolved
resolution (including a visibly diagnosed forced assumption) and application policy allow
it. `ForceOff`/denied policy always wins. Capability change increments the session
generation, invalidates the physical model, and forces a newly rendered full repaint.

### SSH is an environment first, a transport later

When a Broiler application is started through SSH with a PTY, RFC 4254 has already
carried the terminal type and dimensions to the remote PTY. The program uses the backend
for the remote OS exactly as if it had been started locally. A session without a PTY is
noninteractive and takes the explicit plain/redirection path.

Only software that initiates an SSH channel needs a `Broiler.Terminal.Ssh` adapter. That
future adapter should map an injected, authenticated SSH session to terminal connection
contracts. It must not implement cryptography, silently accept host keys, or bypass the
user's credential/agent policy.

### Explicit terminal modes and restoration

A backend factory opens one exclusive internal driver and returns a `TerminalSession` that
already owns it. Opening enters modes in a documented order; asynchronous disposal writes
conservative reset/show-cursor sequences, disables optional reporting modes, and restores
native modes in reverse order under configured per-step and overall cleanup deadlines.

Blocking read and write pumps run independently and post data/completion into one
nonblocking session actor. The actor alone mutates mode, decoder, capability, generation,
and physical presentation state; it never awaits driver I/O, diagnostics, or user code.

No finalizer writes to a terminal. Abrupt process termination can leave a terminal in an
altered state; documentation and an optional recovery command may mitigate that, but the
library cannot promise crash-proof cleanup.

### No raw escape hatch on the active presenter

Drawing text never copies C0/C1 controls, ESC, OSC, CSI, or DCS from user content into the
output stream. Title, hyperlinks, clipboard, and future extensions have distinct typed
APIs with validation, size limits, and feature policy. OSC 52 clipboard access is denied
by default. Unknown or unsupported input sequences produce bounded diagnostics rather
than unbounded buffering.

### Versioned Unicode behavior

Terminal width is not derivable from UTF-16 length, the selected font, or East Asian
Width alone. The core uses extended grapheme clusters, a pinned Unicode data version, a
pinned terminal-width profile, explicit one/two-column policy for ambiguous characters,
and test corpora for combining marks, emoji/ZWJ, flags, variation selectors, CJK, bidi
text, malformed UTF-8, and pathological cluster length.

The first implementation should pin Unicode 17.0.0 unless Phase 0 selects a newer fully
released version. A profile change is observable compatibility data and can require a
major version or an explicitly selected new profile.

## Broiler.UI terminal host

### Current constraint

Broiler.UI's current host contract creates and presents a Graphics `BRenderList`, its
layout coordinates are `BSize`/`BRect`, and standard controls call Graphics text
measurement directly. A terminal host cannot simply substitute a terminal frame without
changing that public boundary.

The mismatch is also asynchronous: `IUiHost.Present` returns `void`, while a terminal write
must be awaited for backpressure, commit, fault, and disposal ordering. Blocking the UI
dispatcher or fire-and-forgetting a terminal write is not an acceptable bridge.

### Recommended first bridge

The first vertical slice should keep the retained UI tree and introduce these explicit
pieces outside this core:

1. an awaited Broiler.UI render/present path, retaining the existing synchronous path for
   existing hosts;
2. a projector inside `Broiler.UI.Terminal` that consumes the closed Graphics command set
   and produces a Terminal render list/frame;
3. exclusively leased Terminal text-metrics and Standard-theme state plus an optional
   `Broiler.UI.Terminal.Standard` theme package that maps each cell to a fixed logical
   rectangle and keeps Standard control dimensions grid-aligned; and
4. a `Broiler.Input.Terminal` coordinator exposing typed keyboard/mouse/text proxies, a
   marked paste event, and a UI-host path for terminal focus.

Mouse coordinates convert from a zero-based cell to the corresponding logical-cell
rectangle. Text layout must use terminal grapheme advances; merely choosing a monospace
font does not solve emoji, combining, or ambiguous-width behavior. The UI work therefore
needs a host-provided text-metrics seam or an equivalent refactor away from static
Graphics measurement.

The host implements `IUiTextInputHost` so published caret geometry becomes Terminal cursor
metadata. Clipboard is an injected UI host service, not OSC 52. Sensitive values are
masked/redacted by UI before recording; Terminal does not infer password semantics from
cell styles.

### Projection policy

The bridge must publish a conformance table rather than imply pixel equivalence:

- text becomes cell text with weight/slant/underline/color downgraded by capabilities;
- fills quantize outward or inward according to one documented rule;
- one-cell strokes may use a theme-selected ASCII/box-drawing border;
- clips quantize to cell rectangles;
- translation and grid-aligned scale may be supported;
- rotation, arbitrary affine transforms, alpha blending, triangles, and images require
  explicit fallback or a reported unsupported result. Because `DrawImage` carries no alt
  text, Preview 1 uses a generic `[image]` placeholder and diagnostic; meaningful alternative
  text waits for an explicit UI semantic side channel.

The frame oracle—not the encoded escape bytes—is the behavioral truth.

### Decision gate after the vertical slice

Phase 5A measures layout correctness, fallback quality, accessibility, and implementation
burden for a narrow fixture: Label, Button, Edit, CheckBox, ListView, ScrollView, Menu, and
Dialog. Its required ADR either accepts the projector with quantitative limits or selects a
presentation-neutral/cell-native UI render contract. Only Phase 5B then expands to TabView,
CodeEditor, RichEdit, and the full published control inventory. Do not complete a broad
projector matrix before this decision or fork every control into an unrelated terminal
hierarchy merely to avoid it.

## Embedded TerminalView (deferred)

The later view needs distinct contracts for a byte connection, emulator session, screen
and scrollback snapshots, damage, input encoding, process exit, and remote disconnect.
ConPTY and Unix PTY types are connections, not presentation backends. A future SSH
connection follows RFC 4254 `pty-req`/`window-change` semantics and never shares the
attached-TUI endpoint lifecycle.

The emulator must always drain child output independently of UI rendering to prevent a
slow UI from deadlocking a PTY. It coalesces damage and bounds scrollback instead of
silently growing memory. Clipboard, hyperlinks, titles, device queries, images, desktop
notifications, and shell integration are untrusted parser actions gated through host
ports. Accessibility exposes bounded logical text ranges with stable line IDs, not one
automation object per cell.

## Alternatives considered

### Make Terminal only a Broiler.Graphics backend

Rejected as the component boundary. It would make terminal capabilities, input modes,
paste, cursor, links, session restoration, and future PTY connections look like pixel
rendering details. A Graphics-to-cell adapter is still useful, but it is an integration
package over a cell-native Terminal core.

### Clone Graphics types with terminal names

Rejected. The command-list/oracle/backend architecture transfers; bitmap resources,
DPI, alpha, font selection, transforms, and device semantics do not.

### Ship an SSH implementation in v1

Rejected. It is unnecessary for a TUI launched under SSH and would bring host-key,
credential, crypto, forwarding, network-failure, and dependency policy into the critical
path. An injected adapter remains possible later.

### Use classic Windows output as the primary path

Rejected. VT output is the cross-platform protocol and Microsoft's recommended direction.
Classic APIs may be used for input, detection, size, and an explicitly documented legacy
fallback, not as the canonical renderer.

### Claim Unix from a Linux implementation

Rejected. `termios`, PTYs, library names, signals, `ioctl` values, EOF behavior, and
packaging differ. Shared concepts do not establish a tested support claim.

## Risks and mitigations

| Risk | Design response |
| --- | --- |
| Terminal width differs by host/font/version | Pinned profile, explicit ambiguous width, virtual oracle, host matrix, full repaint on profile change |
| Capability hints overstate reality | Conservative typed resolver, explicit override, optional bounded probes, diagnostics |
| Escape/control injection | Text/control separation, sanitization, limits, side effects denied by default |
| Partial write desynchronizes the model | Commit only after complete write; mark physical state unknown and full-repaint after recovery |
| Terminal modes remain changed | Scoped lease, reverse-order async cleanup, stress tests, honest crash limitation |
| UI's pixel assumptions make projection poor | Cell theme/metrics seam, control matrix, explicit decision gate for a native UI render contract |
| Synchronous UI presentation loses terminal backpressure/faults | Awaited UI render/present seam; no dispatcher blocking or fire-and-forget writes |
| Input adapter loses paste/focus/native provenance | Typed proxy devices, marked paste event, host-focus path, shared session sequence/clock |
| Input semantics diverge across platforms | Portable baseline plus optional fidelity, separate text/key events, Broiler.Input adapter |
| SSH scope expands without need | Treat launched-under-SSH as a normal PTY; defer initiating SSH |
| Embedded emulator creates an unbounded parser | Separate later contract, strict limits/policies, fuzzing, always-draining transport |
| Accessibility claim exceeds terminal reality | Named screen-reader evidence; no semantic-transport claim for arbitrary outer terminals |

## Open decisions for Phase 0

1. Confirm the package/type names (`Terminal*` is recommended; no `B` prefix is needed
   merely to resemble Graphics).
2. Decide whether `Broiler.Terminal.Testing` is a public package or internal test support.
3. Freeze exact mandatory Windows/Linux/frontend/SSH matrix rows, pairwise coverage, and
   performance/backpressure budgets.
4. Confirm the internal friend-assembly SPI/build enforcement for Windows, Linux, and test
   drivers without exposing a public raw endpoint.
5. Confirm the exact public names for the Input coordinator/proxy devices, marked paste
   event, async UI present seam, Graphics metrics lease, UI.Standard theme lease, and
   optional Standard theme package.
6. Record the attached-presenter threat model and closed diagnostics schema.

None of these questions changes the core dependency direction or the separation between
the TUI presenter and embedded emulator.

The following are no longer open in draft 0.2: the normative
[`broiler-attached-preview1-v1`](profiles/preview-1.md) profile pins Unicode/VT behavior;
Preview 1 ships `Alternate` plus `Plain`, defers `Inline`; attached mode consumes but does
not advertise `TERM`; and an interactive Windows console that cannot enable required VT
fails or takes caller-approved Plain fallback rather than invoking a classic-output
renderer.

## Standards and primary references

The draft uses the following primary specifications and platform documentation. A
published implementation should record exact document and Unicode-data versions in its
conformance manifest.

- [ECMA-48 — Control Functions for Coded Character Sets](https://ecma-international.org/publications-and-standards/standards/ecma-48/)
- [Microsoft: Windows console and terminal definitions](https://learn.microsoft.com/en-us/windows/console/definitions)
- [Microsoft: Classic Console APIs versus virtual terminal sequences](https://learn.microsoft.com/en-us/windows/console/classic-vs-vt)
- [Microsoft: Console virtual terminal sequences](https://learn.microsoft.com/en-us/windows/console/console-virtual-terminal-sequences)
- [Microsoft: Pseudoconsoles](https://learn.microsoft.com/en-us/windows/console/pseudoconsoles)
- [The Open Group Base Specifications Issue 8: General Terminal Interface](https://pubs.opengroup.org/onlinepubs/9799919799/basedefs/V1_chap11.html)
- [RFC 4254 — SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)
- [xterm control sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)
- [terminfo database format](https://invisible-island.net/ncurses/man/terminfo.5.html)
- [Unicode Standard Annex #29 — Text Segmentation](https://www.unicode.org/reports/tr29/)
- [Unicode Standard Annex #11 — East Asian Width](https://www.unicode.org/reports/tr11/)
- [Unicode Technical Standard #51 — Unicode Emoji](https://www.unicode.org/reports/tr51/)

ECMA-48 is necessary but not sufficient for real terminals. DEC and xterm extensions
must appear in a versioned conformance table rather than being implied by the phrase
“ANSI compatible.”
