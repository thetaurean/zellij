# Fixed-Size Tiled `new-pane` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let `zellij-gabi action new-pane` set a fixed (or percent) size for **tiled** (non-floating) spawns by overloading the existing `--width`/`--height` flags, so `new-pane --target-pane editor --direction down --height 2` lands a tiled pane exactly 2 rows tall.

**Architecture:** Carry an optional `size: Option<SplitSize>` on the two tiled placement variants (`NewPanePlacement::Tiled` and `NewPanePlacement::TiledNearTarget`), threading it the same way `cd923bbb` threaded `--target-pane`: CLI → `Action` → IPC protobuf → server `Screen` → `Tab` → `TiledPanes`. The size is applied at the split site by overriding the new pane's `Dimension` along the split axis to `Fixed`/`Percent` (so it "sticks" through relayout, exactly like a layout-declared `size=N` pane), clamped so both sides keep ≥1 cell.

**Tech Stack:** Rust workspace (`zellij-utils`, `zellij-server`); clap CLI; prost protobuf IPC; Cassowary-based tiled layout.

**Scope decision (locked):** Terminal + command tiled panes only — mirroring `cd923bbb`'s actual reach. **Plugin tiled panes are explicitly OUT of scope.** Plugin panes spawn through a separate stack (`Action::NewTiledPluginPane` → `ScreenInstruction::NewTiledPluginPane` → `PtyInstruction::FillPluginCwd` → `PluginInstruction::Load`) that `cd923bbb` never touched and that supports neither `--target-pane` nor a tiled size today. Task 7 documents this gap. The consumer's plugin call (`--plugin … --target-pane drawer --height 2`) will therefore **not** be unblocked by this plan; that is a known, accepted follow-up.

---

## Background: the `cd923bbb` mirror

`git show cd923bbb` (the "targeted tiled pane spawn" feature) is the file-by-file template. It added the `TiledNearTarget` placement variant and threaded `target_pane`/`direction` through the entire CLI→server stack. This plan adds a sibling field, `size`, the same way. Read that commit before starting.

## Key types (already present — do not redefine)

- `SplitSize` — `zellij-utils/src/input/layout.rs:66`:
  ```rust
  pub enum SplitSize { Percent(usize), Fixed(usize) }
  ```
  Has `impl From<PercentOrFixed> for SplitSize`, `From<SplitSize> for PercentOrFixed`, and `fn to_fixed(&self, full_size: usize) -> usize`.
- `PercentOrFixed::from_str` — `zellij-utils/src/input/layout.rs:765` parses `"2"` → `Fixed(2)`, `"20%"` → `Percent(20)`.
- `Dimension` — `zellij-utils/src/pane_size.rs:84`: `Dimension::fixed(usize)`, `Dimension::percent(f64)`, `set_inner(usize)`, `as_usize()`, `as_percent() -> Option<f64>`.
- `MIN_TERMINAL_HEIGHT = MIN_TERMINAL_WIDTH = 5` — `zellij-server/src/tab/mod.rs:142-143`.
- proto `SplitSize` message already exists (`common_types.proto:1066`, generated `client_server_contract.rs:1747`) with `From<layout::SplitSize> for proto::SplitSize` (`protobuf_conversion.rs:3749`). **The reverse (proto→layout) does NOT exist yet** — added in Task 3.

---

## Task 1: Add the `size` field and make the whole workspace compile (inert)

This is a mechanical, **compiler-driven** task. Adding a field to an exhaustively-matched enum breaks every literal-construction and explicit-destructure site until each is updated. The field stays **inert** (always `None`, never read) at the end of this task — behavior is unchanged. Tasks 3–6 give it meaning.

**Files:**
- Modify: `zellij-utils/src/data.rs` (enum + import)
- Modify (mechanical `size: None` / `size: _`): `zellij-utils/src/input/actions.rs`, `zellij-utils/src/kdl/mod.rs`, `zellij-utils/src/plugin_api/action.rs`, `zellij-utils/src/ipc/protobuf_conversion.rs`, `zellij-utils/src/ipc/tests/roundtrip_tests.rs`, `zellij-server/src/route.rs`, `zellij-server/src/screen.rs`, `zellij-server/src/tab/mod.rs`, `zellij-server/src/plugins/zellij_exports.rs`, `zellij-server/src/unit/screen_tests.rs`, `zellij-server/src/tab/unit/tab_tests.rs`, `zellij-server/src/tab/unit/tab_integration_tests.rs`

- [ ] **Step 1: Add the field to both tiled variants**

In `zellij-utils/src/data.rs`, the enum at line 3240. Change:

```rust
    Tiled {
        direction: Option<Direction>,
        borderless: Option<bool>,
    },
    TiledNearTarget {
        target_pane: String,
        direction: Direction,
        borderless: Option<bool>,
    },
```

to:

```rust
    Tiled {
        direction: Option<Direction>,
        borderless: Option<bool>,
        /// Optional fixed/percent size for the new tiled pane along its split
        /// axis (rows for up/down, cols for left/right). `None` = default 50%.
        size: Option<SplitSize>,
    },
    TiledNearTarget {
        target_pane: String,
        direction: Direction,
        borderless: Option<bool>,
        /// Optional fixed/percent size for the new tiled pane along its split
        /// axis (rows for up/down, cols for left/right). `None` = default 50%.
        size: Option<SplitSize>,
    },
```

- [ ] **Step 2: Import `SplitSize` into data.rs**

In `zellij-utils/src/data.rs`, the import at line 5 currently:

```rust
use crate::input::layout::{
    Layout, PercentOrFixed, Run, RunPlugin, RunPluginLocation, RunPluginOrAlias,
};
```

Change to add `SplitSize`:

```rust
use crate::input::layout::{
    Layout, PercentOrFixed, Run, RunPlugin, RunPluginLocation, RunPluginOrAlias, SplitSize,
};
```

(`SplitSize` derives `Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize` — compatible with `NewPanePlacement`'s `#[derive(Clone, Debug, Serialize, Deserialize, PartialEq, Eq)]`.)

- [ ] **Step 3: Build and let the compiler list every broken site**

Run: `cargo build 2>&1 | grep -E "missing field|error\[" | sort -u`
Expected: a list of `missing field \`size\`` errors (literal constructions) and `pattern does not mention field \`size\`` errors (explicit destructures).

- [ ] **Step 4: Fix every construction site — add `size: None`**

For each **literal struct construction** `NewPanePlacement::Tiled { … }` / `NewPanePlacement::TiledNearTarget { … }`, add `size: None,`. These are the sites (verified by grep at plan time — re-confirm with the compiler):

`NewPanePlacement::Tiled { … }` constructions:
- `zellij-utils/src/input/actions.rs:2334` (in `tiled_placement_from_cli`; rewritten in Task 4 — for now `size: None`)
- `zellij-utils/src/plugin_api/action.rs:383`, `:394`, `:2473`
- `zellij-utils/src/kdl/mod.rs:2029`
- `zellij-utils/src/ipc/protobuf_conversion.rs:2075`, `:3581`, `:3613`
- `zellij-server/src/route.rs:605`, `:721`, `:935`
- `zellij-server/src/screen.rs:5916`, `:8118`
- `zellij-server/src/plugins/zellij_exports.rs:1708`, `:1746`, `:2147`, `:2205`
- `zellij-utils/src/ipc/tests/roundtrip_tests.rs:1407`, `:1451`, `:2620`, `:2658`
- `zellij-server/src/unit/screen_tests.rs:2100`, `:2161`
- `zellij-server/src/tab/unit/tab_integration_tests.rs:12374`, `:12437`

`NewPanePlacement::TiledNearTarget { … }` constructions:
- `zellij-utils/src/input/actions.rs:2328` (in `tiled_placement_from_cli`; rewritten in Task 4 — for now `size: None`)
- `zellij-utils/src/input/actions.rs:3858`, `:3908` (test `==` fixtures)
- `zellij-utils/src/ipc/tests/roundtrip_tests.rs:1422`
- `zellij-utils/src/ipc/protobuf_conversion.rs:3597`
- `zellij-server/src/tab/unit/tab_tests.rs` — all `TiledNearTarget { … }` literals (≈21 sites: 459, 492, 521, 554, 588, 622, 655, 695, 739, 771, 812, 861, 899, 948, 16825, 16848, 16994, 17011, 17093, 17110, 18062)

- [ ] **Step 5: Fix every explicit-destructure match arm — add `size: _`**

Match arms that bind named fields **without** a trailing `..` need `size: _,` added. (Arms that already use `..`, e.g. `Tiled { borderless, .. }`, `Tiled { direction, .. }`, `Tiled { .. }`, `TiledNearTarget { .. }`, need **no change**.) The explicit-destructure arms are:

- `zellij-utils/src/kdl/mod.rs:843` — `NewPanePlacement::Tiled { direction, borderless: _ }` → add `size: _,`
- `zellij-utils/src/plugin_api/action.rs:1377` — `NewPanePlacement::Tiled { direction, borderless }` → add `size: _,`
- `zellij-utils/src/plugin_api/action.rs:2515` — `NewPanePlacement::Tiled { direction, borderless }` → add `size: _,`
- `zellij-utils/src/ipc/protobuf_conversion.rs:3504` — `Tiled { direction, borderless: Some(b) }` → add `size: _,`
- `zellij-utils/src/ipc/protobuf_conversion.rs:3511` — `Tiled { direction, borderless: None }` → add `size: _,`
- `zellij-utils/src/ipc/protobuf_conversion.rs:3515` — `TiledNearTarget { target_pane, direction, borderless }` → add `size: _,`
- `zellij-server/src/tab/mod.rs:1505` — `Tiled { direction: None, borderless }` → add `size: _,`
- `zellij-server/src/tab/mod.rs:1518` — `Tiled { direction: Some(direction), borderless }` → add `size: _,`
- `zellij-server/src/tab/mod.rs:1543` — `TiledNearTarget { target_pane, direction, borderless }` → add `size: _,`

(The three `tab/mod.rs` arms get real `size` bindings in Tasks 5/6; `size: _` now keeps them compiling.)

- [ ] **Step 6: Build clean**

Run: `cargo build 2>&1 | tail -5`
Expected: builds with no errors (warnings about unused `size` are acceptable; `size: _`/`None` produce none).

- [ ] **Step 7: Full test suite green (behavior unchanged)**

Run: `cargo test --workspace 2>&1 | tail -20`
Expected: all existing tests pass — the field is inert, so nothing behaves differently.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(new-pane): add inert size field to tiled placements

Mirrors cd923bbb: threads a new optional field through the placement
enum. Field is unused/None everywhere; behavior unchanged. Wiring and
behavior follow in subsequent commits.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Build a free-standing helper for the split geometry (TDD)

Before wiring, build and unit-test the pure function that turns a requested `SplitSize` + available cells into `(new_pane_cells, other_side_cells)` plus the new pane's `Dimension`. Both server split sites (Tasks 5, 6) call it, so DRY it here.

**Files:**
- Modify: `zellij-server/src/panes/tiled_panes/mod.rs` (add helper + imports)
- Test: `zellij-server/src/panes/tiled_panes/mod.rs` (inline `#[cfg(test)]` module)

- [ ] **Step 1: Add imports**

In `zellij-server/src/panes/tiled_panes/mod.rs`, the `zellij_utils` import block (around line 24) already imports `pane_size::{Dimension, …}`. Extend the `input::layout` import to include `SplitSize`. Find:

```rust
        layout::{Run, RunPluginOrAlias, SplitDirection},
```

change to:

```rust
        layout::{Run, RunPluginOrAlias, SplitDirection, SplitSize},
```

- [ ] **Step 2: Write the failing test**

Add at the bottom of `zellij-server/src/panes/tiled_panes/mod.rs`:

```rust
#[cfg(test)]
mod fixed_size_split_tests {
    use super::sized_split;
    use zellij_utils::input::layout::SplitSize;

    // (new_cells, other_cells)
    #[test]
    fn fixed_smaller_than_total_is_exact() {
        let (new, other, _dim) = sized_split(SplitSize::Fixed(2), 20, /*new_is_lead*/ false);
        assert_eq!((new, other), (2, 18));
    }

    #[test]
    fn fixed_clamps_to_leave_one_cell_for_other_side() {
        let (new, other, _dim) = sized_split(SplitSize::Fixed(100), 20, false);
        assert_eq!((new, other), (19, 1));
    }

    #[test]
    fn fixed_clamps_up_to_at_least_one_cell() {
        let (new, other, _dim) = sized_split(SplitSize::Fixed(0), 20, false);
        assert_eq!((new, other), (1, 19));
    }

    #[test]
    fn percent_resolves_against_total() {
        let (new, other, dim) = sized_split(SplitSize::Percent(20), 20, false);
        assert_eq!((new, other), (4, 16));
        assert_eq!(dim.as_percent(), Some(20.0));
        assert_eq!(dim.as_usize(), 4);
    }

    #[test]
    fn fixed_dimension_is_fixed() {
        let (_new, _other, dim) = sized_split(SplitSize::Fixed(3), 20, false);
        assert_eq!(dim.as_percent(), None);
        assert_eq!(dim.as_usize(), 3);
    }
}
```

- [ ] **Step 2b: Run it to verify it fails**

Run: `cargo test -p zellij-server sized_split 2>&1 | tail -15`
Expected: FAIL — `cannot find function \`sized_split\``.

- [ ] **Step 3: Implement the helper**

Add as a free function near `split` usage in `zellij-server/src/panes/tiled_panes/mod.rs` (e.g. just above `impl TiledPanes` or near `pane_group_split_near_pane_id`). The `new_is_lead` argument is unused by the math but documents intent at call sites; keep the signature uniform:

```rust
/// Resolve a requested `SplitSize` against `total` cells available along the
/// split axis. Returns `(new_pane_cells, other_side_cells, new_pane_dimension)`.
///
/// - `new_pane_cells` honors the request, clamped to `[1, total - 1]` so both
///   sides keep at least one cell. This matches layout-declared `size=N` panes,
///   which are NOT floored at `MIN_TERMINAL_*` (e.g. a 2-row status strip is
///   legal); the `MIN_TERMINAL_* * 2` gate elsewhere only decides whether a
///   split happens at all, not how the room is divided.
/// - `new_pane_dimension` is `Fixed`/`Percent` so the pane sticks through
///   relayout instead of being rebalanced like a plain percent split.
fn sized_split(size: SplitSize, total: usize, _new_is_lead: bool) -> (usize, usize, Dimension) {
    let new_cells = size
        .to_fixed(total)
        .max(1)
        .min(total.saturating_sub(1).max(1));
    let other_cells = total.saturating_sub(new_cells);
    let dimension = match size {
        SplitSize::Fixed(_) => Dimension::fixed(new_cells),
        SplitSize::Percent(p) => {
            let mut d = Dimension::percent(p as f64);
            d.set_inner(new_cells);
            d
        },
    };
    (new_cells, other_cells, dimension)
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cargo test -p zellij-server sized_split 2>&1 | tail -15`
Expected: PASS (5 tests). The `unused function` warning is fine until Tasks 5/6 call it.

- [ ] **Step 5: Commit**

```bash
git add zellij-server/src/panes/tiled_panes/mod.rs
git commit -m "feat(tiled): add sized_split geometry helper

Pure helper that resolves a SplitSize against available cells and yields
a Fixed/Percent Dimension that survives relayout. Unit-tested. Not yet
wired to a split site.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Wire `size` through the IPC protobuf (TDD via roundtrip)

Make `size` survive `Action ↔ protobuf` conversion. It rides inside the placement messages, so only `TiledPlacement` / `TiledNearTargetPlacement` change.

**Files:**
- Modify: `zellij-utils/src/client_server_contract/common_types.proto`
- Modify: `zellij-utils/assets/prost_ipc/client_server_contract.rs` (hand-edit generated struct — matches how `cd923bbb` did it)
- Modify: `zellij-utils/src/ipc/protobuf_conversion.rs`
- Test: `zellij-utils/src/ipc/tests/roundtrip_tests.rs`

- [ ] **Step 1: Write the failing roundtrip test**

In `zellij-utils/src/ipc/tests/roundtrip_tests.rs`, inside `fn test_client_messages()` (alongside the existing `NewTiledPane` roundtrip cases near line 1407), add:

```rust
    // size survives roundtrip on TiledNearTarget (consumer's path)
    test_client_roundtrip!(ClientToServerMsg::Action {
        action: Action::NewTiledPane {
            command: None,
            placement: NewPanePlacement::TiledNearTarget {
                target_pane: "editor".to_owned(),
                direction: Direction::Down,
                borderless: Some(true),
                size: Some(SplitSize::Fixed(2)),
            },
            pane_name: None,
            near_current_pane: false,
            tab_id: None,
        },
        terminal_id: Some(1),
        client_id: Some(100),
        is_cli_client: true,
    });
    // size survives roundtrip on plain Tiled, borderless None (exercises the
    // TiledWithOptions routing for size-without-borderless)
    test_client_roundtrip!(ClientToServerMsg::Action {
        action: Action::NewTiledPane {
            command: None,
            placement: NewPanePlacement::Tiled {
                direction: Some(Direction::Down),
                borderless: None,
                size: Some(SplitSize::Percent(20)),
            },
            pane_name: None,
            near_current_pane: false,
            tab_id: None,
        },
        terminal_id: Some(1),
        client_id: Some(100),
        is_cli_client: true,
    });
```

Ensure `SplitSize` is imported in that test file. At the top of `roundtrip_tests.rs`, find the `use crate::input::layout::{…}` line (it already imports layout types for these tests) and add `SplitSize` if absent. If there is no such import, add:

```rust
use crate::input::layout::SplitSize;
```

- [ ] **Step 2: Run it to verify it fails**

Run: `cargo test -p zellij-utils --lib roundtrip 2>&1 | tail -20`
Expected: FAIL — the roundtrip loses `size` (asserts inequality) because the conversion drops it and the legacy `Tiled(direction)` path can't carry it.

- [ ] **Step 3: Add the proto fields**

In `zellij-utils/src/client_server_contract/common_types.proto`, find (line ~847):

```proto
message TiledPlacement {
  optional Direction direction = 1;
  optional bool borderless = 2;
}

message TiledNearTargetPlacement {
  string target_pane = 1;
  Direction direction = 2;
  optional bool borderless = 3;
}
```

Change to:

```proto
message TiledPlacement {
  optional Direction direction = 1;
  optional bool borderless = 2;
  optional SplitSize size = 3;
}

message TiledNearTargetPlacement {
  string target_pane = 1;
  Direction direction = 2;
  optional bool borderless = 3;
  optional SplitSize size = 4;
}
```

- [ ] **Step 4: Hand-edit the generated Rust to match**

In `zellij-utils/assets/prost_ipc/client_server_contract.rs`, find `pub struct TiledPlacement` (line ~1421) and `pub struct TiledNearTargetPlacement` (line ~1429). Add the `size` field to each:

`TiledPlacement` becomes:

```rust
pub struct TiledPlacement {
    #[prost(enumeration="Direction", optional, tag="1")]
    pub direction: ::core::option::Option<i32>,
    #[prost(bool, optional, tag="2")]
    pub borderless: ::core::option::Option<bool>,
    #[prost(message, optional, tag="3")]
    pub size: ::core::option::Option<SplitSize>,
}
```

`TiledNearTargetPlacement` becomes:

```rust
pub struct TiledNearTargetPlacement {
    #[prost(string, tag="1")]
    pub target_pane: ::prost::alloc::string::String,
    #[prost(enumeration="Direction", tag="2")]
    pub direction: i32,
    #[prost(bool, optional, tag="3")]
    pub borderless: ::core::option::Option<bool>,
    #[prost(message, optional, tag="4")]
    pub size: ::core::option::Option<SplitSize>,
}
```

(If the project regenerates prost from `.proto` via a build step rather than committing the generated file, run that instead — but `cd923bbb` hand-edited this committed file, so match that workflow.)

- [ ] **Step 5: Add the reverse proto→layout `SplitSize` conversion helper**

In `zellij-utils/src/ipc/protobuf_conversion.rs`, near the existing `From<layout::SplitSize> for proto::SplitSize` (line ~3749), add:

```rust
// Reverse SplitSize conversion (proto message -> layout)
fn proto_split_size_to_layout(
    size: crate::client_server_contract::client_server_contract::SplitSize,
) -> Result<crate::input::layout::SplitSize> {
    use crate::client_server_contract::client_server_contract::split_size::SizeType;
    match size
        .size_type
        .ok_or_else(|| anyhow!("SplitSize missing size_type"))?
    {
        SizeType::Percent(p) => Ok(crate::input::layout::SplitSize::Percent(p as usize)),
        SizeType::Fixed(f) => Ok(crate::input::layout::SplitSize::Fixed(f as usize)),
    }
}
```

- [ ] **Step 6: Carry `size` in the forward `From<NewPanePlacement>` conversion**

In `zellij-utils/src/ipc/protobuf_conversion.rs`, replace the two `Tiled` arms (lines ~3504-3514, the `borderless: Some(b)` and `borderless: None` arms produced in Task 1 with `size: _`) with a single arm that routes through `TiledWithOptions` whenever borderless **or** size is present:

```rust
            crate::data::NewPanePlacement::Tiled {
                direction,
                borderless,
                size,
            } => {
                if borderless.is_some() || size.is_some() {
                    PlacementType::TiledWithOptions(TiledPlacement {
                        direction: direction.map(direction_to_proto_i32),
                        borderless,
                        size: size.map(Into::into),
                    })
                } else {
                    // legacy wire form (no borderless, no size)
                    PlacementType::Tiled(direction.map(direction_to_proto_i32).unwrap_or(0))
                }
            },
```

And the `TiledNearTarget` arm (lines ~3515-3523) — add `size`:

```rust
            crate::data::NewPanePlacement::TiledNearTarget {
                target_pane,
                direction,
                borderless,
                size,
            } => PlacementType::TiledNearTarget(TiledNearTargetPlacement {
                target_pane,
                direction: direction_to_proto_i32(direction),
                borderless,
                size: size.map(Into::into),
            }),
```

(`size.map(Into::into)` uses the existing `From<layout::SplitSize> for proto::SplitSize`.)

- [ ] **Step 7: Carry `size` in the reverse `TryFrom<proto::NewPanePlacement>`**

In the same file, the reverse conversion (lines ~3579-3617):

`TiledWithOptions(opts)` arm — read size:

```rust
            PlacementType::TiledWithOptions(opts) => {
                let direction = opts.direction.map(proto_i32_to_direction).transpose()?;
                Ok(crate::data::NewPanePlacement::Tiled {
                    direction,
                    borderless: opts.borderless,
                    size: opts.size.map(proto_split_size_to_layout).transpose()?,
                })
            },
```

`TiledNearTarget(opts)` arm — read size:

```rust
            PlacementType::TiledNearTarget(opts) => {
                Ok(crate::data::NewPanePlacement::TiledNearTarget {
                    target_pane: opts.target_pane,
                    direction: proto_i32_to_direction(opts.direction)?,
                    borderless: opts.borderless,
                    size: opts.size.map(proto_split_size_to_layout).transpose()?,
                })
            },
```

legacy `Tiled(direction)` arm (line ~3607) — `size: None` (it was added as `size: None` in Task 1; leave it).

- [ ] **Step 8: Run the roundtrip tests**

Run: `cargo test -p zellij-utils --lib roundtrip 2>&1 | tail -20`
Expected: PASS — both new cases roundtrip `size` intact, and the existing `new_tiled_pane_rejects_non_tiled_placement` and other cases stay green.

- [ ] **Step 9: Commit**

```bash
git add zellij-utils/src/client_server_contract/common_types.proto \
        zellij-utils/assets/prost_ipc/client_server_contract.rs \
        zellij-utils/src/ipc/protobuf_conversion.rs \
        zellij-utils/src/ipc/tests/roundtrip_tests.rs
git commit -m "feat(ipc): carry tiled pane size over the wire

Adds size to TiledPlacement/TiledNearTargetPlacement protobuf messages
and routes plain Tiled through TiledWithOptions when a size is present so
it isn't dropped by the legacy wire form. Roundtrip-tested.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: CLI parsing + axis validation (TDD)

Relax `--width`/`--height` so they apply to tiled spawns, and convert them to `SplitSize` along the split axis inside `actions.rs`.

**Files:**
- Modify: `zellij-utils/src/cli.rs` (relax `requires("floating")` on new-pane width/height only)
- Modify: `zellij-utils/src/input/actions.rs` (`tiled_placement_from_cli` + new `tiled_size_from_cli` + 3 call sites + imports)
- Test: `zellij-utils/src/input/actions.rs` (existing `#[cfg(test)] mod tests`)

- [ ] **Step 1: Write the failing tests**

In `zellij-utils/src/input/actions.rs`, in the `mod tests` block (near the existing `test_new_pane_tiled_with_target_pane_*` tests around line 3625), add. Note: `CliAction::NewPane` already has `width`/`height` fields, so these fixtures just set them:

```rust
    #[test]
    fn test_new_pane_tiled_near_target_with_fixed_height() {
        let cli_action = new_pane_cli_fixture(NewPaneFixture {
            direction: Some(Direction::Down),
            target_pane: Some("editor".to_string()),
            height: Some("2".to_string()),
            ..Default::default()
        });
        let actions =
            Action::actions_from_cli(cli_action, Box::new(|| PathBuf::from("/tmp")), None).unwrap();
        match &actions[0] {
            Action::NewTiledPane { placement, .. } => assert_eq!(
                placement,
                &NewPanePlacement::TiledNearTarget {
                    target_pane: "editor".to_string(),
                    direction: Direction::Down,
                    borderless: None,
                    size: Some(SplitSize::Fixed(2)),
                }
            ),
            _ => panic!("expected NewTiledPane"),
        }
    }

    #[test]
    fn test_new_pane_tiled_with_percent_height() {
        let cli_action = new_pane_cli_fixture(NewPaneFixture {
            direction: Some(Direction::Down),
            height: Some("20%".to_string()),
            ..Default::default()
        });
        let actions =
            Action::actions_from_cli(cli_action, Box::new(|| PathBuf::from("/tmp")), None).unwrap();
        match &actions[0] {
            Action::NewTiledPane { placement, .. } => assert_eq!(
                placement,
                &NewPanePlacement::Tiled {
                    direction: Some(Direction::Down),
                    borderless: None,
                    size: Some(SplitSize::Percent(20)),
                }
            ),
            _ => panic!("expected NewTiledPane"),
        }
    }

    #[test]
    fn test_new_pane_width_on_vertical_split_is_rejected() {
        // down/up split uses --height; --width must error.
        let cli_action = new_pane_cli_fixture(NewPaneFixture {
            direction: Some(Direction::Down),
            width: Some("2".to_string()),
            ..Default::default()
        });
        let result = Action::actions_from_cli(cli_action, Box::new(|| PathBuf::from("/tmp")), None);
        assert!(result.is_err());
        assert!(result.unwrap_err().contains("--width"));
    }

    #[test]
    fn test_new_pane_size_without_direction_is_rejected() {
        let cli_action = new_pane_cli_fixture(NewPaneFixture {
            height: Some("2".to_string()),
            ..Default::default()
        });
        let result = Action::actions_from_cli(cli_action, Box::new(|| PathBuf::from("/tmp")), None);
        assert!(result.is_err());
        assert!(result.unwrap_err().contains("--direction"));
    }
```

To keep these readable without repeating the ~25-field `CliAction::NewPane` literal, add this **test-only** fixture builder at the top of the `mod tests` block (model it on the fully-specified literal already present in `test_new_pane_tiled_with_target_pane_id` at line ~3625 — copy that field list verbatim so every field matches the current `CliAction::NewPane` definition):

```rust
    #[derive(Default)]
    struct NewPaneFixture {
        direction: Option<Direction>,
        target_pane: Option<String>,
        width: Option<String>,
        height: Option<String>,
    }

    fn new_pane_cli_fixture(f: NewPaneFixture) -> CliAction {
        CliAction::NewPane {
            direction: f.direction,
            target_pane: f.target_pane,
            command: vec![],
            plugin: None,
            cwd: None,
            floating: false,
            in_place: false,
            close_replaced_pane: false,
            name: None,
            close_on_exit: false,
            start_suspended: false,
            configuration: None,
            skip_plugin_cache: false,
            x: None,
            y: None,
            width: f.width,
            height: f.height,
            pinned: None,
            stacked: false,
            blocking: false,
            block_until_exit_success: false,
            block_until_exit_failure: false,
            block_until_exit: false,
            unblock_condition: None,
            near_current_pane: false,
            borderless: None,
            tab_id: None,
        }
    }
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cargo test -p zellij-utils --lib test_new_pane_ 2>&1 | tail -25`
Expected: the four new tests FAIL — `tiled_placement_from_cli` doesn't accept width/height yet (compile error or wrong placement), and the wrong-axis/no-direction inputs aren't rejected.

- [ ] **Step 3: Add imports to actions.rs**

In `zellij-utils/src/input/actions.rs`, ensure these are imported (add any missing):

```rust
use crate::input::layout::{PercentOrFixed, SplitSize};
use std::str::FromStr;
```

(`actions.rs` already imports many `input::layout` and `data` items; add `PercentOrFixed`/`SplitSize` to the appropriate existing `use` group rather than duplicating, and add `FromStr` if not present.)

- [ ] **Step 4: Replace `tiled_placement_from_cli` and add `tiled_size_from_cli`**

In `zellij-utils/src/input/actions.rs`, replace the whole `tiled_placement_from_cli` function (lines ~2319-2339) with:

```rust
fn tiled_placement_from_cli(
    target_pane: Option<String>,
    direction: Option<Direction>,
    borderless: Option<bool>,
    width: Option<String>,
    height: Option<String>,
) -> Result<NewPanePlacement, String> {
    let size = tiled_size_from_cli(direction, width, height)?;
    match target_pane {
        Some(target_pane) => {
            let direction =
                direction.ok_or_else(|| "--target-pane requires --direction".to_string())?;
            Ok(NewPanePlacement::TiledNearTarget {
                target_pane,
                direction,
                borderless,
                size,
            })
        },
        None => Ok(NewPanePlacement::Tiled {
            direction,
            borderless,
            size,
        }),
    }
}

/// Resolve `--width`/`--height` into a single `SplitSize` along the tiled
/// split axis:
///   - down/up   → `--height` sets rows; `--width` is an error
///   - left/right → `--width` sets cols; `--height` is an error
/// A size requires a `--direction` (the axis); without one it's an error.
/// No width/height → `Ok(None)` (default 50% split, unchanged behavior).
fn tiled_size_from_cli(
    direction: Option<Direction>,
    width: Option<String>,
    height: Option<String>,
) -> Result<Option<SplitSize>, String> {
    if width.is_none() && height.is_none() {
        return Ok(None);
    }
    let direction = direction.ok_or_else(|| {
        "--width/--height on a tiled pane require --direction (or pass --floating)".to_string()
    })?;
    let parse = |raw: String, flag: &str| -> Result<SplitSize, String> {
        PercentOrFixed::from_str(&raw)
            .map(SplitSize::from)
            .map_err(|e| format!("invalid {flag} value '{raw}': {e}"))
    };
    match direction {
        Direction::Up | Direction::Down => {
            if width.is_some() {
                return Err(
                    "--width does not apply to a down/up tiled split; use --height".to_string(),
                );
            }
            // height is Some here (one of width/height was Some, width is None)
            Ok(Some(parse(height.unwrap(), "--height")?))
        },
        Direction::Left | Direction::Right => {
            if height.is_some() {
                return Err(
                    "--height does not apply to a left/right tiled split; use --width".to_string(),
                );
            }
            Ok(Some(parse(width.unwrap(), "--width")?))
        },
    }
}
```

- [ ] **Step 5: Pass width/height at the three `tiled_placement_from_cli` call sites**

In `zellij-utils/src/input/actions.rs`, the three calls (blocking arm ~1136, command arm ~1246, no-command arm ~1284). Each currently reads:

```rust
                        tiled_placement_from_cli(target_pane, direction, borderless)?
```

(or `let placement = tiled_placement_from_cli(target_pane, direction, borderless)?;`). Change each to pass `width, height` (moving them — safe because within each `if floating { … } else { … }` chain the tiled branch is mutually exclusive with the floating branch that also moves them):

```rust
                        tiled_placement_from_cli(target_pane, direction, borderless, width, height)?
```

> Note: the `else if let Some(plugin) = plugin` arm (≈1147) builds `NewTiledPluginPane` and intentionally does **not** call `tiled_placement_from_cli` — leave it unchanged. That is the plugin path, out of scope (Task 7 documents it).

- [ ] **Step 6: Relax the clap constraint on new-pane width/height**

In `zellij-utils/src/cli.rs`, in the `NewPane` CliAction variant only, the width/height args (lines ~460 and ~463):

```rust
        /// The width if the pane is floating as a bare integer (eg. 1) or percent (eg. 10%)
        #[clap(long, requires("floating"))]
        width: Option<String>,
        /// The height if the pane is floating as a bare integer (eg. 1) or percent (eg. 10%)
        #[clap(long, requires("floating"))]
        height: Option<String>,
```

Change to (drop `requires("floating")`, update doc):

```rust
        /// Pane width: floating panes, or tiled panes with --direction
        /// left/right (eg. bare integer `5` or percent `10%`)
        #[clap(long)]
        width: Option<String>,
        /// Pane height: floating panes, or tiled panes with --direction
        /// down/up (eg. bare integer `2` or percent `10%`)
        #[clap(long)]
        height: Option<String>,
```

> Only touch the `NewPane` variant's width/height. Leave the identical-looking `requires("floating")` lines on other CliAction variants (Edit/Run/NewTab/etc., e.g. lines ~580, ~646, ~986, ~1112) **unchanged** — they are out of scope.

- [ ] **Step 7: Run the actions tests**

Run: `cargo test -p zellij-utils --lib test_new_pane_ 2>&1 | tail -25`
Expected: PASS — including the existing `test_new_pane_tiled_with_target_pane_id` / `_name` (they now also assert `size: None`, which holds).

- [ ] **Step 8: Confirm the binary accepts the flags (no behavior yet)**

Run: `cargo build -p zellij-utils 2>&1 | tail -3 && cargo run -q -- action new-pane --help 2>&1 | grep -E "width|height"`
Expected: `--width`/`--height` listed without a "requires floating" note.

- [ ] **Step 9: Commit**

```bash
git add zellij-utils/src/cli.rs zellij-utils/src/input/actions.rs
git commit -m "feat(cli): accept --width/--height for tiled new-pane

Relaxes requires(floating) on new-pane width/height and converts them to
a SplitSize along the tiled split axis (down/up -> height, left/right ->
width), erroring on the wrong axis or a missing --direction.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: Apply the size on the targeted (`TiledNearTarget`) split (TDD)

Thread `size` from `tab::new_pane` → `new_tiled_pane_near_target` → `insert_pane_near_pane_id` → `pane_group_split_near_pane_id`, and apply it via `sized_split` (Task 2). This is the consumer's primary path.

**Files:**
- Modify: `zellij-server/src/panes/tiled_panes/mod.rs` (`pane_group_split_near_pane_id`, `insert_pane_near_pane_id`, `can_insert_pane_near_pane_id`, `pane_ids_in_insert_group_near_pane_id`, `move_pane_to_pane_id` call)
- Modify: `zellij-server/src/tab/mod.rs` (`new_tiled_pane_near_target` signature + `new_pane` match arm)
- Modify: `zellij-server/src/screen.rs` (no logic change; `TiledNearTarget { .. }` already uses `..`)
- Test: `zellij-server/src/tab/unit/tab_tests.rs`

- [ ] **Step 1: Write the failing tests**

In `zellij-server/src/tab/unit/tab_tests.rs`, near the existing targeted-spawn tests (they already define the `pane_geom(&tab, id)` helper returning `(x, y, cols, rows)`), add:

```rust
#[test]
fn new_pane_down_of_target_with_fixed_height_is_exact() {
    let size = Size { cols: 120, rows: 20 };
    let mut tab = create_new_tab(size, true);

    tab.new_pane(
        PaneId::Terminal(2),
        None, None, false, true,
        NewPanePlacement::TiledNearTarget {
            target_pane: "terminal_1".to_string(),
            direction: Direction::Down,
            borderless: None,
            size: Some(SplitSize::Fixed(2)),
        },
        Some(1), None,
    ).unwrap();

    // terminal_1 keeps the remainder; terminal_2 is exactly 2 rows tall.
    assert_eq!(pane_geom(&tab, PaneId::Terminal(1)), (0, 0, 120, 18));
    assert_eq!(pane_geom(&tab, PaneId::Terminal(2)), (0, 18, 120, 2));
}

#[test]
fn new_pane_down_of_target_with_percent_height() {
    let size = Size { cols: 120, rows: 20 };
    let mut tab = create_new_tab(size, true);

    tab.new_pane(
        PaneId::Terminal(2),
        None, None, false, true,
        NewPanePlacement::TiledNearTarget {
            target_pane: "terminal_1".to_string(),
            direction: Direction::Down,
            borderless: None,
            size: Some(SplitSize::Percent(20)),
        },
        Some(1), None,
    ).unwrap();

    // 20% of 20 rows = 4.
    assert_eq!(pane_geom(&tab, PaneId::Terminal(1)), (0, 0, 120, 16));
    assert_eq!(pane_geom(&tab, PaneId::Terminal(2)), (0, 16, 120, 4));
}

#[test]
fn new_pane_down_of_target_with_oversize_height_clamps() {
    let size = Size { cols: 120, rows: 20 };
    let mut tab = create_new_tab(size, true);

    tab.new_pane(
        PaneId::Terminal(2),
        None, None, false, true,
        NewPanePlacement::TiledNearTarget {
            target_pane: "terminal_1".to_string(),
            direction: Direction::Down,
            borderless: None,
            size: Some(SplitSize::Fixed(100)), // > parent height
        },
        Some(1), None,
    ).unwrap();

    // Clamped so terminal_1 keeps at least 1 row.
    assert_eq!(pane_geom(&tab, PaneId::Terminal(2)).3, 19);
    assert_eq!(pane_geom(&tab, PaneId::Terminal(1)).3, 1);
}

#[test]
fn new_pane_right_of_target_with_fixed_width_is_exact() {
    let size = Size { cols: 120, rows: 20 };
    let mut tab = create_new_tab(size, true);

    tab.new_pane(
        PaneId::Terminal(2),
        None, None, false, true,
        NewPanePlacement::TiledNearTarget {
            target_pane: "terminal_1".to_string(),
            direction: Direction::Right,
            borderless: None,
            size: Some(SplitSize::Fixed(30)),
        },
        Some(1), None,
    ).unwrap();

    assert_eq!(pane_geom(&tab, PaneId::Terminal(1)), (0, 0, 90, 20));
    assert_eq!(pane_geom(&tab, PaneId::Terminal(2)), (90, 0, 30, 20));
}
```

Confirm `SplitSize` is imported at the top of `tab_tests.rs` (it already imports `use zellij_utils::input::layout::{SplitDirection, SplitSize, TiledPaneLayout};` — added by `cd923bbb`). If absent, add it.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cargo test -p zellij-server --lib new_pane_down_of_target_with 2>&1 | tail -25`
Expected: FAIL — `size` is ignored; the new pane lands at the default 50% (10 rows), not the requested size.

- [ ] **Step 3: Thread `size` into `pane_group_split_near_pane_id`**

In `zellij-server/src/panes/tiled_panes/mod.rs`, change the signature (line ~778) to take `size`:

```rust
    fn pane_group_split_near_pane_id(
        &self,
        target_pane_id: PaneId,
        direction: Direction,
        size: Option<SplitSize>,
    ) -> Option<(Vec<(PaneId, PaneGeom)>, PaneGeom, PaneGeom, SplitDirection)> {
```

Then replace the 50/50 split block (lines ~920-938, the `match split_direction { … }` that sets `first_*`/`second_*` to halves) with a size-aware version. The new pane is `second` for `Right`/`Down` and `first` for `Left`/`Up`:

```rust
        let new_is_second = matches!(direction, Direction::Right | Direction::Down);
        match split_direction {
            SplitDirection::Vertical => {
                let total = group_geom.cols.as_usize();
                let (first_cols, second_cols, new_dim) = match size {
                    Some(s) => {
                        let (new_cells, other_cells, dim) = sized_split(s, total, new_is_second);
                        if new_is_second {
                            (other_cells, new_cells, Some(dim))
                        } else {
                            (new_cells, other_cells, Some(dim))
                        }
                    },
                    None => {
                        let first = total / 2;
                        (first, total.saturating_sub(first), None)
                    },
                };
                first_geom.cols.set_inner(first_cols);
                second_geom.x = first_geom.x + first_cols;
                second_geom.cols.set_inner(second_cols);
                if let Some(dim) = new_dim {
                    if new_is_second {
                        second_geom.cols = dim;
                    } else {
                        first_geom.cols = dim;
                    }
                }
            },
            SplitDirection::Horizontal => {
                let total = group_geom.rows.as_usize();
                let (first_rows, second_rows, new_dim) = match size {
                    Some(s) => {
                        let (new_cells, other_cells, dim) = sized_split(s, total, new_is_second);
                        if new_is_second {
                            (other_cells, new_cells, Some(dim))
                        } else {
                            (new_cells, other_cells, Some(dim))
                        }
                    },
                    None => {
                        let first = total / 2;
                        (first, total.saturating_sub(first), None)
                    },
                };
                first_geom.rows.set_inner(first_rows);
                second_geom.y = first_geom.y + first_rows;
                second_geom.rows.set_inner(second_rows);
                if let Some(dim) = new_dim {
                    if new_is_second {
                        second_geom.rows = dim;
                    } else {
                        first_geom.rows = dim;
                    }
                }
            },
        }
```

(The `None` branch reproduces the prior exact 50/50 behavior, keeping all existing `cd923bbb` tests green.)

- [ ] **Step 4: Update the helper's callers**

In the same file:

`can_insert_pane_near_pane_id` (line ~952) and `pane_ids_in_insert_group_near_pane_id` (line ~960) are **validation** helpers — they just check feasibility, so pass `None`:

```rust
    pub fn can_insert_pane_near_pane_id(
        &self,
        target_pane_id: PaneId,
        direction: Direction,
    ) -> bool {
        self.pane_group_split_near_pane_id(target_pane_id, direction, None)
            .is_some()
    }
    pub fn pane_ids_in_insert_group_near_pane_id(
        &self,
        target_pane_id: PaneId,
        direction: Direction,
    ) -> Option<HashSet<PaneId>> {
        self.pane_group_split_near_pane_id(target_pane_id, direction, None)
            .map(|(pane_ids_and_geoms, _, _, _)| {
                pane_ids_and_geoms
                    .into_iter()
                    .map(|(pane_id, _)| pane_id)
                    .collect()
            })
    }
```

`insert_pane_near_pane_id` (line ~1086) — add a `size` param and pass it down:

```rust
    pub fn insert_pane_near_pane_id(
        &mut self,
        target_pane_id: PaneId,
        pane_id: PaneId,
        mut new_pane: Box<dyn Pane>,
        direction: Direction,
        size: Option<SplitSize>,
    ) -> Result<(), Box<dyn Pane>> {
        let Some((grouped_pane_ids_and_geoms, group_side_geom, new_pane_geom, split_direction)) =
            self.pane_group_split_near_pane_id(target_pane_id, direction, size)
        else {
            return Err(new_pane);
        };
        // … rest unchanged …
```

`move_pane_to_pane_id` (line ~990) calls `insert_pane_near_pane_id` (around line ~1071) for the pane-move feature, which has no size — pass `None`:

```rust
            self.insert_pane_near_pane_id(target, source, source_pane, direction, None)
```

- [ ] **Step 5: Thread `size` through `tab::new_tiled_pane_near_target`**

In `zellij-server/src/tab/mod.rs`, add a `size` parameter to `new_tiled_pane_near_target` (signature at line ~1811). Add it after `borderless`:

```rust
    pub fn new_tiled_pane_near_target(
        &mut self,
        pid: PaneId,
        initial_pane_title: Option<String>,
        invoked_with: Option<Run>,
        start_suppressed: bool,
        should_focus_pane: bool,
        target_pane: String,
        direction: Direction,
        client_id: Option<ClientId>,
        blocking_notification: Option<NotificationEnd>,
        borderless: Option<bool>,
        size: Option<SplitSize>,
    ) -> Result<()> {
```

At the `insert_pane_near_pane_id` call (line ~1974) pass `size`:

```rust
        match self
            .tiled_panes
            .insert_pane_near_pane_id(target_pane_id, pid, new_pane, direction, size)
        {
```

Ensure `SplitSize` is imported in `tab/mod.rs`. The file imports `use zellij_utils::input::layout::{… SplitSize …}` already (used by `tab_tests`)? Confirm; if not, add `SplitSize` to the existing `input::layout::{…}` import in `tab/mod.rs`.

- [ ] **Step 6: Bind `size` in the `new_pane` match arm**

In `zellij-server/src/tab/mod.rs`, the `NewPanePlacement::TiledNearTarget` arm of `new_pane` (line ~1543, currently `{ target_pane, direction, borderless, size: _ }` from Task 1). Bind `size` and pass it:

```rust
            NewPanePlacement::TiledNearTarget {
                target_pane,
                direction,
                borderless,
                size,
            } => self.new_tiled_pane_near_target(
                pid,
                initial_pane_title,
                invoked_with,
                start_suppressed,
                should_focus_pane,
                target_pane,
                direction,
                client_id,
                blocking_notification,
                borderless,
                size,
            ),
```

- [ ] **Step 7: Run the targeted tests**

Run: `cargo test -p zellij-server --lib new_pane_down_of_target_with new_pane_right_of_target_with 2>&1 | tail -25`
Expected: PASS (4 new tests).

- [ ] **Step 8: Re-run the full `cd923bbb` targeted-spawn suite to confirm no regressions**

Run: `cargo test -p zellij-server --lib new_pane_ 2>&1 | tail -25`
Expected: PASS — all the existing `new_pane_*_of_*` tests (which spawn with `size: None`) still produce 50/50 geometry.

- [ ] **Step 9: Commit**

```bash
git add zellij-server/src/panes/tiled_panes/mod.rs zellij-server/src/tab/mod.rs
git commit -m "feat(tiled): honor fixed/percent size on targeted new-pane

Threads size through new_tiled_pane_near_target -> insert_pane_near_pane_id
-> pane_group_split_near_pane_id and applies it via sized_split, so a
--target-pane spawn lands at an exact size (clamped) instead of 50%.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: Apply the size on the plain directional (`Tiled`) split (TDD)

Make `new-pane --direction down --height 2` (no `--target-pane`) honor the size too — this is the spec's smoke-test path. Plain directional splits go through `horizontal_split`/`vertical_split` → `split_pane_horizontally`/`split_pane_vertically` (terminal panes only).

**Files:**
- Modify: `zellij-server/src/tab/mod.rs` (`new_pane` Tiled arm; `horizontal_split`/`vertical_split` signatures)
- Modify: `zellij-server/src/panes/tiled_panes/mod.rs` (`split_pane_horizontally`/`split_pane_vertically`)
- Test: `zellij-server/src/tab/unit/tab_tests.rs`

- [ ] **Step 1: Write the failing test**

In `zellij-server/src/tab/unit/tab_tests.rs`, add:

```rust
#[test]
fn new_pane_directional_down_with_fixed_height_is_exact() {
    let size = Size { cols: 120, rows: 20 };
    let mut tab = create_new_tab(size, true);

    tab.new_pane(
        PaneId::Terminal(2),
        None, None, false, true,
        NewPanePlacement::Tiled {
            direction: Some(Direction::Down),
            borderless: None,
            size: Some(SplitSize::Fixed(2)),
        },
        Some(1), None,
    ).unwrap();

    // Active (terminal_1) keeps the top; the new pane is the bottom 2 rows.
    assert_eq!(pane_geom(&tab, PaneId::Terminal(1)), (0, 0, 120, 18));
    assert_eq!(pane_geom(&tab, PaneId::Terminal(2)), (0, 18, 120, 2));
}

#[test]
fn new_pane_directional_right_with_fixed_width_is_exact() {
    let size = Size { cols: 120, rows: 20 };
    let mut tab = create_new_tab(size, true);

    tab.new_pane(
        PaneId::Terminal(2),
        None, None, false, true,
        NewPanePlacement::Tiled {
            direction: Some(Direction::Right),
            borderless: None,
            size: Some(SplitSize::Fixed(30)),
        },
        Some(1), None,
    ).unwrap();

    assert_eq!(pane_geom(&tab, PaneId::Terminal(1)), (0, 0, 90, 20));
    assert_eq!(pane_geom(&tab, PaneId::Terminal(2)), (90, 0, 30, 20));
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cargo test -p zellij-server --lib new_pane_directional_ 2>&1 | tail -20`
Expected: FAIL — the new pane lands at 50% (10 rows / 60 cols), not the requested size.

- [ ] **Step 3: Add `size` to `split_pane_horizontally` / `split_pane_vertically`**

In `zellij-server/src/panes/tiled_panes/mod.rs`:

`split_pane_horizontally` (line ~664) — add `size` and apply it to the new (bottom) pane. Insert the size handling right after the `if let Some((top_winsize, bottom_winsize)) = split(...)` is destructured, making the winsizes mutable:

```rust
    pub fn split_pane_horizontally(
        &mut self,
        pid: PaneId,
        mut new_pane: Box<dyn Pane>,
        client_id: ClientId,
        size: Option<SplitSize>,
    ) {
        // … existing full_pane_size resolution unchanged …
        let active_pane = self.panes.get_mut(active_pane_id).unwrap();
        if let Some((mut top_winsize, mut bottom_winsize)) =
            split(SplitDirection::Horizontal, &full_pane_size)
        {
            if let Some(s) = size {
                let total = full_pane_size.rows.as_usize();
                let (new_cells, other_cells, dim) = sized_split(s, total, /*new_is_lead*/ false);
                // active pane keeps the top; new pane is the bottom.
                top_winsize.rows.set_inner(other_cells);
                bottom_winsize.y = top_winsize.y + other_cells;
                bottom_winsize.rows = dim;
                let _ = new_cells; // bottom rows == new_cells, encoded in dim
            }
            if active_pane.position_and_size().is_stacked() {
                // … existing stacked branch, now using top_winsize …
            } else {
                active_pane.set_geom(top_winsize);
            }
            new_pane.set_geom(bottom_winsize);
            self.panes.insert(pid, new_pane);
            self.relayout(SplitDirection::Vertical);
        }
    }
```

`split_pane_vertically` (line ~709) — symmetric, new (right) pane gets the size:

```rust
    pub fn split_pane_vertically(
        &mut self,
        pid: PaneId,
        mut new_pane: Box<dyn Pane>,
        client_id: ClientId,
        size: Option<SplitSize>,
    ) {
        // … existing full_pane_size resolution unchanged …
        let active_pane = self.panes.get_mut(active_pane_id).unwrap();
        if let Some((mut left_winsize, mut right_winsize)) =
            split(SplitDirection::Vertical, &full_pane_size)
        {
            if let Some(s) = size {
                let total = full_pane_size.cols.as_usize();
                let (_new_cells, other_cells, dim) = sized_split(s, total, false);
                left_winsize.cols.set_inner(other_cells);
                right_winsize.x = left_winsize.x + other_cells;
                right_winsize.cols = dim;
            }
            if active_pane.position_and_size().is_stacked() {
                // … existing stacked branch, now using left_winsize …
            } else {
                active_pane.set_geom(left_winsize);
            }
            new_pane.set_geom(right_winsize);
            self.panes.insert(pid, new_pane);
            self.relayout(SplitDirection::Horizontal);
        }
    }
```

> Keep the existing stacked-pane sub-branches intact; just rename the destructured bindings to `mut` and let the `if let Some(s) = size` block adjust the winsizes before they're used. When `size` is `None`, the winsizes are the unmodified 50/50 split — existing behavior preserved.

- [ ] **Step 4: Add `size` to `horizontal_split` / `vertical_split` in tab/mod.rs**

In `zellij-server/src/tab/mod.rs`, add a `size: Option<SplitSize>` parameter to `horizontal_split` (line ~2562) and `vertical_split` (line ~2629), and forward it to the `split_pane_*` call:

`horizontal_split` — change the signature to add `size: Option<SplitSize>,` (after `borderless`), and the call (line ~2607):

```rust
                self.tiled_panes
                    .split_pane_horizontally(pid, Box::new(new_terminal), client_id, size);
```

`vertical_split` — same: add `size` param, and the call (line ~2674):

```rust
                self.tiled_panes
                    .split_pane_vertically(pid, Box::new(new_terminal), client_id, size);
```

- [ ] **Step 5: Bind and pass `size` in the `new_pane` Tiled arm**

In `zellij-server/src/tab/mod.rs`, the `NewPanePlacement::Tiled { direction: Some(direction), borderless, size: _ }` arm (line ~1518, from Task 1). Bind `size` and pass to both split helpers:

```rust
            NewPanePlacement::Tiled {
                direction: Some(direction),
                borderless,
                size,
            } => {
                if let Some(client_id) = client_id {
                    if direction == Direction::Left || direction == Direction::Right {
                        self.vertical_split(
                            pid,
                            initial_pane_title,
                            client_id,
                            blocking_notification,
                            borderless,
                            size,
                        )?;
                    } else {
                        self.horizontal_split(
                            pid,
                            initial_pane_title,
                            client_id,
                            blocking_notification,
                            borderless,
                            size,
                        )?;
                    }
                }
                Ok(())
            },
```

The `NewPanePlacement::Tiled { direction: None, borderless, size: _ }` arm (line ~1505) calls `new_tiled_pane` (no axis) — leave `size: _` there; a size without `--direction` is already rejected at the CLI (Task 4), so this arm never carries a meaningful size.

- [ ] **Step 6: Find any other callers of `horizontal_split`/`vertical_split`/`split_pane_*` and pass `None`**

Run: `grep -rn "\.horizontal_split(\|\.vertical_split(\|split_pane_horizontally(\|split_pane_vertically(" zellij-server/src`
For every call **other than** the ones changed above (e.g. keybind-driven splits, tests), add a trailing `None` argument for `size`. Update each so the workspace compiles.

- [ ] **Step 7: Run the directional tests + full split suite**

Run: `cargo test -p zellij-server --lib new_pane_directional_ split 2>&1 | tail -25`
Expected: PASS — new directional tests pass; existing `split`/`horizontal_split`/`vertical_split` tests stay green (size `None` ⇒ unchanged 50/50).

- [ ] **Step 8: Full workspace test**

Run: `cargo test --workspace 2>&1 | tail -20`
Expected: all green.

- [ ] **Step 9: Commit**

```bash
git add zellij-server/src/tab/mod.rs zellij-server/src/panes/tiled_panes/mod.rs
git commit -m "feat(tiled): honor size on plain directional new-pane split

--direction down --height N (no --target-pane) now lands the new pane at
an exact size via split_pane_{horizontally,vertically}, reusing
sized_split. Default (no size) is unchanged 50/50.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: Build, live smoke test, docs, and release note

**Files:**
- Modify: `README.md` (fork-notice patch list — see `51626f64`)
- Modify: any fork CHANGELOG if present

- [ ] **Step 1: Release build**

Run: `cargo build --release 2>&1 | tail -5`
Expected: builds `target/release/zellij-gabi` with no errors.

- [ ] **Step 2: Live smoke — fixed rows**

In one shell: `target/release/zellij-gabi -s sizetest`
In a second shell attached to that session:

```sh
target/release/zellij-gabi -s sizetest action new-pane --name strip --direction down --height 2 -- bash
```

Expected: a new tiled pane appears below, **exactly 2 rows tall** (plus its frame). Confirm visually.

- [ ] **Step 3: Live smoke — percent + targeted + default**

```sh
# percent
target/release/zellij-gabi -s sizetest action new-pane --direction down --height 20% -- bash
# targeted, exact rows, borderless (mirrors the consumer minus --plugin)
target/release/zellij-gabi -s sizetest action new-pane --name strip2 --target-pane strip \
  --direction down --height 2 --borderless true -- bash
# default unchanged (50%)
target/release/zellij-gabi -s sizetest action new-pane --direction down -- bash
```

Expected: 20% pane ≈ proportional; targeted 2-row strip lands below `strip`; default splits 50/50. The targeted/command spawn prints the created pane id (e.g. `terminal_7`) to stdout.

- [ ] **Step 4: Live smoke — validation errors**

```sh
target/release/zellij-gabi -s sizetest action new-pane --height 2 -- bash        # no --direction
target/release/zellij-gabi -s sizetest action new-pane --direction down --width 2 -- bash  # wrong axis
```

Expected: each prints a clear error (`require --direction`; `--width does not apply to a down/up tiled split`) and creates no pane. Then `target/release/zellij-gabi -s sizetest kill-session sizetest` to clean up.

- [ ] **Step 5: Update the README fork notice**

In `README.md`, locate the fork-notice patch list (added by `51626f64`) and add a bullet, matching the existing style, e.g.:

```markdown
- **Fixed-size tiled `new-pane`** — `--width`/`--height` now also apply to
  tiled (non-floating) spawns. With `--direction down|up` use `--height`; with
  `--direction left|right` use `--width`. Accepts a bare integer (`2`) or
  percent (`20%`), as `Fixed`/`Percent`, clamped to fit. Composes with
  `--target-pane`, `--borderless`, and `--configuration`. (Terminal/command
  panes only; plugin tiled panes are not yet covered.)
```

- [ ] **Step 6: Update CHANGELOG if the fork keeps one**

Run: `ls CHANGELOG.md 2>/dev/null`
If present, add an entry under the unreleased/next-version heading describing the new `--width`/`--height` tiled behavior in the fork's existing style. If absent, skip.

- [ ] **Step 7: Commit**

```bash
git add README.md CHANGELOG.md 2>/dev/null; git add README.md
git commit -m "docs: note fixed-size tiled new-pane in fork patch list

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 8: Record the consumer/release follow-ups (do not action here)**

Note for the human (not a code change):
1. The tap formula `thetaurean/tap/zellij-gabi` needs a new release once this lands; gabi consumes the binary by name via `gabi_types::ZELLIJ_BIN` (`"zellij-gabi"`).
2. **Known gap:** the consumer's real call uses `--plugin … --target-pane drawer --height 2`. Plugin tiled panes support **neither** `--target-pane` nor a tiled size today (separate `NewTiledPluginPane`/`FillPluginCwd`/`PluginInstruction::Load` stack, untouched here). A follow-up plan must thread a `NewPanePlacement` (target + direction + size) through that plugin stack so the plugin call works end-to-end. Until then, gabi's drawer-tabs strip cannot use this feature; the terminal/command `--height` path is fully functional.

---

## Self-Review

**Spec coverage:**
- Relax `requires("floating")` on `--width`/`--height` → Task 4 Step 6. ✓
- Validate size allowed for floating OR tiled-with-direction → Task 4 (`tiled_size_from_cli`). ✓
- Axis interpretation (down/up→height, left/right→width; wrong axis errors) → Task 4. ✓
- Accept `Fixed` and `Percent` via `PercentOrFixed`/`SplitSize` → Tasks 4 (parse), 2/5/6 (apply). ✓
- No size ⇒ unchanged 50% → `None` branches in Tasks 5/6 preserve exact prior code. ✓
- Clamp like layout `size=N` → Task 2 `sized_split` clamps to `[1, total-1]` (no `MIN_*` floor, matching layout strips); Task 5 oversize-clamp test. ✓
- Mirror `cd923bbb` file-by-file (cli, actions, data, plugin_api, proto, generated, protobuf_conversion, kdl, route, screen, tab, tiled_panes, zellij_exports, tests) → Tasks 1–6 touch each. ✓
- Roundtrip test → Task 3. Tab unit tests (fixed, percent, clamp) → Tasks 5/6. Keep `cd923bbb` tests green → Tasks 5 Step 8, 6 Steps 7-8. ✓
- Build & smoke (`--height 2`, `--height 20%`, default) → Task 7. ✓
- README patch list + CHANGELOG + consumer/release note → Task 7. ✓
- `--height` composes with `--target-pane`/`--borderless`/`--configuration` and prints pane id → Task 7 Steps 3 (terminal/command). Plugin composition explicitly out of scope and flagged → Task 7 Step 8. ✓

**Type consistency:** `size: Option<SplitSize>` is the single name used across the enum (Task 1), CLI conversion (Task 4), IPC (Task 3), and server (Tasks 5/6). The geometry helper `sized_split(SplitSize, usize, bool) -> (usize, usize, Dimension)` is defined once (Task 2) and called identically in Tasks 5/6. Server methods consistently gain a trailing `size: Option<SplitSize>` parameter: `pane_group_split_near_pane_id`, `insert_pane_near_pane_id`, `new_tiled_pane_near_target`, `horizontal_split`, `vertical_split`, `split_pane_horizontally`, `split_pane_vertically`.

**Placeholder scan:** No `TBD`/"add error handling"/"similar to Task N" — every code step shows the code; every error message is spelled out; the mechanical Task 1 sites are enumerated by file:line with the exact `size: None` / `size: _` rule.
