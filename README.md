# Broiler.Terminal

> **Architecture draft.** This repository currently defines the intended component,
> contracts, and delivery plan. It does not contain a runtime implementation yet.

Broiler.Terminal is the cell-mode presentation component for Broiler. Its first job is
to let a Broiler application render and receive input in an existing terminal: Windows
Terminal, Windows Console Host, a Linux console or terminal emulator, and the same
environments reached through SSH.

The component follows the architecture of Broiler.Graphics without pretending that a
terminal is a pixel device:

| Broiler.Graphics | Broiler.Terminal |
| --- | --- |
| Platform-neutral render commands | Platform-neutral cell commands and frames |
| Bitmap/reference renderer | Deterministic in-memory terminal oracle |
| Drawable surface | Owned interactive terminal session |
| Backend presentation | VT presenter plus Windows/Linux endpoint |
| Device loss and resize | Disconnect, unknown terminal state, and cell resize |
| Native replay plus fallback | Minimal diff plus conservative full repaint/plain output |

“Identical” therefore means the same component shape: a safe neutral core, explicit
platform packages, immutable data at the boundary, deterministic fallback and tests,
no native handles in public neutral APIs, and composition by the application. It does
not mean copying the graphics API or rasterizing arbitrary pixels into characters.

## First product direction

The initial data flow is a terminal user interface (TUI):

```text
Broiler.UI
  -> awaited Graphics render-list submission
  -> selected Graphics-to-cell bridge (Phase 5A decision)
  -> complete cell-native ideal frame
  -> deterministic diff and VT encoder
  -> attached Windows console or Linux TTY
  -> local terminal, or the user's terminal through SSH
```

An embedded terminal emulator has the opposite flow—ConPTY, a Unix PTY, or an SSH
channel produces untrusted VT output that must be parsed into a screen and scrollback
model. That is a legitimate later `TerminalView` use case for Broiler.UI, but it is not
the first milestone and it will use separate connection and parser contracts.

## Deployment model

| Scenario | Planned adapter | Important detail |
| --- | --- | --- |
| Windows Terminal | `Broiler.Terminal.Windows` | Uses inherited console handles and VT output; there is no Windows Terminal object-model dependency. |
| Windows Console Host | `Broiler.Terminal.Windows` | Uses the same VT-first output path with native console input/mode management. |
| Linux virtual console or terminal emulator | `Broiler.Terminal.Linux` | Uses the attached TTY, `termios`, terminal size/signals, and a tested capability profile. |
| Broiler app run through SSH with a PTY | Backend for the remote OS | SSH already supplies the remote process a console/PTY. No SSH library is needed in the application. |
| SSH command without a PTY, redirected output, or CI | Neutral plain-output policy | Emits complete UTF-8 snapshots containing graphic text and LF only; forced VT is a visibly unsafe caller assumption, not Plain mode. |
| Broiler initiates an SSH session | Deferred `Broiler.Terminal.Ssh` adapter | Separate remote-session role; it must not implement or weaken SSH cryptography/host verification. |
| Broiler hosts a child shell | Deferred platform connection types | ConPTY/Unix PTY plus a full emulator parser are a later product slice. |

`Broiler.Terminal.Linux` is the first Unix-family backend. macOS, FreeBSD, and other
Unix systems are goals, not aliases for untested Linux ABI assumptions; each earns a
package/support claim through its own validation matrix.

## Planned packages

| Package | Target | Role |
| --- | --- | --- |
| `Broiler.Terminal` | `net10.0` | Safe managed cell model, immutable render-list snapshots, ideal-frame and virtual-frontend oracles, diff planner, safe session, VT codecs, capability and security policy. |
| `Broiler.Terminal.Windows` | `net10.0-windows` | Attached Windows console detection, mode leasing/restoration, native input, resize, and diagnostics. Later, separate ConPTY connection types may live here. |
| `Broiler.Terminal.Linux` | `net10.0` | Attached Linux TTY, `termios`, size/signal integration, capability discovery, and diagnostics. Later, a native PTY launcher may live here. |
| `Broiler.Terminal.Testing` | `net10.0` | Optional scripted internal-driver harness, virtual terminal oracle, fixtures, and conformance helpers if reuse justifies publishing it. |

There is no all-platform meta-package. An application selects the backend it actually
runs on. The neutral package has no dependency on Broiler.Graphics, Broiler.Input,
Broiler.UI, a native handle type, or an SSH implementation.

Applications receive a safe `TerminalSession`, not a raw byte endpoint. Windows, Linux,
and test packages use a friend-internal driver SPI so native input can retain optional
fidelity without exposing console records, handles, or an arbitrary active-session writer.

## Broiler.UI integration

The first integration belongs in the UI/Input/Graphics integration layer rather than in
this neutral core:

```text
Broiler.UI.Terminal
  -> Broiler.UI
  -> terminal-to-Broiler.Input adapter
  -> internal Broiler.Graphics-to-terminal-cell projector
  -> Broiler.Terminal

Broiler.UI.Terminal.Standard
  -> Broiler.UI.Terminal
  -> Broiler.UI.Standard
```

Broiler.UI currently records `BRenderList` graphics commands and measures text through
Broiler.Graphics. The initial host therefore needs an explicit, deterministic projection
to cells and a terminal-aware text-metrics/theme policy. Arbitrary images, rotations,
alpha blending, and sub-cell geometry cannot promise visual parity and must have visible,
tested fallbacks. The longer-term decision—retain that projector or generalize the UI
rendering contract—is an early Phase 5A evidence gate before the broad control matrix.

The current UI `Present` call is synchronous while Terminal presentation is asynchronous.
Preview 1 therefore includes an awaited UI render/present seam; it never blocks the UI
dispatcher on terminal I/O or fire-and-forgets a frame. Standard theming lives in the
optional `.Standard` integration package, and marked paste/caret/focus paths are explicit.

## Design principles

- Cells contain extended grapheme clusters, not UTF-16 code units.
- Unicode segmentation and terminal width data are versioned; ambiguous-width behavior
  is an explicit session policy.
- User text can never become a terminal control sequence. Titles, hyperlinks, clipboard,
  and other side effects use separate, policy-checked APIs.
- Independent driver pumps feed one nonblocking session actor that serializes mode changes,
  input decoding, resize, frame presentation state, and protocol replies. No frame is
  considered committed after a partial write.
- Backend detection produces a typed capability set and diagnostics. A `TERM` value is
  evidence, not permission to emit every extension ever associated with that name.
- Disposal restores all modes the backend changed, on a best-effort basis, in reverse
  order. Crash-proof restoration is impossible and is never claimed.
- The same ideal frame under the same profile has the same planned cell/protocol result on
  every backend. Font shaping, bidi display, and frontend quirks are named compatibility
  observations rather than pixel-equivalence claims.

## Documentation

- [Design draft](docs/design-draft.md) — boundaries, rationale, package ownership, and
  Broiler.UI integration.
- [Specification](docs/specification.md) — proposed normative behavior and public contract
  shape.
- [Roadmap](docs/roadmap.md) — implementation phases, cross-repository work, and exit
  gates.
- [Preview 1 profile](docs/profiles/preview-1.md) — normative Unicode, width, color,
  protocol, mode, and input choices for deterministic fixtures.

The external standards and platform documents used by the draft are linked from the
[design draft](docs/design-draft.md#standards-and-primary-references).

## Status and security

This is pre-preview design work. Terminal byte streams, capability reports, environment
variables, pasted content, and future child-process output are untrusted input. No
production, compatibility, accessibility, or security claim exists until the relevant
roadmap gates have recorded evidence for a named revision.

## License

Broiler.Terminal is licensed under the [Apache License 2.0](LICENSE). The license provides
the software on an “AS IS” basis, without warranties or conditions.
