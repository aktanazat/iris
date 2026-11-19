# Architecture notes

A map of how iris is put together: the processes, who owns each piece of state,
the boundaries between them, and the invariants the code relies on but does not
enforce. Written as the entry point for the OD-1 improvement pass; each
unenforced invariant below is a tracked candidate (C1–C7) that later pass stages
either fix or retire with evidence.

Scope: one crate (`iris-screenshot`), one binary (`iris`), two front doors — a
clap CLI and a stdio MCP server — over one shared capture pipeline. There is no
library target, no daemon, and no persistence: state lives in the running
process and in Chrome's.

## Processes and crates

```
┌───────────────────────────── iris process ─────────────────────────────┐
│                                                                        │
│  main.rs            mcp.rs                     capture.rs              │
│  ┌───────────┐     ┌───────────────┐          ┌───────────────────┐    │
│  │ CLI       │      │ MCP server    │          │ capture pipeline  │    │
│  │ (clap)    │      │ (rmcp, stdio) │          │ + Session         │    │
│  └─────┬─────┘      └──────┬────────┘          └────────┬──────────┘    │
│        │                   │                            │               │
│        └────────┬──────────┘                            │               │
│                 ▼                                       ▼               │
│          Session::capture ─────────────────────► chromiumoxide           │
└──────────────────────────────────────────────────────┬───────────────────┘
                                                       ▼
                                              ┌──────────────────┐
                                              │ Chrome process   │
                                              │ + temp profile   │
                                              └──────────────────┘
```

- **iris process** — one Tokio multi-thread runtime. CLI mode fans batches out
  onto it; MCP mode serves JSON-RPC on it.
- **Chrome process** — launched by `chromiumoxide` with a unique temp user-data
  dir (`$TMP/iris-chrome-<pid>-<n>`). All page work is CDP over that process.
- **No third process, crate, or store.** `ProfileDir` cleanup and the CDP event
  pump are the only auxiliary state, both owned by `Session`.

## State ownership

| State | Owner | Lifecycle |
|---|---|---|
| Chrome process, CDP event-pump task, temp profile dir, scrubbed user agent | `Session` (`capture.rs`) | `Session::launch` creates all four together; `Session::close` (explicit) or field drop (fallback) tears them down. `ProfileDir` cleans its directory in `Drop`; `profile_dir` is declared after `browser` so fallback field-drop stops chromiumoxide's child before the dir vanishes. `is_healthy` reports whether the pump task is still alive. |
| Batch fan-out (per-URL task set, `buffer_unordered(jobs)`, JSON/human reporting, exit code) | `run_capture` (`main.rs`) | Builds `Opts` once, shares one `Arc<Session>` and one `Arc<Opts>` across every job, drains the stream, then closes Chrome via `Arc::try_unwrap(session)`. Process exits 1 when any capture failed. |
| Warm-session lifecycle (lazy launch, health-checked reuse, relaunch after unhealthy captures, 4-permit capture queue) | `McpState` (`mcp.rs`) | `Mutex<Option<Arc<Session>>>` created on first capture; a capture whose session went unhealthy takes the slot (under the mutex, `Arc::ptr_eq` checked so a concurrent relaunch wins) and the next capture launches a fresh session. `Semaphore(4)` bounds concurrent tabs. `McpState::close` unwraps and closes the session at server shutdown. |
| Per-page emulation state (device metrics, UA override, dark-mode media) | the `Page` | Set by `Session::pipeline` per capture; dies with the tab, which `Session::capture` always closes. |
| Tool input validation | `CaptureRequest::prepare` (`mcp.rs`) | Single validation point for the MCP tool; produces `PreparedCapture` (url, output, `Opts`). |
| Output-path derivation and format settlement | `resolve_outputs` (`main.rs`) for CLI; `CaptureRequest::prepare` for MCP | Both apply the same rule: an output extension beats a requested format. CLI derives batch names `<host>-<path>[-<n>]` and creates parent dirs; `CapturedImage::write_to` also creates parents (duplicated rule, C6). |

## Boundaries

1. **CLI → pipeline.** `main.rs` parses clap args, calls `parse_viewport` and
   `capture_mode`, resolves output paths, then drives `Session::capture` per
   URL. Formatting of human lines and JSON Lines stays in `main.rs`.
2. **MCP → pipeline.** An MCP client speaks JSON-RPC over stdio to `iris mcp`;
   rmcp routes the single `capture` tool to `IrisServer::capture_image`, which
   runs `CaptureRequest::prepare` then `McpState::capture`. Tool results carry
   the image inline (base64 content block), a text summary, and the structured
   report; errors reuse the same `ErrorReport` shape.
3. **Rust ↔ page JS via CDP eval.** `Session::eval` runs a JS snippet
   (`await_promise`, `return_by_value`), surfacing page exceptions as errors.
   The snippets live at the bottom of `capture.rs`: `SETTLE_JS` (fonts, near
   images, finite animations, double rAF), `SCROLL_JS` (step-scroll for
   lazy-loading, back to top), `DOC_HEIGHT_JS`, `wait_for_js` /
   `wait_and_scroll_js` (budget-capped selector waits), `element_bounds_js`.
   Budget-capped waits undercut the outer per-page timeout by 500 ms so the
   descriptive selector error wins the race.
4. **capture pipeline → Chrome.** `chromiumoxide` drives CDP: metrics/UA/media
   emulation, navigation, screenshots. The event-pump task tolerates
   `CdpError::Serde` noise (newer Chrome messages the crate can't deserialize)
   and stops on real transport errors — `is_healthy` keys off that.

## Unenforced invariants (tracked candidates)

Each is relied on by convention today; the pass either fixes it or retires it
with evidence. **C1–C6 are seeded from the approved spec; C7 was found while
writing this map.**

| ID | Invariant relied on but not enforced | Where |
|---|---|---|
| C1 | `--selector` / `--wait-for` are non-empty: the CLI accepts empty or whitespace-only values and fails mid-capture with a page-side JS error, while `CaptureRequest::prepare` rejects both up front. | `main.rs` `capture_mode`; `mcp.rs` `prepare` |
| C2 | `-` appears at most once among URL args: a second `-` re-reads stdin (yielding nothing) and silently loses that input. | `main.rs` `collect_urls` |
| C3 | Full-page capture fits Chrome's ~16k-pixel texture limit in both dimensions: only height × scale is checked before choosing the retina path; a wide viewport can still exceed the limit in width. | `capture.rs` full-page branch |
| C4 | Timeout > 0: the CLI silently coerces `--timeout 0` to 1 s (`timeout.max(1)`); the MCP path rejects `timeout_seconds: 0`. | `main.rs` `run_capture`; `mcp.rs` `prepare` |
| C5 | `--jobs` is bounded: any value is accepted and fans out that many concurrent tabs against one Chrome. | `main.rs` `run_capture` |
| C6 | Output parent directories are created once: `resolve_outputs` and `CapturedImage::write_to` both create them — a duplicated rule to fold into the refactor. | `main.rs`; `capture.rs` |
| C7 | One Chrome stays healthy for a whole CLI batch: jobs share one `Arc<Session>` with no health check or relaunch, so a Chrome crash fails every remaining job. The MCP path (`McpState::capture`) is the counter-example — it detects unhealthy sessions and relaunches. | `main.rs` `run_capture`; `mcp.rs` `McpState::capture` |

Two further conventions are documented here because later stages lean on them,
though they are enforced (by tests or types) rather than merely assumed: JSON
Lines errors go to stdout, never stderr (`tests/cli.rs::json_batch_reports_
success_and_failure_on_stdout`), and selectors are first-match-only, settled in
the in-module browser contract test.

## Where the tests live

- **Pure / no browser** — clap conflict and mode-construction tests, JSON report
  shapes (`main.rs`), `round_clip`, `normalize_url`, `parse_viewport`
  (`capture.rs`), `CaptureRequest::prepare` rules and tool-schema shape
  (`mcp.rs`).
- **Browser-dependent (need Chrome-family binary)** — `browser_element_capture_
  contract` (`capture.rs`), `capture_tool_returns_inline_pixels_metadata_and_
  optional_file` (`mcp.rs`), and in `tests/cli.rs`: `json_batch_reports_…`,
  `mcp_stdio_lists_one_tool_and_returns_an_inline_capture`,
  `mcp_sigterm_closes_chrome_and_removes_its_profile`. All drive the
  `tests/fixtures/precise-capture.html` fixture through `file://` URLs; CI is
  the authority for them on machines without Chrome.

## Relationship to the improvement pass

- **D2 correctness** fixes C1–C5 (each with a reproducing test); candidates that
  do not reproduce are retired here with evidence.
- **D3 performance** measures against the README's methodology; this map is its
  "who owns the work" reference.
- **D4 refactor** splits `capture.rs` along the ownership lines above (session
  lifecycle / pipeline / page JS / reports / parsing), unifies CLI+MCP mode
  construction and conflict rules, and gives output-dir creation a single owner
  (C6).
- **D5 README-gap audit** checks the promises in `README.md` against the
  behaviour mapped here.
