# shuck — agent context

A streamed shell-command picker for neovim (a vim-grepper / live-grep
replacement). The user edits an arbitrary shell command in a prompt window; its
stdout/stderr stream live into a results window as the command runs. Renders in
plain neovim splits, not floats.

See `README.md` for the user-facing docs (commands, keymaps, setup options).

## Layout

- `lua/shuck/init.lua` — the entire implementation (single module `M`, ~770
  lines). Organized into banner-delimited sections (search for
  `-- ===`):
  - **Config** — `M.config` defaults (`default_cmd`, `max_render`,
    `max_results`, `stream_flush_ms`, `spinner_ms`).
  - **State** — a single module-local `state` table holding the live picker
    (buffers, windows, child process, output lines, history, selection, run
    epoch). `nil` when the picker is closed. Highlight namespaces and spinner
    frames live here too.
  - **Helpers** — child process kill, window title/spinner, close, search-dir
    discovery (`discover_search_dir`: cwd if buffer under it, else nearest git
    root, else buffer dir).
  - **History** — per-directory command history persisted as JSON under
    `stdpath("data")/shuck/` (path hashed by cwd). Load/save/record + cycle
    state.
  - **Rendering** — `render_results` redraws the results buffer (capped at
    `max_render`) with highlights; `schedule_flush` debounces redraws during
    streaming by `stream_flush_ms`.
  - **Streaming chunk handler** — `handle_chunk` splits incoming stdout/stderr
    data into complete lines, buffers the trailing partial line, applies path
    rewriting, and enforces the `max_results` cap.
  - **Command execution** — `run_command` is the core: records history, bumps a
    `run_epoch` (used to ignore stale async callbacks from a prior run), clears
    output, starts a spinner timer, and launches `vim.system({"sh","-c",cmd})`
    with streaming stdout/stderr callbacks (all wrapped in `vim.schedule`).
  - **Selection / actions** — `move_selection`, `open_selected` (split/vsplit/
    tab variants), `to_quickfix`.
  - **History UI** — a secondary in-results-window picker mode
    (`picker_mode`) for browsing history (`<C-r>`), plus `<Up>`/`<Down>`
    prefix-match cycling.
  - **UI: floating windows + keymaps** — `open_windows` builds the prompt +
    results splits; `setup_keymaps` wires the prompt-buffer keymaps.
  - **Public API** — `M.open`, `M.toggle`, `M.close`, `M.setup`.

- `doc/shuck.txt` — neovim help docs (`:help shuck`). `doc/tags` is generated
  and gitignored.
- `README.md` — user docs.

## Key architectural notes

- **Single live instance.** Only one picker exists at a time, in the module
  `state`. Opening reuses/recreates it; closing tears it down and sets
  `state = nil`.
- **Epoch guarding.** Async stdout/stderr/exit callbacks check
  `state.run_epoch == epoch` before touching state, so output from a
  superseded command run is discarded.
- **Path rewriting.** When the search cwd differs from neovim's cwd, result
  paths are rewritten so `<CR>`/quickfix open the right file
  (`maybe_rewrite_path`, gated by `state.rewrite_paths`).
- **No external dependencies.** Pure neovim API + `vim.uv`/`vim.system`.
  Requires Neovim 0.12+.

## Tests

There is no automated test suite or CI in this repo. Verify changes manually:

```vim
:Shuck
```

Or load the local copy and exercise it interactively. On the maintainer's
machines this repo is checked out at `~/src/shuck` and loaded directly from
there (rtp prepend) on macOS, while Linux fetches it from GitHub via
`vim.pack`, so local edits apply immediately on macOS.
