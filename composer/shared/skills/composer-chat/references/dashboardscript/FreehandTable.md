# Freehand Table (StyleBI Dashboard Script Reference)

Sources:
- https://www.inetsoft.com/docs/stylebi/InetSoftUserDocumentation/1.0.0/dashboardscript/FreehandTable.html
  (`layoutInfo` API index and per-method pages)
- https://www.inetsoft.com/docs/stylebi/InetSoftUserDocumentation/1.0.0/dashboard/AddFreehandTable.html
  ("Add Formula Cells" / "Dynamic Formulas" section — the cross-cell reference syntax)
- https://www.inetsoft.com/docs/stylebi/InetSoftUserDocumentation/1.0.0/dashboardscript/SpecialFreehandFunctions.html
  (`toList`/`rowList`/`mapList`/`toArray`/`inArray` — the value-building functions a formula cell
  calls on `data[...]`/`q[...]`, including `toList`'s `date`/`rounddate` grouping options)

> A Freehand (calc) table is a grid of individually-bound cells (`{row, col}` addressing), unlike a
> Crosstab's uniform aggregation model. Each cell is independently one of: static text, a bound field,
> or a formula script. This is the mechanism to reach for whenever a business requirement mixes
> straight aggregation with cross-row/cross-cell arithmetic (e.g. "order total", "return total", and
> "net sales = order total − return total" in one table) — a Crosstab has no vocabulary for the third
> kind of cell.

---

## The `$<name>` cross-cell reference — the answer to "how does one cell's formula reference another cell's value"

**This is the one thing that is easy to get wrong by guessing, and StyleBI's own official docs confirm
it explicitly**: a formula cell refers to another cell's value with `$cell_name`, where `cell_name` is
whatever name was assigned to that cell (via `layoutInfo.setCellName` in script, or the "Cell Name"
field in the Composer's cell-properties dialog — the composer-chat MCP tool surface exposes this as
`set_cell_binding`'s `name` field).

```js
// Cell (0,1) was named "OrderTotal", cell (1,1) was named "ReturnTotal".
// A formula cell can now reference both by name:
FreehandTable1.layoutInfo.setCellBinding(2, 1, 3, "$OrderTotal - $ReturnTotal");
```

**What does *not* work, despite looking plausible**: spreadsheet-style `B1-B2` addressing, a bareword
column/cell identifier with no `$` prefix, and `field["ColumnName"]` (that syntax is for binding a
cell to a column of the table's *source data*, not to another calc-table cell's computed value — see
below). All three either throw a `ReferenceError`/`SyntaxError` or silently evaluate to nothing.

A cell's own name can also be inserted into the script editor directly by picking it from the editor's
"Cell" folder, rather than typing `$name` by hand — useful context if a human built the sheet through
the Composer UI and a script inherited from them uses this syntax already.

### Referencing the underlying source data

To access the table's bound *source* data (as opposed to another calc-table cell), use the `data`
keyword — this is the freehand-table equivalent of what `field[...]` does when a cell is directly
bound to a column:

```js
// Unique, sorted list of values from the source data's 'Company' field:
toList(data['Company'], 'sort=desc')

// Query-style filtering: value of 'Product' where 'Category' matches the value
// currently held by the cell named 'CategoryCell':
toList(data['Product@Category:$CategoryCell'])
```

Remember to set the cell's expansion (`layoutInfo.setExpansion`, below) whenever a formula returns an
array of values — a formula that returns an array but isn't marked to expand renders only the first
value.

**`data['...']` (and `$name['...']`) is not JS property/array access — it's a string-literal query
DSL, hand-parsed on the Java side.** The bracketed string is a small grammar, not a JS expression:
`col@group:value` filters by field, `;` chains multiple filter conditions, `?expr` filters by an
arbitrary JS boolean expression, `[r1,c1]:[r2,c2]` selects a row range, a leading `*` returns a
sub-table instead of an array, and a leading `=` treats the whole string as a per-row derived-column
expression. None of `@`/`?`/`:`/`;`/`=`/leading-`*` are JS operators here — the JS engine never
parses them; it only evaluates the outer `data[...]`/`$name[...]` call and hands the literal string
to a separate Java parser. Do not try to build this string by composing JS syntax (template
literals, ternaries, etc.) *inside* the brackets — construct the exact string grammar StyleBI
expects.

**A `data[...]`/`$name[...]` reference that resolves to `undefined`, or a group-expansion that
silently collapses to zero rows, is not proof the syntax is wrong.** This exact class of symptom has
a documented history of being a genuine engine-level defect rather than a caller mistake: during the
Rhino → GraalJS engine migration, this string-DSL's member-dispatch path broke silently (GraalJS
only calls the resolver when a separate "does this member exist" check first reports it present,
which a `@`/`?`/`=`-prefixed pseudo-column name naturally fails) and had to be patched in several
separate rounds. If a syntactically-correct `data[...]` expression
that worked before suddenly reads as `undefined` or drops all rows after any engine/runtime upgrade,
treat that as a plugin/StyleBI regression to investigate first — not as a sign the DSL string itself
needs to be rewritten.

See also (for more advanced source-data referencing, not calc-table-specific): "Reference Query Data",
"Reference Datasource Data", and "Run a Query from Script" in StyleBI's dashboard-script docs.

### A date from `data[...]`/`q[...]` is a Java date, not a JS `Date` — `.getFullYear()`/`.getMonth()` throw

A date read out of `data[...]`/`q[...]`/`$name` is a `java.util.Date`/`java.sql.Timestamp`, not a
JS-native `Date`. Per `UserFunctions.md`'s Date Object Functions note, JS `Date` prototype methods only
run on an object built with `new Date(...)` in *this* script — on anything else, GraalJS dispatches to a
same-named Java method instead, and `java.util.Date` has no `getFullYear`/`getMonth`, so it throws
`TypeError: ... is not a function` with no hint that the real cause is "never a JS Date to begin with".

```js
// WRONG — $MonthGrp / q['ORDER_DATE'][i] are Java dates from the query: throws TypeError
if ($MonthGrp.getFullYear() === q['ORDER_DATE'][i].getFullYear()) { ... }

// RIGHT — datePart() is a plain global function, not a Date-prototype method, so it works on either:
if (datePart('yyyy', $MonthGrp) === datePart('yyyy', q['ORDER_DATE'][i])) { ... }
```

A `new Date()` built fresh in the script (e.g. "this year" as a baseline for "last year") IS a real JS
Date, so `.getFullYear()` on that one value is fine — the pitfall is only about data-sourced dates. Use
`datePart`/`dateAdd`/`dateDiff` (`../commonscript/UserFunctions.md`) for arithmetic/comparison on those.

---

## `toList`'s `rounddate` option, and `rowList()` — building what a group/formula cell iterates over

`toList(list, 'rounddate=<year|quarter|month|...>')` buckets dates by that period and returns the
rounded value (first-of-month, etc.) — the documented way to get "Jan 2005"/"Jan 2006" as distinct
buckets instead of collapsing every January together (`date=<levels>` returns the period label instead).
Sort defaults to ascending. It only buckets what's already there, with no year-filter option of its own
— to restrict to "last year's months", filter the result afterward with `datePart('yyyy', d)` above,
rather than guessing at a `rounddate=month,year=2025`-style option that doesn't exist.

```js
toList(q['Order Date'], 'rounddate=month');
// [Jan-2-2002, Feb-21-2004, Feb-25-2004, Nov-25-2005] -> [Jan-1-2002, Feb-1-2004, Nov-1-2005]
```

`rowList(query, 'col ? condition', 'options')` extracts one column's values from rows matching a
condition on another column, and puts every matched row's other fields into scope as `field['colName']`
for a sibling cell — e.g. `rowList(q, 'Date? Total > 10000')` then `field['Category']`. No official
example does a date comparison inside the condition — treat one as unverified DSL-guessing (same caution
as `data[...]`'s own grammar above) and prefer the explicit `datePart()` loop for a year/month filter.

---

## `layoutInfo` — the per-cell configuration API

`FreehandTable1.layoutInfo` is the entry point for every per-cell property below. Every method takes
`row`/`col` (0-indexed cell coordinates) as its first two arguments. Either the unqualified
(`layoutInfo.setX(...)`) or qualified (`FreehandTable1.layoutInfo.setX(...)`) form works inside that
table's own component script; the qualified form is required from `onInit`/`onRefresh` or another
component's script.

### `layoutInfo.setCellBinding(row, col, type, value)`

Sets what a cell holds. `type` is the discriminator:
- `1` — plain static text. `value` is the literal text.
- `2` — a bound field name. `value` is the column name from the table's source data (equivalent to
  `field["ColumnName"]` binding).
- `3` — a formula script. `value` is the script text — this is where `$<name>` cross-cell references
  and `data[...]` source-data references (above) are used.

```js
FreehandTable1.layoutInfo.setCellBinding(1, 0, 1, 'label text');           // static text
FreehandTable1.layoutInfo.setCellBinding(1, 0, 2, 'State');                // bound field
FreehandTable1.layoutInfo.setCellBinding(1, 0, 3,
  "toList(q['Date'], 'sort=asc, rounddate=year')");                        // formula
```

### `layoutInfo.setCellName(row, col, name)`

Names a cell so other cells can reference its computed value as `$name` (above). This is the
prerequisite step for any cross-cell formula — a cell has no name by default.

```js
FreehandTable1.layoutInfo.setCellName(1, 0, 'state');
```

### `layoutInfo.setExpansion(row, col, type)`

Marks a cell to expand into multiple rows/columns when its formula returns an array (typically via
`toList(...)`). `type`: `1` = horizontal expansion, `2` = vertical expansion. This is what makes a
"group" cell actually generate one row/column per distinct value — a group cell with no expansion
renders plausibly with the wrong row/column count and nothing in a tool's return value says so; always
render-check (`get_viewsheet_image`) after setting this.

```js
var q = runQuery('ws:global:Examples/AllSales');
FreehandTable1.layoutInfo.setCellBinding(1, 0, 3, "toList(q['State'], 'sort=asc')");
FreehandTable1.layoutInfo.setExpansion(1, 0, 1); // horizontal expansion
```

### `runQuery(...)` belongs in `onLoad`, not inside a formula cell's own script

Call `runQuery` once in `onLoad`, assign it to a variable (`q` above), and have every formula cell
reference `q[...]`. Calling `runQuery(...)` directly inside a formula cell's own script also
returns a correct value, so it's easy to mistake for correct — but `onLoad` runs once per refresh
while a formula cell's script can evaluate more than once per refresh, so an inline call
re-executes the live query redundantly. A correct-looking result doesn't confirm the placement is
right; check where the `runQuery` call actually sits.

### `layoutInfo.setRowGroup(row, col, name)` / `setColGroup(row, col, name)`

**This is the ONLY pair of these four methods that affects a computed value.** Sets which named
cell (or `'(default)'`, or `null` for a true, position-independent grand total) an aggregate cell's
**aggregation scope** follows — which parent group cell it nests under, i.e. what range a
Sum/Count/etc. is actually computed over. Needed whenever an aggregate cell's automatic (nearest-
enclosing-cursor) scope doesn't line up with the intended dimension cell — e.g. a same-row summary
cell that must aggregate at a *coarser* level than the group cursors already active in that row.

```js
FreehandTable1.layoutInfo.setRowGroup(1, 0, '(default)');
FreehandTable1.layoutInfo.setColGroup(1, 0, 'state');
```

### `layoutInfo.setMergeCells(row, col, Boolean)`

Enables/disables merging for an *expanded* cell — useful when expansion produces duplicate/adjacent
entries that should visually collapse into one merged cell. Not the same as merging two originally
distinct static cells (that's a Composer UI action — "Merge Cells" — with no direct script
equivalent documented here).

```js
FreehandTable1.layoutInfo.setMergeCells(1, 0, true);
```

### `layoutInfo.setMergeRowGroup(row, col, name)` / `setMergeColGroup(row, col, name)`

**Do not confuse this with `setRowGroup`/`setColGroup` above — despite the near-identical name and
signature, this pair is a structurally unrelated, purely visual concern that never affects a
computed value.** It only matters when `setMergeCells(row, col, true)` is also set, and it controls
*where the merge span restarts*, not what any cell sums or counts. This has been independently
misdiagnosed more than once — each time with a real, reproducible test case that looked like solid
evidence of a defect — because the symptom ("I set `mergeRowGroup` and the value didn't change") is
real and reproduces every time; these fields genuinely are inert with respect to computed values,
by design. If a diagnosis or a change concludes these fields should affect a value, that conclusion
is wrong regardless of how solid the supporting evidence looks; the fields that actually control
scope are `setRowGroup`/`setColGroup` above.

**A same-labeled UI trap, worth knowing before trusting a screenshot or a manual repro report**: the
native Composer UI has two separate panels that reuse the identical label for two unrelated
controls. The **Row Group**/**Column Group** dropdowns under the cell's grouping settings write
`rowGroup`/`colGroup` (real aggregation scope); the same-labeled **Row Group**/**Column Group**
dropdowns under the "Merge Expanded Cells" checkbox write `mergeRowGroup`/`mergeColGroup` (merge
span only), and are greyed out until that checkbox is on. If a user describes what they clicked, or
you're reading a screenshot, confirm which panel — the label alone does not say which field it
wrote.

**A third, unrelated mechanism, so as not to conflate three things that all involve the word
"merge"**: `modify_calc_layout(op:"mergeCells", row, col, rows, cols)` is a *static, design-time*
structural merge of fixed grid cells (like a spreadsheet's merge-cells), stored as span data on the
table layout itself — unrelated to both `setMergeCells`'s boolean (dynamic expand-time merging) and
`setRowGroup`/`setMergeRowGroup` (aggregation scope vs. merge-span identity). Same English word,
three different mechanisms in this one feature area.

Because it is only a merge-span control, it only makes coherent sense on a **GROUP** cell (whose
label is expected to repeat identically across its own expand range) — never on a **SUMMARY**
cell, where every expanded instance is *supposed* to differ. If a summary/aggregate cell's value
looks wrong and the fix under consideration is "set its `mergeRowGroup`/`mergeColGroup`", stop —
that is the wrong field pair; use `setRowGroup`/`setColGroup` instead.

The `name` argument also is not equivalent to `setRowGroup`'s: with `setMergeCells(row, col, true)`
active, `'(default)'` restarts the merge span every time the bound value changes (the usual
"collapse repeated values into one cell" look), while `null` merges the cell's *entire* expand
range into a single span unconditionally, always labeled with the first row's value — regardless of
how many distinct values it actually spans. Picking `null` when `'(default)'` was meant renders one
mislabeled cell across what should have been several distinct merged groups, with no error anywhere.

```js
FreehandTable1.layoutInfo.setMergeRowGroup(1, 0, 'state');
FreehandTable1.layoutInfo.setMergeColGroup(1, 0, 'state');
```

### Group-total / subtotal rows, and merging a group label across them

A subtotal row (e.g. one "Subtotal" per Region, right after that region's own detail rows) needs no
group/expand cell of its own: give its summary cells a `rowGroup` naming the detail row's group cell
(not `'(default)'`), and the row repeats once per group value as that cell's sibling.

To visually merge the group's label across the detail block and the subtotal row (one "USA East"
spanning both, confirmed working in the Composer UI via Ctrl-click + "Merge Cells"), use the STATIC
structural merge — `modify_calc_layout(op:"mergeCells", row, col, rows, cols)` — anchored at the
detail row's group cell, spanning down into the subtotal row.

**Not `mergeRowGroup`/`setMergeRowGroup`.** That only collapses one cell's own repeated instances; it
cannot fuse a DIFFERENT cell (the subtotal row's) into the span, even pointed at the same named
group — confirmed by live testing: round-trips clean on `get_cell_binding`, no error, no visible
merge. Same silent-no-op shape as the `rowGroup`/`colGroup` trap above.

The static merge does discard the merged-away cell's own binding (`{mergedInto: {row, col}, binding:
null}`) — harmless here since that cell wasn't driving the subtotal row's repetition (see above), but
check `get_calc_layout` first if the cell being merged away still carries a live `expand` binding.

### `layoutInfo.setSpan(row, col, width, height)`

Sets how many cells (horizontally/vertically) a single cell should visually span, starting from
`(row, col)`.

```js
FreehandTable1.layoutInfo.setSpan(0, 0, 2, 1); // span two columns
FreehandTable1.layoutInfo.setSpan(0, 0, 1, 2); // span two rows
```

---

## Table-level properties (not per-cell)

### `fillBlankWithZero` (Boolean)
Populates empty result cells with `0` instead of leaving them blank (empty cells occur when no data
matches a given row/column heading combination).
```js
fillBlankWithZero = true;
```

### `keepRowHeightOnPrint` (Boolean)
Preserves on-screen row heights when exported via a print layout, instead of StyleBI's default
auto-adjustment for print.
```js
FreehandTable1.keepRowHeightOnPrint = true;
```

### `sortOthersLast` (Boolean, default `true`)
When Top/Bottom ranking's "Group all others together" is enabled, controls whether the "Others" group
is forced after every ranked group (`true`, ignoring the specified dimension sort) or placed according
to that sort (`false`).
```js
sortOthersLast = false;
```

---

## Row/column insertion — a known open question, not answered by the official docs

StyleBI's UI supports inserting/appending/deleting rows and columns via right-click context menu
(`Insert Row`, `Append Row`, `Insert Column`, `Append Column`, `Delete Row`, `Delete Column`) and
merging multiple originally-distinct cells (`Ctrl`-click select, then "Merge Cells" — for combining
static cells, not the same as `setMergeCells`'s expanded-cell merging above). The official
documentation does not state whether an existing `$<name>` cross-cell reference automatically survives
a row/column insertion that shifts the referenced cell's position, or whether references are resolved
by name (survives insertion) vs. position (would need rewriting). **This is worth confirming
empirically** rather than assuming either way — the naming mechanism above strongly suggests references
are name-based (looked up by the `$name` string, not by `{row, col}`), which would make them
insertion-safe by construction, but this reference doc does not itself state that as a documented
guarantee.
