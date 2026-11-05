# Broiler.Terminal draft specification

**Status:** Draft for implementation review  
**Version:** 0.2-draft  
**Date:** 2026-09-03

## 1. Scope and conformance language

This document specifies the first Broiler.Terminal product: a platform-neutral cell
renderer, terminal presenter/input decoder, and attached-terminal backends for Windows
and Linux. It also fixes boundaries that a future embedded terminal emulator must obey.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** identify
normative requirements. Proposed C# names illustrate the intended contract shape; names
remain provisional until the Phase 0 API review. Requirement identifiers are stable
within this draft and should be referenced by tests and implementation pull requests.

Conformance has four named levels. They are separately evidenced, but they are not all
independent: an Attached-backend claim includes Core; a TUI-host claim includes Core and
at least one named Attached backend. Emulator-connection work is a separate future
profile and does not inherit from or imply TUI-host conformance.

| Level | Meaning |
| --- | --- |
| Core | Implements the neutral model, immutable render-list snapshot, frame oracle, physical planner, VT codecs, limits, and virtual frontend oracle. |
| Attached backend | Opens, sizes, reads, writes, diagnoses, and restores one attached interactive endpoint on a named OS. |
| TUI host | Presents frames and maps terminal input for an application or Broiler.UI. |
| Emulator connection | Hosts a child/remote terminal byte stream; deferred and never implied by attached-backend conformance. |

Passing on one terminal program or OS does not establish a family-wide support claim.

Requirements apply by prefix as follows unless a requirement explicitly says that it is
future-only:

| Prefixes | Conformance owner |
| --- | --- |
| `ARC`, `GEO`, `LIM`, `UNI`, `CELL`, `STY`, `REN`, `CAP`, `LIFE`, `PRE`, `IN`, `ERR`, Core portions of `SEC`/`TST`/`PKG` | Core over a scripted internal driver |
| `WIN` or `LNX`, plus inherited Core and applicable backend `SEC`/`TST`/`PKG` | Named Attached backend |
| `UI` plus applicable `SEC`/`TST`/`PKG` | TUI host |
| Requirements explicitly marked future-only and section 22 | Emulator connection; not v0.2 Preview 1 |

“Preview 1” in this document means the Phase 5 single-host attached-TUI milestone in the
roadmap. It ships `Alternate` and `Plain` modes; `Inline`, heterogeneous simultaneous UI
hosts, initiated SSH, and embedded emulation are later milestones. Preview 1 APIs remain
prerelease and may change. The first stable release is the Phase 8 milestone.

## 2. Goals and non-goals

### 2.1 Required goals

- **G-001:** Provide deterministic cell rendering independent of native backend code.
- **G-002:** Present an interactive full-screen or inline application in supported
  Windows and Linux terminal environments.
- **G-003:** Work when the application is launched through SSH with a remote PTY, using
  the backend for the remote operating system.
- **G-004:** Provide explicit behavior for redirected/no-PTY execution without emitting
  control sequences by accident.
- **G-005:** Provide the base contracts needed by a `Broiler.UI.Terminal` host.
- **G-006:** Bound untrusted input, separate text from control operations, and restore
  terminal state on normal/cancelled/faulted shutdown as far as the platform permits.
- **G-007:** Match Broiler.Graphics' neutral-core/backend/oracle/package discipline.

### 2.2 Preview 1 non-goals

- Pixel-identical rendering with Broiler.Graphics.
- Arbitrary images, sixel, kitty graphics, iTerm images, shaders, audio, or video.
- Implementing SSH cryptography, authentication, agent access, or host-key policy.
- Hosting a child shell through ConPTY or a Unix PTY.
- A complete application-output/xterm emulator and scrollback model.
- Sending rich Broiler.UI accessibility semantics through arbitrary terminal protocols.
- Assuming every Unix ABI is Linux-compatible.
- Exposing an API that writes arbitrary escape bytes into an active session.

These may be later products, but no Preview 1 type should conflate them with an attached TUI
endpoint.

## 3. Terminology

- **Frontend:** The program that displays cells and gathers user input, such as Windows
  Terminal, Console Host, a Linux virtual console, xterm, or the local side of SSH.
- **Terminal client:** The Broiler application using this component to present a TUI.
- **Attached endpoint:** The inherited console/TTY handles of the current process.
- **Connection:** A later child/remote process channel such as ConPTY, a Unix PTY, or an
  SSH session channel.
- **Ideal frame:** The immutable, capability-independent logical cell state produced by
  the grid renderer under one explicit Unicode width profile.
- **Physical model:** The presenter's best known model of what the frontend contains.
- **Committed frame:** A frame whose complete encoded update was successfully written.
- **Profile:** Versioned terminal capabilities, Unicode width rules, protocol choices,
  and quirks used for one session.
- **Plain output:** Append-only text with no terminal control sequences.

Windows Terminal and Console Host are frontends. ConPTY is a connection. A Unix terminal
emulator is a frontend. A Unix PTY is a connection/device. SSH may provide the attached
remote PTY environment or, in a later initiating-client role, a connection.

## 4. Architecture requirements

- **ARC-001:** `Broiler.Terminal` MUST target `net10.0`, contain no P/Invoke or native
  handle types, and be safe managed code.
- **ARC-002:** The core MUST be nullable-enabled, warning-clean, trimming-compatible,
  AOT-compatible, deterministic, and independently buildable.
- **ARC-003:** `Broiler.Terminal.Windows` and `Broiler.Terminal.Linux` MUST depend on the
  core. The core MUST NOT depend on either backend.
- **ARC-004:** The core MUST NOT reference Broiler.Graphics, Broiler.Input, Broiler.UI,
  an SSH implementation, or a platform UI framework.
- **ARC-005:** Native handles, file descriptors, console records, `termios` layouts, and
  signal constants MUST remain internal to the applicable backend.
- **ARC-006:** Applications MUST construct a backend explicitly. There MUST NOT be a
  reflection/plugin scan or an all-platform meta-package.
- **ARC-007:** Public cross-platform state MUST use immutable records/value types or
  explicitly owned immutable snapshots. Mutable command builders are permitted only as
  single-writer construction objects; they MUST freeze to an immutable snapshot before
  validation, replay, cross-thread transfer, or presentation.
- **ARC-008:** Backend diagnostics MUST reveal the selected endpoint kind, profile source,
  enabled modes, downgrades, and failure category without exposing a native handle or
  writing unsanitized environment/input data.

Planned initial packages are `Broiler.Terminal`, `Broiler.Terminal.Windows`, and
`Broiler.Terminal.Linux`. `Broiler.Terminal.Testing` MAY become a package only if multiple
repositories need its fixtures/oracle; otherwise it remains test infrastructure.

## 5. Neutral geometry and limits

### 5.1 Geometry

The public model SHOULD have the following shape:

```csharp
public readonly record struct TerminalSize
{
    public TerminalSize(int columns, int rows)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(columns);
        ArgumentOutOfRangeException.ThrowIfNegative(rows);
        Columns = columns;
        Rows = rows;
    }

    public int Columns { get; }
    public int Rows { get; }
}

public readonly record struct TerminalPoint(int Column, int Row);

public readonly record struct TerminalRect
{
    public TerminalRect(int column, int row, int width, int height)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(width);
        ArgumentOutOfRangeException.ThrowIfNegative(height);
        _ = checked(column + width);
        _ = checked(row + height);
        Column = column;
        Row = row;
        Width = width;
        Height = height;
    }

    public int Column { get; }
    public int Row { get; }
    public int Width { get; }
    public int Height { get; }
}
```

- **GEO-001:** Origins are zero-based; column increases rightward and row downward.
- **GEO-002:** Negative sizes and integer-overflowing bounds MUST be rejected.
- **GEO-003:** An empty rectangle is valid. An interactive surface size MUST be positive.
- **GEO-004:** Resize MUST be expressed as a new complete `TerminalSize`; partial width or
  height mutation is not observable.
- **GEO-005:** Pixel dimensions MAY accompany size diagnostics/SSH negotiation but MUST
  NOT affect cell layout unless a future explicitly named pixel protocol is selected.

### 5.2 Resource limits

The core MUST accept one immutable `TerminalLimits` wherever untrusted or caller-sized
data enters: render-list builders/freezing, grid rendering, frame import, codecs, session
creation, and future parsers. Preview 1 defaults are:

| Limit | Proposed default |
| --- | ---: |
| Columns | 1,000 |
| Rows | 1,000 |
| Total visible cells | 1,000,000 |
| Unicode scalar values in one stored grapheme | 64 |
| Render commands in one frozen list | 100,000 |
| Total Unicode scalar values in one frozen list | 4,000,000 |
| Distinct graphemes in one frame | 65,536 |
| Distinct styles in one frame | 16,384 |
| Distinct links in one frame | 4,096 |
| Total hyperlink-table UTF-8 bytes in one frame | 1 MiB |
| Optional frame-metadata entries | 1,024 |
| One encoded presentation update | 64 MiB |
| Incomplete/unknown frontend input sequence | 4 KiB |
| One complete bracketed-paste transaction | 1 MiB |
| Title | 4,096 Unicode scalar values |
| Hyperlink URI | 8 KiB UTF-8 |
| Pending non-coalescible input events | 4,096 |
| Pending presentations | 2 |
| Diagnostic events per rolling minute | 10,000 |

- **LIM-001:** Limits MUST be checked before allocation or multiplication.
- **LIM-002:** A limit violation MUST produce a typed fault/diagnostic and deterministic
  recovery; it MUST NOT silently wrap, allocate without bound, or execute content.
- **LIM-003:** Pointer-move and resize events MAY be coalesced. Key, text, focus, and paste
  events MUST NOT be silently dropped. Queue exhaustion faults the input stream.
- **LIM-004:** Limits in a future emulator parser MUST additionally cover scrollback rows
  and bytes, OSC/DCS/APC payloads, links, replies, and action rate.
- **LIM-005:** Aggregate limits MUST be checked independently of per-item limits. Interning
  MUST NOT make the accepted input size unbounded, and encoded-size limits MUST be checked
  while planning rather than after allocating the complete output.
- **LIM-006:** A slow diagnostic sink MUST NOT block the session actor or bypass diagnostic
  rate/queue limits. Overflow emits one coalesced, value-free notice when possible.

## 6. Unicode and cell model

### 6.1 Width profile

The public model SHOULD expose a versioned profile similar to:

```csharp
public sealed record TerminalWidthProfile(
    string Id,
    Version UnicodeVersion,
    TerminalAmbiguousWidth AmbiguousWidth,
    TerminalEmojiWidth EmojiWidth,
    TerminalInvalidTextPolicy InvalidTextPolicy);
```

- **UNI-001:** Text MUST be decoded as valid Unicode scalar values and segmented into
  extended grapheme clusters using a pinned Unicode version.
- **UNI-002:** The first implementation SHOULD pin Unicode 17.0.0 unless Phase 0 records
  a different fully released version.
- **UNI-003:** Column width MUST be calculated by a versioned terminal-width profile. It
  MUST NOT be inferred from UTF-16 length, font metrics, or East Asian Width alone.
- **UNI-004:** Ambiguous-width characters MUST have an explicit one- or two-column policy.
- **UNI-005:** Emoji presentation sequences and ZWJ clusters MUST follow an explicit
  profile. Unknown/pathological clusters MUST use a documented fallback width/glyph.
- **UNI-006:** Invalid UTF-8 MUST be rejected or replaced deterministically according to
  policy; it MUST never enter parser control state as decoded text.
- **UNI-007:** Profile identity MUST be included in snapshots and diagnostics. Changing a
  profile invalidates all frames and requires a full repaint.
- **UNI-008:** `TerminalInvalidTextPolicy` applies equally to malformed UTF-8 and to
  ill-formed UTF-16 supplied through public .NET `string` APIs. Unpaired surrogates are
  rejected with a scalar offset or replaced with U+FFFD before segmentation.
- **UNI-009:** The core preserves the caller's scalar sequence. It performs neither Unicode
  normalization nor bidirectional reordering. Snapshot cell order is logical left-to-right
  command order; frontend shaping and bidi behavior are compatibility observations outside
  oracle equality and MUST be recorded as named frontend deviations.

### 6.2 Cell invariants

A public cell may expose a grapheme and style directly while the implementation interns
them. It MUST distinguish at least empty/one-column/two-column-leading/continuation state.

- **CELL-001:** A printable grapheme occupies exactly one leading cell and zero or one
  continuation cell in Preview 1.
- **CELL-002:** Every two-column leading cell MUST be followed on the same row by exactly
  one continuation cell. Every continuation cell MUST have a preceding leading cell.
- **CELL-003:** Writing over either half of a two-column grapheme clears both old cells
  before writing the replacement.
- **CELL-004:** Clipping or resizing MUST NOT leave an orphan leading/continuation cell.
- **CELL-005:** Combining-only input at a position with no base MUST use a documented
  replacement/dotted-circle policy; it MUST NOT create a negative-width cell.
- **CELL-006:** C0/C1 controls, ESC, newline, and tab are layout/control input, never stored
  verbatim as a printable cell grapheme.
- **CELL-007:** Soft-wrap metadata is not required for the attached TUI frame, but any
  future emulator screen/history model MUST preserve it separately from hard line breaks.
- **CELL-008:** An empty cell is distinct in the ideal frame from a one-column U+0020 cell.
  An empty cell carries its clear style but no grapheme or link. A styled or linked space is
  never equivalent to empty for diffing. Plain output treats both as blank for trailing
  trimming because plain output discards style and links.
- **CELL-009:** Preview 1 text commands accept LF as a hard line break, ignore a preceding
  CR only as part of CRLF, and reject a standalone CR and all other C0/C1 controls. TAB
  advances to the next session-configured tab stop (default every eight columns) by writing
  unlinked styled spaces. Text never causes terminal protocol control bytes to be copied.
- **CELL-010:** `FillRect` accepts exactly one valid one-column grapheme or an explicit
  empty-fill value. It rejects controls, zero-column combining-only input, two-column
  graphemes, and strings containing more than one extended grapheme cluster. Wide text is
  written only through `DrawText`.

## 7. Color, style, links, and cursor

The ideal frame MUST represent default color, indexed color, and RGB color without first
quantizing to the attached frontend. At minimum it supports bold, faint, italic, single
underline, inverse, hidden, and strike attributes as independent flags.

- **STY-001:** Foreground/background `Default` MUST remain distinct from an assumed RGB
  value so user themes continue to work.
- **STY-002:** Capability downgrade MUST be deterministic. The reference conversion from
  RGB to indexed color MUST use a documented nominal palette/distance function with
  deterministic tie-breaking.
- **STY-003:** Unsupported style flags are removed; they MUST NOT be replaced with a
  semantically different decoration without an explicit theme policy. `Hidden` is the
  exception: an unsupported hidden/conceal attribute renders blank cells rather than
  revealing its grapheme. Hidden remains a visual effect, not a secure-redaction feature.
- **STY-004:** Hyperlink identity MUST be separate from cell text and URI data MUST pass a
  caller-provided scheme/length policy. Link identity and URI MUST reject ESC, BEL, C0/C1,
  Unicode line separators, invalid Unicode, and every OSC/ST terminator representation
  before encoding. Unsupported/denied links render as plain text.
- **STY-005:** The cursor is frame metadata containing position, visibility, and requested
  shape. Unsupported shapes downgrade while retaining position/visibility.
- **STY-006:** A title is session metadata through a typed API. Text drawing MUST NOT set
  titles. Title changes are disabled in plain mode and policy-check length/control data.
- **STY-007:** Clipboard protocols, including OSC 52, are disabled by default and are not
  part of first-preview conformance.

## 8. Render list, frame, and reference renderer

### 8.1 Proposed contract shape

```csharp
public sealed class TerminalRenderListBuilder
{
    public void Clear(TerminalStyle style);
    public void FillRect(TerminalRect rect, string grapheme, TerminalStyle style);
    public void DrawText(TerminalTextRun text, TerminalPoint origin);
    public void PushClip(TerminalRect rect);
    public void PopClip();
    public void SetCursor(TerminalCursor cursor);
    public TerminalRenderList Freeze(TerminalLimits limits);
}

public sealed class TerminalRenderList
{
    public IReadOnlyList<TerminalRenderCommand> Commands { get; }
    public void Validate(TerminalLimits limits);
}

public sealed record TerminalSurfaceDescriptor(
    TerminalSize Size,
    TerminalWidthProfile WidthProfile,
    TerminalStyle ClearStyle,
    long SessionGeneration);

public readonly record struct TerminalFrameContext(
    long FrameIndex,
    TerminalOverflowPolicy OverflowPolicy);

public sealed class TerminalGridRenderer
{
    public TerminalFrame Render(
        TerminalSurfaceDescriptor surface,
        TerminalRenderList renderList,
        TerminalFrameContext context);
}
```

- **REN-001:** The command hierarchy MUST be closed or otherwise exhaustively validated.
- **REN-002:** A render list is pure managed ordered data and MUST NOT retain a backend,
  stream, native handle, or live session.
- **REN-003:** Clip stacks MUST be balanced and underflow-free. Validation occurs before
  any endpoint write.
- **REN-004:** Replay MUST be deterministic for the same descriptor, list, context, and
  Unicode data version.
- **REN-005:** The reference grid renderer is the semantic oracle. Backends MUST NOT
  implement different clipping, width, overflow, or overwrite behavior.
- **REN-006:** `TerminalFrame` MUST be an immutable snapshot with size, profile identity,
  cells, cursor, frame index, and optional bounded metadata.
- **REN-007:** Frame memory ownership MUST be explicit. Disposing/reusing a builder or
  source list MUST NOT mutate a published frame.
- **REN-008:** Drawing text MUST accept explicit clip/wrap/truncate policy. A wide cluster
  that cannot fit MUST follow that policy without writing half a cluster.
- **REN-009:** Text APIs MUST sanitize or reject terminal controls before they reach the
  VT encoder. There is no public raw-sequence command.
- **REN-010:** A builder is single-threaded and mutable only until `Freeze`. Freezing
  validates limits, copies or transfers ownership atomically, and permanently closes the
  builder. Replay accepts only the frozen `TerminalRenderList`; later source-string or
  builder changes cannot affect it.
- **REN-011:** `TerminalTextRun` contains style/link plus one explicit horizontal overflow
  choice: `Clip`, `Wrap`, or `TruncateWithMarker`. `TerminalFrameContext.OverflowPolicy`
  governs vertical overflow after LF/wrap and never silently overrides the run's horizontal
  choice. A wide cluster that does not fit is handled atomically by that choice.

### 8.2 Commands intentionally absent

Images, arbitrary paths, triangles, alpha blending, affine transforms, sub-cell
coordinates, and font selection are not core Terminal commands. Integration adapters may
project them to supported cell commands or return a structured unsupported/fallback
diagnostic.

## 9. Capabilities and profiles

`TerminalCapabilities` MUST be typed and immutable. It includes quantitative color level
and explicit support for at least cursor addressing, erase operations, alternate screen,
cursor visibility/shape, mouse protocol variants, focus reporting, bracketed paste,
synchronized output, hyperlinks, title change, and native Windows key fidelity. Enhanced
VT keyboard protocols are future profile additions, not Preview 1 capabilities.

- **CAP-001:** Capability resolution MUST collect backend facts first, merge conservative
  terminfo/environment hints second, and merge optional bounded probe results third. It
  then applies typed per-capability overrides (`Auto`, `ForceOff`, or
  `AssumeOnForTestingOrForcedVt`) and finally intersects the result with application
  feature permission. A forced assumption is never described as detected support.
- **CAP-002:** `TERM`, `COLORTERM`, `WT_SESSION`, SSH variables, and similar strings are
  untrusted hints. No single environment variable enables unsafe side effects.
- **CAP-003:** `TERM=dumb` and a noninteractive endpoint MUST select a conservative profile.
  A caller MAY request forced VT only through the visibly named unsafe-assumption option;
  the session records every assumed capability and never calls the result detected support.
- **CAP-004:** Active queries are disabled by default. When enabled, they MUST be bounded
  by byte count, outstanding-query count, exact response grammar, and timeout.
- **CAP-005:** Query replies MUST be accepted only for a matching outstanding query. Other
  bytes remain user input or a bounded unknown event.
- **CAP-006:** The effective capability set is the intersection of resolved/assumed support
  and application feature policy. `ForceOff` and denied policy always win over `AssumeOn`.
- **CAP-007:** The session MUST expose resolution source and every downgrade in sanitized
  diagnostics.
- **CAP-008:** A backend MUST NOT advertise a TERM profile to a child/remote peer until
  its corresponding conformance table passes. This applies to future emulator connections.
- **CAP-009:** Erase support distinguishes ordinary erase from back-color erase (`bce`).
  When erasing cannot reproduce the target blank-cell style exactly, the planner writes
  explicit styled spaces instead of using `EL`, `ED`, or `ECH`.

### 9.1 Preview 1 protocol baseline

The checked-in [Preview 1 profile manifest](profiles/preview-1.md) is normative test data
and carries its own revision/hash.
Preview 1 uses UTF-8 graphic text and 7-bit ESC-prefixed controls only; it never emits
single-byte C1 controls. The baseline vocabulary is closed:

| Function | Preview 1 sequence/profile rule |
| --- | --- |
| Absolute cursor | `CSI row ; column H` with one-based parameters |
| Rendition | `CSI ... m`; reset `CSI 0 m`; 16/256/RGB forms only when enabled |
| Erase | `EL`/`ED`/`ECH` only when the selected `bce` and blank-style rules are exact |
| Cursor visibility/shape | DEC `?25h`/`?25l`; DECSCUSR only when enabled |
| Alternate screen | DEC private mode 1049; mandatory for Preview 1 interactive mode |
| Bracketed paste | DEC private mode 2004 |
| Focus reporting | DEC private mode 1004 |
| Mouse | SGR 1006 with explicitly selected tracking mode 1000 or 1002 |
| Synchronized output | DEC private mode 2026, optional |
| Hyperlinks | OSC 8 with ST terminator, optional and policy-gated |
| Title | OSC 2 with ST terminator, optional and policy-gated |

The manifest additionally fixes SGR reset minimization, nominal 16/256 palettes, RGB
distance/tie-breaking, autowrap and bottom-right strategy, tab stops, response grammars,
and named frontend quirks. No implementation work depending on those decisions may pass
Phase 0 until the manifest and golden fixtures are committed. Attached mode consumes
`TERM`; it does not advertise a `TERM` value. Advertising belongs to future connections.

## 10. Endpoint and session lifecycle

### 10.1 Proposed contract shape

```csharp
public interface ITerminalBackend
{
    ValueTask<TerminalSession> OpenAttachedAsync(
        TerminalOpenOptions options,
        CancellationToken cancellationToken = default);
}

public sealed record TerminalSessionSnapshot(
    TerminalSessionState State,
    TerminalEndpointInfo Endpoint,
    TerminalSize Size,
    TerminalCapabilities Capabilities,
    string ProfileId,
    long Generation);

public abstract record TerminalSessionEvent(
    long SequenceNumber,
    TerminalTimestamp Timestamp)
{
    public sealed record Input(long SequenceNumber, TerminalTimestamp Timestamp, TerminalInputEvent Value)
        : TerminalSessionEvent(SequenceNumber, Timestamp);
    public sealed record SnapshotChanged(long SequenceNumber, TerminalTimestamp Timestamp, TerminalSessionSnapshot Value)
        : TerminalSessionEvent(SequenceNumber, Timestamp);
    public sealed record Disconnected(long SequenceNumber, TerminalTimestamp Timestamp, TerminalDisconnectReason Reason)
        : TerminalSessionEvent(SequenceNumber, Timestamp);
    public sealed record Faulted(long SequenceNumber, TerminalTimestamp Timestamp, TerminalFault Fault)
        : TerminalSessionEvent(SequenceNumber, Timestamp);
}

public sealed class TerminalSession : IAsyncDisposable
{
    public TerminalSessionSnapshot Snapshot { get; }

    public IAsyncEnumerable<TerminalSessionEvent> ReadEventsAsync(
        CancellationToken cancellationToken = default);

    public ValueTask<TerminalPresentResult> PresentAsync(
        TerminalFrame frame,
        CancellationToken cancellationToken = default);

    public ValueTask<TerminalMetadataResult> SetTitleAsync(
        string title,
        CancellationToken cancellationToken = default);
}
```

The application-facing API contains no raw endpoint. The core exposes an internal friend
assembly SPI to the Windows, Linux, and test assemblies. Its one ordered ingress union may
carry arbitrary byte chunks, native key/pointer records expressed as neutral internal
fields, resize, capability facts, EOF, and faults. Its write operation is write-all: success
means every byte was accepted; any exception or cancellation after the operation begins
means an unknown prefix may have been accepted. `Read == 0` means EOF only for a non-empty
buffer. The driver rejects concurrent reads and writes unless its contract explicitly
supports one read pump and one write pump.

Native handles, file descriptors, and console record structs never cross the friend SPI.
The Windows backend translates them into the neutral ingress union without first reducing
them to VT bytes. Linux/stream backends submit arbitrary UTF-8/escape byte chunks; chunk
boundaries have no semantic meaning.

### 10.2 Ownership

- **LIFE-001:** `OpenAttachedAsync` returns a session that already exclusively owns its
  internal driver. No public raw endpoint or transferable alias exists.
- **LIFE-002:** Disposing the session always disposes/restores its internal driver. Ownership of
  underlying caller-supplied streams is selected explicitly when a stream backend is
  constructed; it is not inferred.
- **LIFE-003:** Two active sessions MUST NOT share one terminal/console driver.
- **LIFE-004:** Disposal is idempotent. Operations after disposal throw
  `ObjectDisposedException` or return a documented terminal result.
- **LIFE-005:** No finalizer writes bytes, changes native modes, waits for user input, or
  closes caller-owned streams.
- **LIFE-005a:** Supplying a stream transfers exclusive operational use for the session
  lifetime even when `leaveOpen` preserves close ownership. Writes through a retained stream,
  `Console.Out`, or another alias are unsupported external interference; the application
  MUST route logging through the injected sink. A host that knowingly permits external
  writes MUST invalidate the generation/physical model before the next presentation.

### 10.3 State machine

The lifecycle states are `Opening`, `Active`, `Suspended`, `Closing`, `Closed`, and
`Faulted`. `Created` is an internal factory state. `Opening` is observable through
structured diagnostics/open faults; a successfully returned session starts at `Active`.
All later states are observable through `Snapshot` and the ordered event/completion path.

| From | Trigger | To | Public operation rule |
| --- | --- | --- | --- |
| Opening | all acquisition/activation steps succeed | Active | factory completes |
| Opening | cancellation/failure | Closed or Faulted after rollback | factory throws a typed open fault containing rollback faults |
| Active | OS job-control suspension or later explicit suspend | Suspended | new presents are rejected; pumps retain only bounded state |
| Suspended | resume/re-entry succeeds | Active with incremented generation | next present MUST be a fresh full frame |
| Active/Suspended | dispose | Closing then Closed | new operations fail; bounded cleanup continues despite caller cancellation |
| Active/Suspended | unrecoverable I/O/invariant fault | Faulted | event is published; only snapshot/diagnostics/disposal remain valid |
| Faulted | dispose | Closing then Closed | cleanup faults are aggregated with the primary fault |

`Snapshot` is an atomically replaced immutable value. Size, width profile, effective
capabilities, mode, or resume changes increment `Generation` and publish one ordered
`SnapshotChanged` event before any later input event is delivered.

- **LIFE-006:** Opening captures original native state before its first mutation.
- **LIFE-007:** Mode activation and cleanup order MUST be documented and tested. Cleanup
  disables only reporting the session enabled, resets rendition/shows the cursor only if
  interactive output was activated, leaves the selected screen mode, then restores native
  modes/code pages/signals in reverse acquisition order. Plain cleanup writes no terminal
  sequences.
- **LIFE-008:** Cancellation during opening rolls back completed steps. Cancellation of a
  public operation does not skip the session's bounded cleanup path. `TerminalOpenOptions`
  carries a cleanup deadline (Preview 1 default: two seconds per blocking driver step and
  ten seconds overall). Timeout advances to the next independent restoration step and is
  retained as a restoration fault.
- **LIFE-009:** Abrupt process termination is documented as non-restorable; the API MUST
  NOT claim crash-proof restoration.
- **LIFE-010:** Suspend/resume or capability change invalidates the physical model and
  requires a complete repaint before incremental updates resume.
- **LIFE-010a:** A frame carries the generation used to render it. `PresentAsync` rejects a
  frame whose generation, size, or width-profile identity differs from the current snapshot
  with `StaleFrame`; it never crops or silently reinterprets the frame. Duplicate/decreasing
  frame indexes are rejected within one generation.
- **LIFE-010b:** Exactly one consumer MAY enumerate `ReadEventsAsync`. A second concurrent
  enumeration fails immediately. Events have monotonic sequence numbers; coalescing MAY
  replace only adjacent pointer-move or resize records and MUST publish the last such record
  before a following non-coalescible event.

### 10.4 Screen modes

The public contract distinguishes:

- `Alternate`: owns the terminal viewport and requests alternate-screen behavior;
- `Inline`: owns a bounded region without claiming the whole viewport; and
- `Plain`: append-only, no cursor addressing or reporting modes.

Preview 1 implements `Alternate` and `Plain`. `Inline` is a reserved enum value whose open
attempt returns `UnsupportedMode` until Phase 6 defines and tests region reservation,
scrolling, resize/reflow, external-writer interference, and final cursor placement. This
avoids treating an underspecified shared viewport as an implementation option.

- **LIFE-011:** The selected mode is explicit. Failure to provide a required interactive
  feature returns a structured failure/downgrade; it MUST NOT silently clear the user's
  main screen or emit ANSI to a redirected file.
- **LIFE-012:** A plain renderer writes rows as text with trailing blank cells removed and
  LF after every row, including the last. It emits a complete snapshot on every successful
  `PresentAsync`, including an unchanged frame; snapshots are separated by no additional
  marker. It never reads interactive input, changes terminal/native modes, or emits ESC,
  C1, BEL, BS, TAB, CR, or any C0 byte other than LF. Repeated snapshots append; they do
  not pretend to update an earlier file position.
- **LIFE-013:** `Alternate` enters DEC private mode 1049 before enabling optional input
  reporting and leaves it after reporting is disabled during cleanup. Failure to enter
  required alternate-screen/cursor-addressing capability fails interactive open; automatic
  fallback to Plain occurs only when `TerminalOpenOptions` explicitly permits it.
- **LIFE-014:** Interactive eligibility is evaluated independently for input and output.
  Preview 1 requires both sides to refer to the same attached terminal/console. TTY input
  plus redirected output, redirected input plus TTY output, different TTYs, a missing
  handle, or an implicitly discovered `/dev/tty` is noninteractive unless the caller
  explicitly supplies and owns a matching stream backend. Redirected output defaults to
  Plain; redirected input is ignored in Plain.

## 11. Presentation and physical-state recovery

- **PRE-001:** Presentation compares an ideal target frame to the last committed physical
  model and plans a deterministic capability-aware update.
- **PRE-002:** In interactive mode, an unchanged frame emits zero display bytes unless
  cursor, title, mode, or another explicit metadata item changed. Plain-mode snapshot
  emission follows `LIFE-012` and is intentionally not deduplicated.
- **PRE-003:** The planner MAY choose a full row/screen repaint when it is safer or cheaper
  than a sparse diff. The cost/tie-breaking rules MUST be deterministic for a profile.
- **PRE-004:** Output writes are serialized. `PresentAsync` provides backpressure; Preview 1
  MUST NOT secretly discard or reorder frames.
- **PRE-005:** A frame becomes committed only after the complete encoded update succeeds.
  A cancellation, short/partial write, broken pipe, or unknown exception marks the
  physical model unknown.
- **PRE-006:** After an unknown physical state, the next successful presentation MUST
  reset necessary modes and fully repaint before diff mode resumes.
- **PRE-007:** The encoder MUST handle the last column/bottom-right/autowrap case according
  to the selected profile without unintended scroll or half-written wide cells.
- **PRE-008:** Synchronized-output mode MAY bracket a frame only when explicitly supported.
  Its absence means visual atomicity is not promised.
- **PRE-009:** The presenter MUST leave rendition in a documented neutral state after a
  frame so unrelated cleanup/diagnostic output is not accidentally styled.
- **PRE-010:** Logging/diagnostics for an active full-screen session MUST use a supplied
  sink and MUST NOT write uncontrolled text to the presented stdout stream.
- **PRE-011:** The UI/integration layer always submits a complete ideal frame. Only
  `TerminalSession` owns the downgraded target, last committed physical model, diff plan,
  partial-write invalidation, and full-repaint recovery.
- **PRE-012:** `SetTitleAsync` is serialized with presentations. Success commits the title;
  a cancellation/failure after writing begins leaves title state unknown and causes the
  next permitted title operation to rewrite it. Titles are never part of plain snapshots.

The diff planner is `O(columns * rows)` time in the worst case and stores no more than a
constant number of visible frames plus bounded scratch data. Benchmarks MUST separately
record unchanged, sparse, full, resize, wide-Unicode, and high-style-entropy frames.

The driver read pump and write pump may block independently. They publish completion/data
to one nonblocking session actor that alone mutates decoder, mode, snapshot, and physical
state. User code is invoked only by asynchronous enumeration after actor state is released.
The actor never awaits a driver read, write, or diagnostic sink. Presentation requests are
ordered by actor acceptance and bounded by `TerminalLimits.PendingPresentations`; Preview 1
does not coalesce them. Disposal stops admission, cancels pumps, and runs the independent
cleanup sequence even if an earlier step times out.

## 12. Frontend input

### 12.1 Event model

Terminal input events include separate key, committed text, pointer, paste, focus, and
unknown/diagnostic variants. Resize/disconnect/capability changes are session events and
are serialized in the same ordered event stream before the next render.

Every event carries one session sequence number and a timestamp from the injected
monotonic session clock. Byte-stream events are timestamped when the read pump receives
their first contributing byte; native records retain a backend timestamp only as optional
diagnostic fidelity and use the session clock for portable ordering. Optional native key
fields use nullable values rather than invented zero virtual-key/scan codes.

- **IN-001:** Physical/named key information and committed text are separate. Consumers
  insert committed text and use key events for commands; they MUST NOT infer text twice.
- **IN-002:** A legacy VT printable byte sequence MAY produce text without a physical key
  event. Key-up, scan code, repeat, and lock state are optional fidelity fields.
- **IN-003:** On a Windows attached console, the backend SHOULD use native console input
  records when available so resize, mouse, key transition/repeat/scan information is not
  discarded merely to force VT input.
- **IN-004:** On Linux, an incremental UTF-8/escape decoder MUST preserve state across
  arbitrary read boundaries.
- **IN-005:** Escape-key ambiguity uses a configurable bounded timeout and injectable
  clock. Tests MUST cover every split point and timeout edge; no wall-clock sleeps are
  required in core tests.
- **IN-006:** Unknown or malformed sequences MUST recover deterministically and within the
  configured byte limit. They MUST NOT invoke output side effects.
- **IN-007:** Bracketed paste is one or more explicitly marked paste events. Escape-looking
  bytes inside the bracketed payload are text, subject to size/control policy, not keys;
  the first exact `CSI 201~` delimiter terminates the transaction and is not payload.
- **IN-008:** Paste does not imply trust. Multiline/control-character confirmation is a UI
  policy above this component.
- **IN-009:** Mouse coordinates are converted to zero-based cells, validated against
  limits, and emitted only for an enabled/supported reporting protocol.
- **IN-010:** Focus events are emitted only when focus reporting was enabled by this
  session; unsolicited matching bytes remain bounded unknown input.
- **IN-011:** IME/pre-edit composition is generally owned by the frontend and MAY arrive
  only as committed text. Composition fidelity MUST NOT be claimed on legacy VT input.
- **IN-012:** Signal handling is explicit. Preview 1 `Alternate` mode defaults to delivering
  Ctrl+C/Ctrl+Z as input so the host decides whether to cancel or exit and can dispose the
  session. An explicit option MAY preserve OS signal behavior and is diagnosed as reduced
  cleanup assurance. `Plain` never changes inherited signal behavior. Every changed native
  flag/handler is restored.

### 12.2 Serialization and reentrancy

- **IN-013:** Input decoding, session resize, capability change, mode changes, frame
  presentation, and protocol replies execute through one serialized session context.
- **IN-014:** User callbacks are not invoked while holding parser/model locks.
- **IN-015:** Concurrent callers MAY enqueue operations only where documented. Observable
  writes and events remain ordered.
- **IN-016:** A TUI host maps one legacy terminal keypress to the balanced input transaction
  its target UI expects (for example synthetic down/up) when the protocol has no key-up.
  Synthetic fidelity MUST be identified as such.
- **IN-017:** Fragmentation invariance means that the same timestamped byte stream produces
  the same events regardless of read chunk boundaries. Advancing the virtual clock past the
  Escape timeout is a different timed stream and MAY produce different events. Fixture data
  includes arrival times, timeout advances, cancellation, and EOF flush.
- **IN-018:** A paste transaction that exceeds its total scalar/byte limit faults that paste,
  enters bounded discard-to-terminator state, and never reinterprets residual payload as
  keys or control replies. EOF/missing terminator produces one typed paste fault. The
  decoder retains only delimiter-prefix state while discarding.
- **IN-019:** Only enabled outstanding active queries can consume a matching reply. A timed-
  out or malformed reply is ordinary bounded input; matching proves grammar and ordering,
  not authenticity, which is why probes remain disabled by default.

## 13. Windows attached backend

- **WIN-001:** The backend detects whether inherited input/output are console handles,
  VT-capable streams, or redirected/noninteractive handles before changing modes.
- **WIN-002:** Windows Terminal and Console Host use one backend and VT-first output. The
  package MUST NOT depend on the Windows Terminal settings/object model.
- **WIN-003:** The backend enables only required console flags, records original values,
  and restores them exactly on disposal where the handles remain valid.
- **WIN-004:** Output encoding/code-page changes, if any, are scoped and restored. The core
  continues to operate on Unicode/UTF-8 semantics.
- **WIN-005:** Native input SHOULD use `ReadConsoleInputW` when possible. A VT byte-input
  path MAY be used for pipe/pseudoconsole cases and MUST report reduced fidelity.
- **WIN-006:** Resize reports the visible terminal viewport in columns/rows, not an
  unrelated backing-buffer maximum.
- **WIN-007:** Redirected output defaults to plain. An explicit forced-VT policy is caller
  responsibility and appears in diagnostics.
- **WIN-008:** Blocking native reads MUST have a tested cancellation/shutdown strategy;
  disposal MUST NOT hang indefinitely.
- **WIN-009:** The release manifest names exact mandatory Windows/runtime versions,
  x64/Arm64, Console Host and Windows Terminal builds, PowerShell/cmd versions, and Windows
  OpenSSH server/client builds. It also covers PTY/no-PTY, resize storms, selection/external
  suspend, every mixed redirected-handle case, and the Phase 0 pairwise coverage policy.

ConPTY is not used merely because the app runs in Windows Terminal. Future ConPTY child
hosting uses a separate connection type and emulator flow.

## 14. Linux attached backend

- **LNX-001:** The backend verifies the attached file descriptors with `isatty` (or an
  equivalent direct OS check) before entering interactive mode.
- **LNX-002:** It snapshots and restores the exact relevant `termios` state through a
  scoped lease; it modifies inherited flags rather than assuming a zero/default struct.
- **LNX-003:** It reads size with the supported `winsize` interface and observes `SIGWINCH`
  or an equivalent safe notification path. Resize storms MAY be coalesced.
- **LNX-004:** Signal registration and teardown MUST avoid process-global handler races
  and MUST NOT call arbitrary managed callbacks from an unsafe native signal context.
- **LNX-005:** Capability discovery uses a tested terminfo/environment resolver and
  records missing/malformed database entries as diagnostics with conservative fallback.
- **LNX-006:** Interactive first-preview output requires a UTF-8-capable profile. A
  non-UTF-8 locale fails or takes an explicit documented compatibility policy; it MUST NOT
  silently reinterpret UTF-8 bytes.
- **LNX-007:** Exact Linux virtual-console/emulator, `tmux`/screen, locale, no-PTY/
  redirection, and Linux OpenSSH builds are distinct release-manifest rows; “common” or
  “current” is not a reproducible matrix value.
- **LNX-008:** glibc/musl and x64/Arm64 claims require their own build/runtime evidence.
- **LNX-009:** No macOS/BSD/“all Unix” support statement follows from these tests.

A future Unix PTY launcher MUST NOT execute managed code after `fork()` in a
multithreaded .NET process. It SHOULD use a small audited native launcher or another
platform-safe spawn primitive.

## 15. SSH behavior

### 15.1 Application launched through SSH

- **SSH-001:** With an allocated remote PTY, the application uses the attached backend for
  the remote OS. The SSH client/server is outside the component boundary.
- **SSH-002:** The effective profile derives conservatively from the remote PTY's terminal
  type/capabilities and dimensions. Network latency does not change frame semantics.
- **SSH-003:** Without a remote PTY, the endpoint is noninteractive and follows plain or
  explicit failure policy.
- **SSH-004:** Automated conformance MUST test Linux and Windows OpenSSH PTY and no-PTY
  execution, resize propagation, Unicode, disconnect, latency/jitter, and diagnostics.

### 15.2 Future initiated SSH connection

- **SSH-005:** An optional adapter maps an already authenticated/verified SSH channel to a
  separate connection contract, including RFC 4254 `pty-req`, `window-change`, exit status,
  exit signal, EOF, and disconnect.
- **SSH-006:** The component MUST NOT silently accept host keys, implement weak crypto,
  expose credentials in diagnostics, or enable agent/X11/port/environment forwarding by
  default.
- **SSH-007:** A system-`ssh`-through-local-PTY integration SHOULD precede an in-process
  dependency so the user's existing agent, known-hosts, and configuration policy remain
  authoritative.
- **SSH-008:** Disconnect is not described as resumable; persistence is provided by a
  remote application such as `tmux`/`screen`, not this transport.

## 16. Broiler.Input and Broiler.UI integration contract

### 16.1 Input adapter

- **UI-001:** `Broiler.Input.Terminal` is an integration/coordinator package over
  `Broiler.Input`, `.Keyboard`, `.Mouse`, `.Text`, and `Broiler.Terminal`. It exposes typed
  keyboard, mouse, and text proxy devices sharing one non-owning adapter; it does not create
  a new universal input-device base class. `Broiler.UI.Terminal` owns the adapter and the
  `TerminalSession`, so device start/stop/dispose cannot independently close the session.
- **UI-002:** Key/text/pointer events are adapted into Broiler.Input types before normal UI
  routing. Terminal-frontend focus is a host-activation event handled directly by
  `Broiler.UI.Terminal`, not element focus or a fabricated input device. The Terminal core
  MUST NOT define application-global focus/capture or depend on Input/UI types.
- **UI-003:** Legacy terminal keypresses without release MUST be translated into a
  documented balanced transaction for controls that activate on key-up. Native Windows
  input retains real transitions when available. Preview 1 claims
  real transitions only for native Windows console records; no enhanced VT keyboard
  protocol is claimed until a later profile enumerates its negotiation and fixtures.
- **UI-004:** Text and key paths MUST not double-insert printable characters. AltGr and
  Ctrl+Alt MUST not be treated as equivalent without platform evidence.
- **UI-004a:** The adapter emits the canonical `ArrowLeft`/`ArrowRight`/`ArrowUp`/
  `ArrowDown` names used by Broiler.Input. Broiler.UI MUST centralize legacy aliases
  instead of making the terminal adapter synthesize Windows virtual-key codes.
- **UI-004b:** Broiler.Input.Text gains an explicitly marked paste event carrying text,
  transaction identity, start/end flags, and untrusted provenance. It is not collapsed to
  ordinary text before the UI paste policy runs. Existing key-event fields define zero as
  “unavailable” for native/scan code and one as the synthetic repeat count; synthetic
  balanced transitions use `InputEventSource.Synthetic`.
- **UI-004c:** The adapter maps the Terminal event timestamp/sequence into
  `InputEventHeader`, assigns stable per-session proxy device IDs, and preserves ordering
  across keyboard, mouse, text, paste, focus, and resize dispatch on the UI actor.

### 16.2 Initial Broiler.UI host adapter

The current synchronous `IUiHost.Present(BRenderList)` contract cannot preserve Terminal's
asynchronous backpressure, commit result, cancellation, and fault propagation. Preview 1
therefore requires a Broiler.UI migration: an awaited `RenderFrameAsync`/async-present host
path that leaves the existing synchronous path available for existing GUI hosts. The
terminal host uses only the awaited path; it never blocks a UI dispatcher on a `ValueTask`
and never fire-and-forgets a presentation.

- **UI-005:** The adapter declares a fixed logical cell rectangle. UI viewport size is
  terminal columns/rows multiplied by those logical cell dimensions.
- **UI-006:** Mouse cell coordinates map through the same dimensions used by projection.
- **UI-007:** The adapter projects each full UI replay to one complete immutable ideal
  frame and passes it to `TerminalSession`. It does not retain/diff a physical frame and
  does not require dirty rectangles from Broiler.UI.
- **UI-008:** The projection table covers every closed `BRenderCommand` and returns a
  visible/testable fallback or diagnostic for unsupported commands.
- **UI-009:** `Broiler.UI.Terminal` contains the host/projector. An optional
  `Broiler.UI.Terminal.Standard` depends on `Broiler.UI.Standard` and owns the grid-aligned
  Standard theme. The base terminal host does not claim to theme Standard controls without
  that package. Style/color downgrade remains in Terminal presentation.
- **UI-010:** Text measurement uses the exact same width profile as cell rendering. A
  monospace Graphics font is not a substitute for grapheme width.
- **UI-011:** Preview 1 MAY use process-global Graphics text metrics and Standard theme
  state only after their owning repositories add explicit exclusive leases that capture/
  restore prior state, reject overlapping incompatible leases, and are disposed after UI
  rendering stops. Preview 1 documents that simultaneous heterogeneous hosts remain
  unsupported.
- **UI-012:** Robust multi-host support requires Broiler.UI/Graphics to make text metrics
  and theme resolution session/host-scoped. This is a Phase 6/stable heterogeneous-host
  gate, not a Preview 1 gate or hidden global mutation.
- **UI-013:** UTF-16 code-unit editing/hit-testing in controls MUST be audited and changed
  to grapheme/terminal-width-safe behavior before Unicode text-editing parity is claimed.
- **UI-014:** The terminal event loop serializes input, resize, dispatched UI work, layout,
  and render through one UI context even if the current `UiSession` dispatcher does not
  enforce that invariant itself.
- **UI-015:** Integration builds MUST pin mutually compatible Terminal, UI, Input, and
  Graphics revisions. A command used by a control MUST exist in the Graphics revision
  actually vendored by UI; a newer sibling workspace is not evidence that it does.
- **UI-016:** The terminal host implements `IUiTextInputHost`: published caret geometry is
  converted with the same cell rectangle and controls `TerminalFrame.Cursor`. Clipboard
  support is a separate injected `IUiClipboardHost`; OSC 52 remains disabled and no
  desktop clipboard is implied. The host prevents duplicate drawn/terminal caret policy.
- **UI-017:** Broiler.UI owns sensitive-control redaction before it records a render list or
  semantic/plain snapshot. Password controls contribute only masked display cells and
  redacted semantics. The Terminal core cannot infer UI sensitivity and never claims that
  `Hidden` style is privacy metadata.

### 16.3 Graphics projection baseline

Text, solid fills, rectangular clips, and grid-aligned translation are the baseline.
Borders may use theme-selected ASCII or box-drawing cells. Rounded rectangles, triangles,
non-grid transforms, alpha blending, and images MUST NOT claim faithful support without
explicit conformance. Alternative text/placeholder content is preferable to silently
dropping meaningful UI.

Because `BRenderCommand.DrawImage` carries no alternative text, Preview 1 renders a generic
`[image]` placeholder plus a structured diagnostic. Meaningful alternative text requires a
future UI semantic/alternative-content side channel; the projector MUST NOT claim to infer
it from the image handle.

After the narrow Phase 5A vertical slice, Broiler.UI maintainers record an ADR deciding
whether projection is sufficient or whether host/render contexts need a
presentation-neutral contract. The broader control matrix starts only after that decision.

## 17. Accessibility

- **A11Y-001:** A raw TUI MUST NOT claim to transmit the Broiler.UI semantic tree to an
  arbitrary outer terminal. Screen-reader behavior depends on the frontend.
- **A11Y-002:** Cursor position for active text editing, stable visible text, focus order,
  reduced repaint noise, high contrast, keyboard-only operation, and an optional linear
  mode MUST be tested with named terminal/screen-reader combinations before a claim.
- **A11Y-003:** Password/private fields MUST not expose their value through diagnostics,
  plain fallback, semantic snapshots, copied buffers, or accessibility helpers. For a TUI
  host this is a Broiler.UI pre-render/pre-snapshot obligation as defined by `UI-017`.
- **A11Y-004:** Alternate screen and visual erasure do not guarantee secret removal from
  terminal scrollback, multiplexer logs, remote logging, screen readers, or capture tools.
- **A11Y-005:** A future embedded TerminalView exposes bounded, snapshot-versioned logical
  text ranges, selection, caret, visible bounds, and links—not one accessibility node per
  cell.

No preview accessibility statement is allowed without an attributable test record for a
named revision.

## 18. Security and privacy

- **SEC-001:** Terminal input, capability/environment strings, paste, query replies, and
  future child output are untrusted.
- **SEC-002:** Text/control separation is structural. Sanitization at the last write alone
  is not sufficient.
- **SEC-003:** OSC title/link/clipboard, DCS, APC, notifications, shell integration, image
  protocols, and URL launch are separate feature-policy actions. Unsupported/denied
  actions have no OS side effect.
- **SEC-004:** URI activation and clipboard mutation require a host/user policy. The core
  never opens a link or talks to a desktop clipboard directly.
- **SEC-005:** Diagnostics use a closed schema and never contain raw environment values,
  input bytes, query replies, paste/text content, titles, link URIs, or private control
  values. They MAY contain allowlisted normalized enums, lengths, sequence numbers, profile
  IDs, error codes, and one-way fixture identifiers. Free-form backend messages are not
  public diagnostic fields.
- **SEC-006:** Parser/input reply rates and outstanding requests are bounded to prevent
  loops and amplification.
- **SEC-007:** A frame containing secret text remains ordinary managed memory. The
  component makes no secure-erasure guarantee and documentation MUST say so.
- **SEC-008:** Native backend code, a future PTY launcher, and any SSH dependency require
  revision-scoped human/security review before production claims.
- **SEC-009:** An attached-presenter threat model is approved in Phase 0 and reviewed again
  after the Phase 2 protocol implementation before Preview 1. Phase 8 performs the final
  revision-scoped review; it is not the first review of untrusted-byte handling.

## 19. Faults and diagnostics

Fault categories include at least unsupported environment, invalid profile, invalid
render data, input limit, output failure, disconnect/EOF, mode configuration, mode
restoration, native API, and internal invariant failure.

The error channel is fixed per operation:

| Operation/failure | Channel |
| --- | --- |
| Invalid arguments, disposed object, concurrent second event enumerator | standard exception before side effects |
| Caller cancellation | `OperationCanceledException`; write state is conservatively unknown when writing had begun |
| Open/configuration/rollback failure | `TerminalOpenException` with primary and rollback faults |
| Expected present outcome (`StaleFrame`, unsupported metadata, disconnected, output failure) | typed result; terminal fault is also ordered into the session event stream when session state changes |
| Decoder/input limit/EOF | ordered `Faulted` or `Disconnected` event followed by stream completion |
| Invariant/unexpected failure | session enters `Faulted`; result/event preserves a typed internal fault |
| Cleanup/restoration failure | `TerminalCloseException` from `DisposeAsync`, containing the original primary fault plus all cleanup faults |

Idempotent disposal repeats the cached success or the same structured close failure; it
does not rerun terminal mutations.

- **ERR-001:** Each typed exception/result preserves category, operation, recoverability,
  and sanitized backend detail without requiring callers to parse a message. Implementers
  MUST use the channel table above rather than choosing exception versus result ad hoc.
- **ERR-002:** Mode restoration failures are reported even when another failure caused
  shutdown; both errors remain inspectable.
- **ERR-003:** EOF, broken pipe/network disconnect, user cancellation, and child exit in a
  future connection are distinct outcomes.
- **ERR-004:** Diagnostics are delivered to an injected sink and include monotonic
  timestamps/sequence numbers when ordering matters.
- **ERR-005:** The library does not swallow an invariant violation and continue incremental
  presentation with an unknown physical model.

## 20. Testing and conformance evidence

### 20.1 Neutral tests

- Every valid and invalid render command/list invariant.
- Golden ideal frames for clipping, overflow, style, cursor, and Unicode corpora.
- Cell wide-leading/continuation property tests under overwrite, clip, and every resize.
- Diff plans for unchanged, one-cell, row, full, style-heavy, and resize cases.
- Encoded VT replayed into an independent virtual frontend; its final model equals the
  downgraded target frame.
- Frontend input fixtures split at every byte boundary, plus timeout and malformed data.
- Deterministic virtual-clock tests; no timing sleeps in core suites.
- Fuzzing of UTF-8, escape sequences, limits, render lists, resize, and cancellation.
- Concurrency/ownership tests for serialized writes and idempotent disposal.
- Model-based state-machine tests cover open/cancel/read/present/resize/fault/dispose races,
  independent pump progress, event ordering, stale generations, and cleanup deadlines.
- Public-API tests prove that an application cannot obtain a raw active-session byte writer;
  native Windows ingress retains optional fidelity without exposing console structs.
- Trim/AOT analyzers and an actual published smoke run before preview.

### 20.2 Platform tests

Each backend has hardware-independent fake/PTY tests plus a recorded real-host matrix.
Phase 0 freezes exact mandatory OS/runtime/frontend versions and a pairwise coverage plan;
“current,” “common,” or “selected” are not matrix values. Rows include architecture,
libc where applicable, shell, TERM/profile revision, local/SSH, color/Unicode mode, input,
resize, suspend/resume, redirection, cleanup, result, artifact URI, manual/automated status,
skip reason, and known deviations. Mandatory rows block a claim; exploratory rows do not.

### 20.3 Required observable gates

- **TST-001:** An unchanged interactive frame emits no display update. Every successful
  Plain presentation emits its complete snapshot as required by `LIFE-012`.
- **TST-002:** Every successful interactive encoded update produces the target physical
  model in the virtual frontend oracle; every failed/partial update marks it unknown.
  Plain output is compared byte-for-byte with its snapshot serialization instead.
- **TST-003:** The same timestamped input bytes split at every possible read boundary
  produce the same events; timing changes are separate fixtures.
- **TST-004:** Every public operation either rejects invalid input atomically or produces a
  frame satisfying `CELL-002`; property tests exercise arbitrary write/clip/resize orders.
- **TST-005:** Cancellation/fault injection at every open/close step restores all earlier
  acquired state in reverse order.
- **TST-006:** Redirected default output contains only valid UTF-8 graphic text and LF; it
  contains none of the prohibited bytes listed by `LIFE-012`.
- **TST-007:** Text containing escape/control payloads cannot change title, clipboard,
  cursor mode, or hyperlink state.
- **TST-008:** Long-running resize/input/frame and SSH-disconnect soaks have bounded memory,
  handles/file descriptors, and queues.
- **TST-009:** Broiler.UI integration tests await presentation, propagate terminal faults,
  preserve paste provenance, map caret/focus and event headers, restore the metrics lease,
  and never maintain a second physical diff model.

Conformance claims are revision-scoped and link to artifacts. “ANSI compatible,” “xterm
compatible,” “Unix compatible,” and “accessible” are not accepted without an enumerated
feature/matrix definition.

## 21. Packaging and release requirements

- **PKG-001:** Follow Broiler's vendored packaging metadata and lockstep preview version.
- **PKG-002:** Ship XML documentation, deterministic binaries, SourceLink, and symbol
  packages. Tests, demos, diagnostics, fuzzers, and native build tools are not packable.
- **PKG-003:** Platform configurations build only the applicable native/demo projects;
  the neutral core builds/tests on Windows and Linux.
- **PKG-004:** NuGet package dependencies reflect the arrows in section 4. A backend pulls
  the neutral core, never another OS backend.
- **PKG-005:** CI builds/tests neutral, Windows, and Linux legs; packages are validated in
  clean independent consumer projects.
- **PKG-006:** Native assets, Unicode/terminfo-derived data, and third-party code have
  reproducible provenance and required notices.
- **PKG-007:** Public API and TERM/profile claims freeze only after Broiler.UI consumer
  review and a revision-scoped human/security review.

## 22. Deferred embedded-emulator requirements

These constraints reserve a correct future direction but are not Preview 1 conformance:

- A separate `ITerminalConnection` supplies duplex bytes, resize, completion/exit, and
  disconnect. It is not the internal attached-endpoint driver and does not manipulate the
  host TUI modes.
- The transport reader always drains independently of UI rendering. Slow UI coalesces
  versioned damage; it never blocks a PTY until a child deadlocks.
- A streaming application-output parser handles ECMA-48 plus an enumerated DEC/xterm
  profile across arbitrary chunk boundaries and mutates only a bounded model.
- Main/alternate buffers, margins, tabs, saved state, autowrap, cursor, style, title,
  hyperlinks, stable logical line IDs, soft wraps, and bounded scrollback are explicit.
- Parser actions cannot touch clipboard, open URLs, notify the desktop, load images, or
  write replies without policy and serialized session routing.
- ConPTY, Linux PTY, macOS/BSD PTY, and RFC 4254 SSH are separately tested connection
  types. One passing connection does not establish another's semantics.

The roadmap may promote this section into a dedicated emulator specification after the
attached TUI preview has validated the shared value types and Unicode model.

## 23. Preview 1 acceptance summary

Preview 1 is ready only when:

1. the core, `Alternate`, and `Plain` satisfy every Preview 1 Core requirement in the
   applicability matrix and the frozen profile manifest;
2. Windows and Linux attached backends pass their named matrices and restore state;
3. local and SSH-launched-with-PTY demos have the same oracle-level behavior;
4. no-PTY/redirection defaults are safe and plain;
5. input, Unicode, security-limit, fuzz, partial-write, resize, and soak evidence exists;
6. the narrow projector decision ADR is complete, the selected Broiler.UI terminal-host
   design passes the mandatory control matrix, and its async presentation, Input events,
   caret, metrics lease, theme ownership, and sensitive-content responsibilities pass;
7. every mandatory exact-version support row passes, while exploratory/unsupported rows
   are visibly classified; and
8. package, API, dependency, license, Phase 0 threat-model, post-protocol security review,
   and attributable human review are complete for the exact release revision.

Anything less is an experimental build, not a compatibility or production claim.
