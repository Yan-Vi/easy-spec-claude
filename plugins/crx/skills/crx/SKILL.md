---
name: crx
description: Use when creating, editing, inspecting, or replaying Flows / Page Objects / Scenarios in an Easy Spec - Playwright project (flows/*.flow.json, pages/*.page.json, scenarios/*.scenario.json, variables.json) via the `easyspec` CLI or the playwright-easy-spec MCP tools -- e.g. "add a step to the login flow", "create a page object", "replay this scenario", "what flows exist in this project", "list the steps in <flow>".
---

# Easy Spec - Playwright: CLI & MCP

This repo's own tooling for authoring/replaying flows in *another* Playwright project's data
(flows/pages/scenarios/variables.json) -- not the extension UI itself. Two front ends
(`cli/index.js` and `mcp/server.js`), one shared core (`lib/projectCore.js`'s `ProjectCore` +
`lib/codegen.js`): neither front end has any logic the other lacks, so pick whichever fits the
moment. `ProjectCore` is storage-agnostic (an injected adapter -- Node `fs` or the browser's File
System Access API) and is the exact same class the side panel itself now uses internally, with one
unified undo/redo log -- an edit made via a tool call and one made by clicking around the UI are
genuinely indistinguishable to it, including for Ctrl+Z.

## Which front end to use

- If the `playwright-easy-spec` MCP server is connected (check the available tools), prefer it --
  structured input/output, no JSON-escaping through a shell, and its `live_*` tools can drive the
  user's actual open browser tab (real login/session), not just generate files.
- Otherwise shell out to `easyspec` (this repo's bin) or `node cli/index.js` from the repo root.

## Replaying a saved flow vs. driving the tab manually

Two different jobs, don't default to one for both:

- **The user asks for an action that takes several steps, and a flow/scenario already covers it**
  (check `list_flows`/`list_scenarios` first if unsure) -- replay it (`live_replay_flow` if a side
  panel's connected, `replay_flow` otherwise) rather than re-driving each step by hand via
  `live_run_step`. It's the tested, maintained path: real Page Object selectors, whatever
  params/dataset the flow already defines, one call instead of several that could each drift from
  what the flow actually encodes.
- **The user asks to explore/investigate** ("what's on this page", "look for X", "check whether
  Y", anything open-ended rather than a named known procedure) -- prefer the manual workflow
  (`live_snapshot` to see the page, `live_run_step`/`live_pick_element` to poke at it) even if a
  saved flow loosely resembles it. Exploration means the actual steps aren't decided yet; forcing
  it through a fixed flow assumes an answer that's the point of exploring to find out.

## Resolving which project

Every file-based tool/command needs the *target* Playwright project's root folder (the one
containing `flows/`, `pages/`, `scenarios/`, `variables.json` -- generally NOT this extension
repo), but MCP tools resolve this live-first: if a connected side panel's folder *name* matches
`project`/`EASYSPEC_PROJECT` (or there's exactly one connected panel and neither was given), the call
runs against THAT panel's own live `ProjectCore` instance instead of reading/writing files
directly -- same result either way, but a live-connected edit shows up in the UI immediately and
shares that panel's undo history, while an fs-only edit (no panel connected) does not.

- CLI: `--project <dir>`, else `$EASYSPEC_PROJECT`, else the current directory. Always direct-fs, never
  live (an `easyspec` invocation is a one-shot process, not something a side panel can talk to
  mid-command).
- MCP: `project` arg, else the `EASYSPEC_PROJECT` env var the server was launched with, else -- live
  resolution only -- the sole connected side panel (call `live_status` first to see what's
  connected; a browser folder handle only ever exposes a *name*, never a real path, which is
  exactly why live resolution matches on name rather than path). `replay_flow`/`replay_scenario`
  are the one exception -- they always need a real fs path (they shell out to `npx playwright
  test`), live panel or not.

## Data model

- **Flow**: `{ id, name, folder, description, steps[], flowData[], stateVars[], outputData[] }`.
  Each step: `{ kind, method, selector? | pageObjectName+pageObjectMethod+pageObjectArgs? |
  elementAlias?, args[], options{}, negate?, ... }`. `kind` is one of `locator | page | assert |
  variable | raw`, deliberately mirroring Playwright's own API -- see `lib/actionRegistry.js` for
  every supported method/matcher and its exact param/option shape before constructing a step by
  hand.
- **Page Object**: `{ name, folder, methods: { methodName: selector } }` -- shared across every
  flow that references it. A selector containing `${paramName}` (bare identifier only) becomes a
  parametric method automatically.
- **Scenario**: `{ id, name, folder, flows: [{ flowId, params }] }` -- ordered flow composition,
  generates a runnable `<id>.spec.ts`.
- **Step path**: dot-separated indices into nested `.steps` arrays (Conditional/Repeat/Iterate
  bodies), e.g. `"2.0"` = the 1st step inside the 3rd top-level step's own body.

## CLI command reference (`easyspec`)

```
list flows|scenarios|pages|variables
show flow|scenario|page <id>

flow create <name> [--id ID] [--folder F] [--description D]
flow delete <id>
flow meta <id> [--name] [--folder] [--description]
flow add-step <id> [--at N] [--parent PATH] --kind K [--method M] [--selector S]
              [--page-object PO] [--element METHOD] [--po-args JSON] [--args JSON]
              [--options JSON] [--negate] [--variable NAME]
flow update-step <id> <path> [any of the add-step flags, partial]
flow remove-step <id> <path>
flow copy-steps <sourceId> <from> <to> <targetId> [--source-parent PATH] [--target-parent PATH] [--at N]
flow dataset <id> <name> <jsonValue>
flow remove-dataset <id> <name>
flow state-var <id> <name> <initialJsonValue>
flow remove-state-var <id> <name>
flow output <id> <name> <defaultJsonValue>
flow remove-output <id> <name>

page create <name> [--folder F]
page delete <name>
page set-locator <name> <method> <selector>
page remove-locator <name> <method>

scenario create <name> [--id ID] [--folder F] [--description D]
scenario delete <id>
scenario add-flow <id> <flowId> [--params JSON]
scenario remove-flow <id> <index>

var set <name> <jsonValue>
var unset <name>

replay flow <id> [--params JSON] [--dataset NAME] [--keep]
replay scenario <id>
```

`--args`/`--options`/`--params`/`--po-args` all take a JSON string. Everything after a bare `--`
on a `replay` command passes straight through to `npx playwright test`. Full help: `easyspec --help`.

## MCP tools (server name: `playwright-easy-spec`)

- **Inspect**: `list_flows`, `list_scenarios`, `list_page_objects`, `get_variables`, `get_flow`,
  `get_scenario`, `get_page_object`
- **Author flows**: `create_flow`, `delete_flow`, `set_flow_meta`, `add_step`, `add_multiple_steps`,
  `update_step`, `remove_step`, `copy_steps`, `set_flow_dataset`, `remove_flow_dataset`,
  `set_state_var`, `remove_state_var`, `set_output_field`, `remove_output_field`. `copy_steps`
  copies a `[from, to]` range of steps from one flow into another (or the same flow), deep-cloned,
  source untouched -- use it instead of reading a flow's steps and re-adding each one by hand (e.g.
  splitting one flow into two, or reusing a chunk of steps another flow already has).
  `add_multiple_steps` takes a `steps` array and adds them all in one call/one undo entry -- prefer
  it over N `add_step` calls when recording several steps at once (e.g. one recorded step per
  `live_run_step`/`live_run_multiple_steps` action during a live exploration session).
- **Page objects**: `create_page_object`, `delete_page_object`, `set_locator`, `remove_locator`
- **Scenarios**: `create_scenario`, `delete_scenario`, `add_flow_to_scenario`,
  `remove_flow_from_scenario`
- **Variables**: `set_variable`, `unset_variable`
- **Replay** (fresh browser via `npx playwright test`, no login state): `replay_flow`,
  `replay_scenario`
- **Live** (drives the user's actual open tab through the side panel's own `chrome.debugger`
  session -- requires the side panel open and connected): `live_status`, `live_replay_flow`,
  `live_pick_element`, `live_run_step`, `live_run_multiple_steps`, `live_snapshot`,
  `live_screenshot`, `live_start_run`, `live_list_runs`, `live_get_run`, `live_cancel_run`,
  `live_detach`. `live_snapshot`/`live_run_step`/`live_run_multiple_steps`/`live_screenshot` keep
  the tab's `chrome.debugger` session attached across a quick back-and-forth burst of calls
  (idle-timeout detach, ~20s) rather than detaching after each individual one -- detaching
  mid-exploration was observed to close transient page UI (an open dropdown) even with no click in
  between, since it's the attach/detach cycle itself that disturbs the page, not just a click. Call
  `live_detach` when done exploring so Chrome's debugging banner doesn't linger on the tab.
  `live_run_step` isn't just for actions -- it
  returns whatever the underlying Playwright call itself returned (`returnValue`), so a read like
  `{kind:"locator", method:"count", selector:"li.item"}` or `{kind:"page", method:"title"}` works
  too, not just click/fill/goto. `live_run_multiple_steps` runs a whole ordered sequence in one
  call sharing one variable context (a `variable` step's captured value can feed a later step's own
  expression arg, same as inside a real flow), stopping at the first failing step -- prefer it over
  several `live_run_step` calls back to back once the sequence is already decided, both for fewer
  round trips and because the tab stays claimed for the whole sequence instead of being released
  and re-claimed between each action. `live_snapshot` (an ARIA/accessibility tree, text) is the
  normal way to find a selector to act on next; reach for `live_screenshot` only when the question
  is genuinely visual. `live_replay_flow` blocks until the run finishes (ok/fail); `live_start_run`
  is the non-blocking sibling -- returns a `runId` immediately, poll with `live_get_run`/
  `live_list_runs`, stop early with `live_cancel_run`. Either way, a run is isolated to the tab it
  started on (one run per tab -- a second one on a busy tab is rejected, not queued) and shows up
  live in the panel: per-step status dots on the open Flow editor's step rows (if that flow's open)
  and an entry in the Runs tab (current + session history, capped at 30).

## Keep flows minimal

Prefer the fewest steps that correctly reproduce the real sequence of actions. A repeated run of
single-key `press` steps to type a string (one step per character) is a clear no-go -- it bloats
the flow for no benefit and there's already a real fix: `pressSequentially` (a whole string, real
per-character keydown/keyup events in one step -- see the masked-input gotcha below). If a
multi-step pattern like that shows up anywhere else, look for the one-step equivalent (a single
locator method, a batched multi-select tool call, etc.) before accepting N near-identical steps as
the answer. This also applies to the authoring side: `live_run_multiple_steps`/`add_multiple_steps`
exist specifically so exploring/recording a sequence doesn't need one MCP round trip per action.

## Gotchas

- `set_state_var`'s `initial`, `set_output_field`'s `default`, and `set_variable`'s `value` are
  schema-less (`z.any()`) -- unlike an object/array-typed field (e.g. `set_flow_dataset`'s
  `value`), there's no schema hint telling the call to JSON-parse the argument, so it's passed
  through as a raw string. Write a plain string value with NO surrounding quotes (`Direct`, not
  `"Direct"`) -- typing `"Direct"` stores the literal 8-character string `"Direct"`, quote marks
  and all. Confirm with `get_flow`/`get_variables` after setting one of these if unsure.
- Every mutating call regenerates BOTH the `.json` source of truth and the generated `.ts` --
  never hand-edit a generated `.ts` file, it's overwritten on the next save.
- `add_step`/`flow add-step` needs `kind` at minimum, plus whichever of
  `selector` / `pageObjectName`+`pageObjectMethod` / `elementAlias` that kind requires, and
  `method` for anything but a bare `raw` step.
- `raw` steps aren't executable via `live_run_step` or Replay -- there's no structured call to
  make; they still generate correct code.
- Deleting a page object locator (`remove_locator`/`page remove-locator`) is blocked if any flow
  step still references that method -- update or remove those steps first.
- A flow referencing a global variable needs `variables.ts` to already exist on disk (Variables
  saved at least once) before its own generated `.flow.ts` import resolves.
- `replay_flow`/`replay_scenario` genuinely run `npx playwright test` inside the *target*
  project -- that project needs Playwright installed/configured there (browsers installed,
  `playwright.config.ts` present), independent of this repo's own dependencies.
- `live_replay_flow`/`live_pick_element`/`live_run_step` target whichever tab is pinned in the
  side panel's header (the "Target: ..." button next to Connect folder), or the browser's actual
  active tab if none is pinned -- pin one before a live call if the user might be focused on a
  different tab (e.g. reading something else) while it runs.
- On a target app whose dropdown/overlay widgets leave their closed panel in the DOM for a moment
  (Angular component libraries with animated overlay panels -- e.g. PrimeNG's `p-dropdown` -- are a
  common source of this, since the closing panel stays mounted through its leave animation instead
  of being removed immediately), a just-closed panel can still be fully "visible" to Playwright
  right as the next one opens -- so a bare `role=option[name="X"]`-style selector throws a
  strict-mode "resolved to 2 elements" error whenever the stale and fresh panels happen to share an
  option label (common across a form with several similar-shaped questions, e.g. repeated
  Yes/No/decline choices). Neither `[selected=false]` nor `>> visible=true` reliably disambiguates
  (the stale item can be unselected AND still report visible mid-transition). What held up under
  repeated automated replay: scope to the LAST matching panel, which is the freshly-opened one
  whenever the library appends new overlay panels rather than reusing the old one --
  `<panel-selector> >> nth=-1 >> role=option[name="${value}"]` (`nth=` accepts negative indices).
  If a form has several fields sharing an option vocabulary, apply this proactively to all of them
  rather than only the one observed to fail -- the collision is real but easy to miss in a one-off
  manual click, and shows up reliably only under full-speed automated replay.
- A masked/formatted input (a date field enforcing `MM/DD/YYYY`, a phone/SSN mask, etc.) can reject
  `fill()` outright -- the mask's own JS listens for real keystroke events to validate/reformat as
  you type, and `fill()` sets the value directly without dispatching them, so the field can look
  filled in a snapshot immediately after but silently revert (or fail a "required" check) once
  something else touches the page. The fix is `pressSequentially` (one step, the whole string, real
  per-character key events) -- not a `press` step per character, which works but is exactly the
  step-bloat this skill's "keep flows minimal" section above says to avoid.
- Don't assume a flaky SPA step (a click that doesn't visibly "stick" when steps run back-to-back,
  but works fine when replayed slowly by hand) is a simple timing race fixable with a wait --
  `waitForTimeout`, `waitForLoadState('networkidle')` in either position, and even a genuine
  multi-second real delay (via `waitForFunction` polling elapsed time) can all fail to help against
  this kind of flake: the click reaches the page fine but its effect gets silently swallowed, not
  delayed, so waiting longer doesn't fix it. Inserting extra `assert` steps to catch/verify it (a
  reasonable-looking fix) can make it measurably *worse* -- each assert is its own
  chrome.debugger/CDP round trip, and that extra traffic right after the click can actively
  interfere with the page the same way the idle-detach issue above does (the CDP protocol activity
  itself disturbs the page, independent of any click). If a step is flaky in this specific way,
  don't reach for waits or extra verification steps by default -- try simply leaving the flow as
  the minimal sequence of real actions with nothing extra between them first.
- If the flake above is tied to a specific async action (e.g. a Save button whose write hasn't
  actually landed yet when the next step fires), look for a genuine completion signal before
  falling back to a blind wait -- many apps show a toast/snackbar on a successful save. Test for it
  with a real, complete, valid save first -- an empty/invalid one may skip the toast entirely and
  wrongly suggest none exists. If one does exist, assert `toBeVisible` on it (confirms the save's
  own async work actually completed, not just that the click landed) and then `toBeHidden` right
  after (most toasts auto-dismiss after a few seconds; waiting for that gives the page's own
  re-render genuinely time to settle before the next step touches it, since the toast appearing
  doesn't mean the surrounding DOM has stopped moving yet). This is a real condition-based wait, not
  a duration guess -- prefer it over `waitForTimeout` whenever such a signal exists.
- Don't assume every Save button in a multi-section form shows the same completion signal (or any
  at all) just because one of them does -- verify each one independently. Observed on one real form:
  of ~14 Save buttons, only 3 ever showed a toast; the rest saved with no visible confirmation
  whatsoever, and adding a `toBeVisible` assert on a signal that never appears just makes a
  previously-fine step fail every run. Where a toast *does* exist, two more traps: (1) some
  toasts are gated on an actual value diff -- clicking Save again with the field unchanged from what
  the server already has produces no toast at all, so testing "does this button show a toast" by
  repeatedly re-clicking Save with the same data can wrongly conclude "no toast" when a genuinely
  fresh fill would trigger one; force a real change (or use a genuinely blank/unfilled record) before
  concluding a signal doesn't exist. (2) a single toast message can be broadcast to several
  simultaneously-mounted component instances at once (a shared/global toast service rendering into
  each one), so scoping the assert to one specific component (`<component-tag> >> text=...`, mirroring
  which component happened to be active when it was first observed) is fragile -- which components are
  mounted can depend on prior tab-navigation history in ways that hold for one flow's fixed step order
  but aren't a real guarantee. Preferring `text=<the message> >> nth=0` (just take the first match,
  uninterested in which mounted copy it is) sidesteps this fragility entirely and is the more robust
  choice for a new toast-wait; an existing one already proven reliable in place doesn't need touching
  just for consistency.
- A step recorded against a dropdown's unset placeholder text (e.g. `text=Select Option`/`Select
  Result`) can fail when re-run via `live_run_step_range`/Replay against a record where that same
  field is already set -- once a value is chosen, the widget shows the picked value instead of the
  placeholder, so the placeholder text the step targets is simply gone. This can persist two ways,
  and it's not always obvious which: some widgets just hold the picked value client-side for the
  rest of the browser session (a real page reload resets them back to the placeholder); others are
  tied to a field that's genuinely saved server-side once its own Save button is clicked, and stay
  showing the saved value forever, surviving even a full reload, until the value is explicitly
  changed again. Either way this is not a flow bug -- it only shows up when re-running against a
  record that's already (partly) filled; the original live fill is still correctly recorded, and the
  step works fine against a genuinely fresh/blank record. Don't rewrite the selector to also match
  an already-selected value just to make a stale re-test pass -- verify correctness at fill-time via
  `live_snapshot` instead (confirm the field shows what you intended right after setting it), the
  same way every other section in this flow was verified. A full end-to-end replay is only a
  meaningful test against a fresh record for this reason -- re-running it against the same
  already-filled one will predictably fail at the first server-persisted field it reaches.
