# thetaurean/zellij — fork notes

This fork tracks upstream `zellij-org/zellij` and adds pane-targeting
primitives that gabi (https://github.com/thetaurean/gabi — TBD) needs to
place panes deterministically without depending on the user's focus state.

## Patches in this fork

### v0.44.3-gabi.1 — `new-pane --target-pane <name|id>` (Patch 1 of 3)

Adds a CLI flag that anchors `zellij action new-pane` to a named target
pane rather than the currently focused pane:

```sh
zellij action new-pane --target-pane editor --direction Right --name thread
zellij action new-pane --target-pane terminal_3 --direction Down --name drawer
```

`<name|id>` accepts:
- A custom pane title (set via `--name` on spawn or `RenamePane`).
- A process title (the foreground command's name, e.g. `vim`).
- A pane-id form: `terminal_<n>`, `plugin_<n>`, or a bare `<n>` which
  defaults to `terminal_<n>`.

If two tiled panes share a name, the call errors with an "ambiguous"
message; pass the pane-id form (`terminal_3`) to disambiguate.

#### Scope and known gaps

This patch is **CLI-only for v1.**

- **Plugin API.** The protobuf path that plugins use to dispatch actions
  back to the server explicitly rejects `NewPanePlacement::TiledNearTarget`
  with `Err("NewTiledPane target_pane is not supported in plugin API
  protobuf")`. Plugins continue to spawn panes via the existing focused-
  pane-anchored placement. A future patch (1b) will expose target-pane to
  plugins once gabi's plugin-host migration needs it.

- **KDL keybinds.** The KDL parser (`zellij-utils/src/kdl/mod.rs`) does not
  yet accept a `target_pane` field on `Run` keybinds. Keybind-triggered
  `NewTiledPane` actions continue to construct `NewPanePlacement::Tiled`
  with no target. Use a CLI command in a keybind body (`zellij action
  new-pane --target-pane ...`) as a workaround until KDL parsing lands.

Upstream may not accept this CLI-only shape; expect a request to wire the
keybind and plugin paths before any zellij-org/zellij PR is merged.

### v0.44.3-gabi.2 — `move-pane --to-pane-id` (Patch 2 of 3)

Adds a CLI flag that structurally re-parents a tiled pane to land adjacent
to a target pane in a requested direction:

```sh
zellij action move-pane Right --pane-id terminal_5 --to-pane-id terminal_3
zellij action move-pane Down  --pane-id terminal_5 --to-pane-id 3
```

The new flag requires both `--pane-id` (the source) and a direction
positional argument. Pane IDs accept the standard `terminal_<n>`,
`plugin_<n>`, or bare `<n>` (= `terminal_<n>`) forms.

Direction semantics mirror Patch 1's `--target-pane`:

- `Left`/`Right`: source lands as a sibling of target's *whole
  column-strip* (anything sharing target's `x` and `cols`). The strip
  slides aside; source spans the strip's combined height.
- `Up`/`Down`: source lands inside target's slot only (vstack with
  target).

Source pane vacates its old location; neighbors grow to fill the freed
space (same `fill_space_over_pane` semantics as `close-pane`).

#### Scope and known gaps

This patch is **CLI-only for v1.**

- **Name resolution.** Unlike Patch 1's `--target-pane` (which accepts
  custom pane names and process titles), `--to-pane-id` accepts only
  numeric pane IDs (`terminal_<n>`, `plugin_<n>`, or `<n>`). The flag
  name signals the convention; callers that have a name must resolve
  it to an id first.

- **Plugin API.** Plugins cannot dispatch `Action::MovePaneToPaneId`;
  the protobuf converter returns the standard
  `"These are CLI-only actions, not available in keybindings"` error
  used by every other `*ByPaneId` variant.

- **KDL keybinds.** Not yet wired to the KDL parser. Use a CLI command
  in a keybind body (`zellij action move-pane ...`) as a workaround.

Cases that error cleanly (no panic, exit status 1, clear message):

- `source == target`
- Source is a floating pane (use a tiled source)
- Target is a floating pane
- Source is a stacked pane (not yet supported)
- Fullscreen is active (exit fullscreen first)
- Source and target are already in the same insert-group (move would
  be a no-op or ambiguous)
- Source or target pane doesn't exist in the tab

### v0.44.3-gabi.3 — `close-pane --absorb-to` (Patch 3 of 3, pending)

Not yet implemented.

## Versioning

Tags follow `vX.Y.Z-gabi.N` where `X.Y.Z` is the upstream zellij version
and `N` is the sequential fork-patch number.

## Building

Same as upstream. Plugin WASM blobs must be built before a workspace
`cargo build`:

```sh
cargo build --target wasm32-wasip1 --release -p status-bar -p tab-bar  # etc.
cargo build --release
```

For iterative work on the changed crates only (skips the plugin WASM
include_bytes):

```sh
cargo check -p zellij-server -p zellij-utils -p zellij-client
```
