# Broiler.Terminal Preview 1 profile

**Profile ID:** `broiler-attached-preview1-v1`  
**Manifest revision:** 1  
**Status:** Normative for the 0.2 draft  
**Date:** 2026-09-03

This manifest closes the deterministic choices used by Core fixtures. A release records
the SHA-256 of this file and of its generated Unicode tables in frame snapshots,
diagnostics, and conformance artifacts. Named frontend quirk overlays may disable a
feature or select a listed strategy; they may not change ideal-frame semantics.

## Unicode and cells

- Unicode data and extended grapheme segmentation use Unicode 17.0.0 and UAX #29 revision
  47. No normalization or bidi reordering is performed.
- Default ambiguous width is one column. An explicit session option selects two columns
  and changes the width-profile ID.
- EAW `W`/`F`, RGI emoji presentation sequences, RGI flags/keycaps, and recognized emoji
  ZWJ sequences occupy two cells. EAW `A` follows the session option. Other printable
  clusters occupy one cell.
- Variation Selector-16 selects the emoji rule when the sequence is recognized; Variation
  Selector-15 selects text width. An unrecognized cluster uses U+FFFD at one column.
- A leading combining-only cluster is rendered as U+25CC plus the marks at width one.
- Invalid UTF-8 or UTF-16 is replaced by U+FFFD by the default policy; the strict profile
  rejects it. Cluster, list, and aggregate limits come from `TerminalLimits`.
- LF, CRLF, TAB, and prohibited-control behavior is exactly `CELL-009`; tab stops start at
  columns 0, 8, 16, and so on.

## Colors and styles

- The nominal ANSI-16 palette is xterm's conventional black/red/green/yellow/blue/magenta/
  cyan/white plus bright variants: `000000`, `CD0000`, `00CD00`, `CDCD00`, `0000EE`,
  `CD00CD`, `00CDCD`, `E5E5E5`, `7F7F7F`, `FF0000`, `00FF00`, `FFFF00`, `5C5CFF`,
  `FF00FF`, `00FFFF`, `FFFFFF`.
- Indexed 16–231 use the 6×6×6 cube with levels `00,5F,87,AF,D7,FF`; 232–255 use gray
  levels `08 + 10*n`. RGB downgrade compares squared Euclidean distance in sRGB bytes and
  chooses the lower palette index on a tie.
- Unsupported `Hidden` becomes styled blank cells. Other unsupported style flags are
  removed. Default colors remain default and are never resolved to nominal RGB.
- Erase controls are used only when their `bce` behavior exactly reproduces the target;
  otherwise explicit styled spaces are emitted.

## Output vocabulary

- Graphic text is UTF-8. Controls use 7-bit `ESC` forms; single-byte C1 is never emitted.
- Required interactive controls are CUP (`CSI r;cH`), SGR/reset, DECSET/DECRST 1049, and
  cursor visibility 25. DECSCUSR, EL/ED/ECH, mouse, focus, bracketed paste, synchronized
  output, OSC 2, and OSC 8 are emitted only when the effective capability enables them.
- Preview 1 enables bracketed paste with mode 2004, focus with 1004, and SGR mouse with
  1006 plus tracking 1000 or 1002. Cleanup disables them in reverse order.
- No active query is enabled by the default profile.
- The encoder does not rely on automatic wrap for cursor placement: every noncontiguous
  run starts with CUP. A run ending in the bottom-right cell must be the last graphic write
  in its update. Any later control begins with CUP, clearing delayed-wrap state. A frontend
  that scrolls immediately on the bottom-right write requires a quirk overlay that enables
  a tested autowrap-disable strategy or forbids that cell and reports the limitation.
- SGR planning minimizes encoded byte count. Ties prefer fewer control sequences, then the
  plan that writes the earlier row/column first, then full-row over sparse repaint. Screen
  repaint ties prefer the full repaint.

## Input vocabulary

- Required byte-stream keys are Escape, Enter, Tab, Backspace, arrows, Home, End, Insert,
  Delete, PageUp, PageDown, F1–F12, and Shift/Alt/Ctrl modifier forms enumerated by fixtures.
- Required optional protocols are bracketed paste, focus reporting, and SGR mouse when
  their modes were enabled by this session. No enhanced keyboard protocol is claimed.
- Escape ambiguity timeout defaults to 50 ms and is measured by the injected monotonic
  clock. Conformance fixtures carry byte-arrival timestamps and EOF flush markers.
- Unknown/malformed sequences reset to ground according to the checked-in fixture grammar;
  paste overflow follows `IN-018` and never reinterprets discarded payload.

## Modes and claims

- Preview 1 interactive mode is `Alternate`; mode 1049 and absolute cursor addressing are
  required. `Inline` returns `UnsupportedMode`.
- `Plain` follows `LIFE-012` and emits no terminal controls.
- Attached mode consumes the inherited `TERM`/environment as untrusted hints and never
  advertises a `TERM` value to another process.
- Actual glyph shape, font fallback, bidi presentation, and host width deviations are
  named compatibility observations. Oracle equality covers ideal cells and the planned
  terminal state, not pixel-identical frontend rendering.
