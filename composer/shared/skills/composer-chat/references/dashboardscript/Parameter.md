# `parameter` — the viewsheet's parameter/context map

Source: https://www.inetsoft.com/docs/stylebi/InetSoftUserDocumentation/1.0.0/dashboardscript/parameter.html
— fetch this URL with `curl`/raw HTML if re-verifying; `WebFetch`'s AI-summarized read of this
page has returned a plausible-sounding but wrong/incomplete answer for this exact topic before.

## The one syntax that matters most

```js
parameter.variableName
```

`variableName` is the name of **either** a Variable asset defined in the Data Worksheet (created
via `add_variable`) **or a Form component defined in the Dashboard** — a ComboBox, RadioButton,
Spinner, Slider, TextInput, CheckBox, etc., referenced **by its own assembly name, directly, with
no `add_variable`/`set_variable_values` step required**. Both live in the same parameter store;
the syntax does not distinguish them.

Official examples from the doc page:

```js
parameter.stateSelector     // a worksheet Variable asset named "stateSelector"
parameter.RadioButton1      // a Form component named "RadioButton1" — its own current value
parameter.myParamName = 'Hello';   // parameters can be WRITTEN too, not just read
```

**This is the same store `set_condition`/`set_highlight`'s `{type: "variable", name: "X"}` value
reads.** `{type: "variable", name: "ComboBox1"}` and `parameter.ComboBox1` are two shapes over the
same value — a bare reference vs. a JS expression that can compute on it (e.g.
`parameter.RevenueThreshold * 0.9` for "90% of the threshold"). Neither needs an
`add_variable`-created worksheet variable when the name is a form component's own assembly name —
confirmed live against a connected viewsheet.

## Other `parameter` members (all read-only, all confirmed from the doc page)

| Member | Type | What it returns |
|---|---|---|
| `parameter._USER_` | String | The current user's username. |
| `parameter._ROLES_` | Array of Strings | Roles the current user belongs to. |
| `parameter._GROUPS_` | Array of Strings | Groups the current user belongs to. |
| `parameter.__LINK_HOST__` | String | Hostname:port of the server hosting the dashboard. |
| `parameter.__LINK_URI__` | String | Root URL the current dashboard is served from. |
| `parameter.__principal__` | Object (`SRPrincipal`) | Session-identity object; see the doc site's "Access the User Session" page for writing to it. |
| `parameter.length` | Integer | Count of currently defined parameters. |
| `parameter.parameterNames` | Array of Strings | Names of every currently defined parameter — use this to discover what's available rather than guessing a name. |

```js
alert('Logged in as ' + parameter._USER_)
alert('User roles: ' + parameter._ROLES_.join(', '))
alert(parameter.parameterNames[0])
```

**A bare `_USER_`/`_ROLES_`/`_GROUPS_` (no `parameter.`/`parameter[...]` prefix) throws
`ReferenceError`** — these are keys inside `parameter`, never standalone globals, despite the
naming making them look like they should be.

## Other sources of parameters (per the doc page's own list)

- Data Worksheet Variables (`add_variable`).
- Form components in the Dashboard (a form assembly's own name — see above).
- Parameters passed into the Dashboard via a hyperlink (drill-through).
- Session parameters (`_USER_`/`_ROLES_`/`_GROUPS_`/etc., above).
- Parameters defined directly in Dashboard script, either `parameter.name = value` or implicitly
  via a Condition's "Variable" option in the Composer UI.
- Parameters passed into an embedded Dashboard from a parent Dashboard, via `thisParameter` — a
  **distinct** reference from `parameter`, scoped more narrowly (to the current script/event
  rather than the whole dashboard); don't assume it's a synonym for `parameter`.
