# thetaurean/zellij — fork TODO

Fork-local follow-ups discovered while building the gabi pane-primitives
patch set. Upstream-PR-tracked TODOs (e.g. plugin-API / KDL parity for
`--target-pane`) live in `FORK_NOTES.md` under each patch's "Scope and
known gaps" section; this file is for things that don't belong in those
PR descriptions.

## Latent: `pane_group_split_near_pane_id` Left/Right Fixed-rows override leaves new pane non-resizable on the perpendicular axis

**Where:** `zellij-server/src/panes/tiled_panes/mod.rs:871-878` (the
non-stacked, Left/Right arm of the bounding-box construction).

```rust
Direction::Left | Direction::Right => {
    group_geom.rows = Dimension::fixed(max_y.saturating_sub(min_y));
},
```

**Why it exists:** for `--target-pane <T> --direction Left|Right`, the
group can span multiple visual rows (e.g. editor + drawer column-strip).
The combined height isn't expressible as a single Percent of the
display, so `group_geom.rows` is set to a Fixed cell span — the new pane
(e.g. thread) inherits `rows = Fixed(...)` from `split()` and Cassowary
treats it as a hard span across multiple grid rows. That part is
correct.

**Latent bug:** because `new_pane.set_geom(new_pane_geom)` lands the new
pane with `rows = Fixed`, every subsequent operation that calls
`increase_pane_height` (or `reduce_pane_height`) on it silently
no-ops — both `TerminalPane::increase_height`
(`zellij-server/src/panes/terminal_pane.rs:543`) and
`PluginPane::increase_height` (`zellij-server/src/panes/plugin_pane.rs`)
guard on `geom.rows.as_percent()` and do nothing for Fixed dims. The
result is the same shape of bug the Up/Down branch had (fixed in this
PR): `close-pane --absorb-to <new-pane>` from a vertical sibling would
fail to grow the new pane's height, and PaneResizer's Cassowary solver
would redistribute the freed rows across other Percent panes instead.

**Why we didn't fix it together with the Up/Down case:** the Up/Down fix
was load-bearing for gabi's drawer↔thread flow (`close-pane --absorb-to
editor` from thread). The Left/Right case is symmetric but only bites
if a future flow tries to absorb a vertical sibling INTO the
Left/Right-spawned pane — gabi has no such flow today.

**Intended fix:** mirror the Up/Down fix's structure. Two viable
approaches:

1. **Spawn-site fix (preferred — symmetric with Up/Down).** Instead of
   forcing `group_geom.rows = Dimension::fixed(...)` and letting the new
   pane inherit it, compute a Percent representation of the group's
   combined height (sum of the group members' `rows.as_percent()`
   values, since the same_group filter already constrains them to
   the same x/cols column). Apply `Percent(sum)` to `group_geom.rows`.
   The new pane then inherits `rows = Percent(...)` and `increase_height`
   works. Cassowary still gets the right visual span because the Percent
   total matches what Fixed would have been.

   Risk: if any group member's `rows` isn't Percent (e.g. a Fixed-row
   pane was previously inserted into the column), summation falls back
   to a representation that may differ from what Cassowary expects.
   Add a fallback to the current Fixed path when not all members are
   Percent, and add a regression test that covers the mixed case.

2. **Close-site normalization (narrower scope).** In
   `tiled_pane_grid::fill_space_over_pane_absorbing_to`, before
   calling `grow_panes`, walk `panes_to_grow` and convert any Fixed
   member's relevant axis to Percent using
   `cells / display_area_dimension * 100`. Doesn't touch spawn; doesn't
   help any other resize flow. Useful if (1) turns out to break tests.

**Regression tests should cover:** after `--target-pane editor
--direction Left` spawns thread, `close-pane --pane-id <vertical-sibling>
--absorb-to terminal_<thread>` (or a `--name thread` lookup) must grow
thread's height by the closed pane's height. Mirror
`cross_patch_gabi_layout_thread_first_drawer_second_absorb_to_editor`'s
shape but invert axes — thread is the absorber, not the absorbee.

**Trigger to actually do this:** the moment gabi (or any other zellij
consumer) introduces a "close X absorb-to thread" pattern, or upstream
review surfaces it. Until then, the Up/Down fix's comment block in
`pane_group_split_near_pane_id` references this TODO.

---

# Zellij fork TODO: right-click plugin dispatch

The zellij-gabi fork ships from `thetaurean/tap`. The fork's source lives at `~/dev/zellij`.

`zellij-server/src/panes/plugin_pane.rs:719` defines `handle_right_click` but it is never called by the server. As a result, `Mouse::RightClick` events never reach plugins: `gabi-files`, `gabi-tree`, and any future `gabi-overlay-menu` consumer cannot offer right-click as a `⋮`-equivalent trigger.

When wired, the `gabi-overlay-menu` crate is the natural integration point: it already owns `Trigger` and `Menu`, and a right-click event would route through the same `hit_test` path that left-clicks use today.

This note is duplicated as a TODO in the fork repo so the work is discoverable from either side.
