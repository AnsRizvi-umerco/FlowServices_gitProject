# Hand-Authoring webMethods / IWHI Flow Services via Code — Field Handbook

This is a condensed knowledge transfer of everything learned building multiple production flow services for IBM webMethods Hybrid Integration (IWHI) entirely by writing the on-disk XML files directly (never through the browser flow editor), across several real projects and many rounds of live testing. Give this file to any Claude agent at the start of a session where the user wants flow services built "via code" / "not by browser," and it should be able to produce working, git-committable flow services without repeating the trial-and-error documented here.

## 1. The core distinction: DAFS vs. Flow Service

IWHI has two kinds of flow service assets:

- **"Flow Service"** — runs in the cloud, is edited/saved entirely server-side, and **never syncs to a linked git repo**, no matter what auto-commit settings are enabled.
- **"Deploy Anywhere Flow Service" (DAFS)** — runs on a registered runtime (e.g. "Cloud Designtime") and **auto-commits to the linked git repo on every save** in the browser, and can be built entirely by writing files directly into that repo.

If the user wants git-tracked, code-authored integration logic, it must be a DAFS. If they created something as a plain "Flow Service" and wonder why it's not showing up in git, that's the answer — tell them to recreate it as a DAFS.

## 2. On-disk project layout

Once a project's repo is cloned, DAFS assets live at:

```
ns/project/<package_name>/integrations/<ServiceName>/
├── node.ndf     ← metadata: signature (sig_in/sig_out), display name, step hints
└── flow.xml     ← the actual FLOW/INVOKE/MAP/BRANCH/LOOP logic
```

Other relevant top-level files:

- `config/scaffolding/<ProjectName>.yml` — **platform-managed, do not hand-edit.** It lists package dependencies (e.g. `WmB2BProvider`, `WmSlackProvider`) per service. The platform writes to this itself the first time you exercise a dependent connector/adapter step in the browser UI (opening it in the mapper is often enough). If you hand-edit it and the platform also touches it independently, you'll get a git "checkout conflict" on next pull — see §8.
- `manifest.v3`, `config/aclmap_pkg.cnf` — also platform-managed bookkeeping files that appear/change automatically; don't need to be authored by hand.

## 3. `node.ndf` structure

Minimum skeleton (always present):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Values version="2.0">
  <value name="svc_type">flow</value>
  <value name="svc_subtype">unknown</value>
  <value name="svc_sigtype">unknown</value>
  <record name="svc_sig" javaclass="com.wm.util.Values">
    <record name="sig_in" javaclass="com.wm.util.Values">
      <!-- node_type/record wrapper + rec_fields array of input fields -->
    </record>
    <record name="sig_out" javaclass="com.wm.util.Values">
      <!-- same shape, output fields -->
    </record>
  </record>
  ...
  <record name="node_hints" javaclass="com.wm.util.Values">
    <value name="createdBy">user@company.com</value>
    <value name="createdDate">1784311291326</value>       <!-- epoch millis, REQUIRED -->
    <value name="lastModifiedBy">user@company.com</value>
    <value name="lastModifiedDate">1784311291326</value>   <!-- epoch millis, REQUIRED -->
    <value name="displayName">ServiceName</value>
    <value name="description"></value>
    <record name="stepNodeHints" javaclass="com.wm.util.Values">
      <!-- one entry per flow.xml step, see §5 -->
    </record>
  </record>
  ...
</Values>
```

**Gotcha:** if `createdDate`/`lastModifiedDate` are omitted (only `createdBy` present), the project tile in the IWHI UI shows **"Updated: Invalid date."** Always include both, as plain epoch-millisecond integers (`date +%s%3N` in bash).

### Signature field declaration shape (sig_in / sig_out)

Each field in `rec_fields`:

```xml
<record javaclass="com.wm.util.Values">
  <value name="node_type">field</value>
  <value name="node_subtype">unknown</value>
  <value name="node_comment"></value>
  <record name="node_hints" javaclass="com.wm.util.Values">
    <value name="field_largerEditor">false</value>
    <value name="field_password">false</value>
    <value name="field_usereditable">false</value>
  </record>
  <value name="is_public">false</value>
  <value name="field_name">yourFieldName</value>
  <value name="field_type">string</value>
  <value name="field_dim">0</value>
  <value name="nillable">true</value>
  <value name="form_qualified">false</value>
  <value name="is_global">false</value>
</record>
```

## 4. `flow.xml` structure and step types

Root:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<FLOW VERSION="3.2" CLEANUP="true">
  <COMMENT></COMMENT>
  <!-- nodes -->
  ... steps ...
</FLOW>
```

### INVOKE (calling a built-in or custom service)

```xml
<INVOKE NAME="DisplayLabel" SERVICE="pub.string:tokenize">
  <COMMENT></COMMENT>
  <!-- nodes -->
<MAP MODE="INPUT">
  <COMMENT></COMMENT>
  <MAPTARGET>
    <!-- the SERVICE's OWN fixed input schema, see §6 -->
  </MAPTARGET>
  <MAPSOURCE>
    <!-- the pipeline's currently-available fields, see §6 -->
  </MAPSOURCE>
  <!-- nodes -->
<MAPCOPY FROM="/pipelineField;1;0" TO="/serviceInputField;1;0"/>
<MAPSET NAME="Setter" OVERWRITE="true" VARIABLES="false" GLOBALVARIABLES="false" FIELD="/serviceInputField2;1;0">
  <DATA ENCODING="XMLValues" I18N="true">
<Values version="2.0">
  <value name="xml">literal or %substituted% value</value>
</Values>
</DATA>
</MAPSET>
</MAP>

<MAP MODE="OUTPUT">
  <COMMENT></COMMENT>
  <MAPCOPY FROM="/serviceOutputField;1;0" TO="/pipelineFieldName;1;0"/>
</MAP>
</INVOKE>
```

- If the service's own output field name already matches the pipeline variable name you want, the OUTPUT map can be empty (`<MAP MODE="OUTPUT"><COMMENT></COMMENT></MAP>`) — the value just flows through by name.
- **A lightweight INVOKE (MAP body with only `MAPCOPY`, no `MAPTARGET`/`MAPSOURCE`) can execute successfully at runtime**, but the Designer's mapper UI will show: *"The mappings listed are not displayed correctly... Do you want to delete these invalid mappings?"* — annoying but non-fatal. **Always include full `MAPTARGET`/`MAPSOURCE` on the INPUT map** to avoid this, using the service's real declared input schema for MAPTARGET and the current pipeline state for MAPSOURCE.

### MAPCOPY / MAPSET path suffix convention (`;X;Y`)

Every field reference in `MAPCOPY FROM=`/`TO=` and `MAPSET FIELD=` needs a `;X;Y` suffix:

- `X` = `1` for scalar/string fields, `2` for record fields **and record arrays** (i.e. it reflects field *type*, not whether it's a list).
- `Y` has been `0` in every observed working case.

Examples: `/edidata;1;0` (string), `/document;2;0` (record), `/document/payload/lines;2;0` (record array — **not** `;1;0`, that's a real bug we hit that silently makes the referenced value resolve to empty/zero at runtime with no error at save time).

**Omitting the suffix entirely** (e.g. `FIELD="/num2"` instead of `FIELD="/num2;1;0"`) causes the literal value to silently never be set — manifesting later as a runtime `Missing Parameter: <name>` error, with no indication at save/open time that anything was wrong. Always double check every `MAPSET FIELD=` and `MAPCOPY FROM=`/`TO=` has the suffix.

### MAPSET value substitution (`%...%`)

Inside a `MAPSET` with `VARIABLES="true"`, the literal text in `<value name="xml">` supports:

- `%fieldName%` — substitutes a scalar pipeline field.
- `%parentField/childField%` — nested document field access (e.g. `%document/buyerName%`).
- `%arrayField[N]%` — array element by index (e.g. `%segFields[2]%`).
- You can concatenate accumulator + new content in one MAPSET: `%xmldata%<newTag>%value%</newTag>` — this is the standard pattern for building up a string across loop iterations or sequential steps (see §7).

Use `VARIABLES="false"` for pure literal values with no substitution (e.g. setting a delimiter to `"*"` or a status to `"SUCCESS"`).

### TRANSFORM (standalone Map step, no service call)

Used for building/setting values with pure MAPSET/MAPCOPY, no INVOKE:

```xml
<MAP NAME="TRANSFORM" TIMEOUT="" MODE="STANDALONE">
  <MAPTARGET>...</MAPTARGET>
  <MAPSOURCE>...</MAPSOURCE>
  <!-- nodes -->
<MAPSET .../>
</MAP>
```

### LOOP

```xml
<LOOP NAME="RepeatLabel" IN-ARRAY="/document/payload/lines"  MAX-THREADS="1"  PARALLEL-ERROR-HANDLING="reportError">
  <COMMENT></COMMENT>
  <!-- nodes -->
  ... steps using the current item ...
</LOOP>
```

- If you also want the (possibly modified) items written back, add `OUT-ARRAY="/sameArrayPath"`.
- **Looping over a plain string array** (e.g. `/valueList`): inside the loop, the current element is exposed as a *scalar* under the exact same field name (`valueList`), not a nested path.
- **Looping over an array of records** (e.g. `/document/payload/lines`): inside the loop, the current record's fields are exposed at the *same nested path* the array itself occupied (e.g. `%document/payload/lines/quantity%`), not flattened to the pipeline root.
- Fields that are **not** part of the loop's IN-ARRAY/OUT-ARRAY (e.g. an accumulator string, or a running list) persist correctly across iterations and after the loop ends — this is the standard way to build up a result inside a loop (see §7).

### BRANCH (conditional / switch)

Two flavors seen working:

**Regex-label validation branch** (evaluate a pipeline field against a pattern):

```xml
<BRANCH LABELEXPRESSIONS="true">
  <COMMENT></COMMENT>
  <!-- nodes -->
<SEQUENCE NAME="fieldName = /^regexPattern$/" EXIT-ON="FAILURE">
  <!-- runs if the regex matches -->
</SEQUENCE>
<SEQUENCE NAME="$default" EXIT-ON="FAILURE">
  <!-- nodes -->
<EXIT NAME="EXIT" FROM="$parent" SIGNAL="FAILURE" FAILURE-MESSAGE="Descriptive message with %fieldName%">
  <COMMENT></COMMENT>
</EXIT>
</SEQUENCE>
</BRANCH>
```

This is the standard pattern for "validate input, and if invalid, throw a Flow failure with a message" (which a wrapping Try/Catch, §7, can then catch and act on — e.g. send a Slack alert).

**Segment/type classifier branch inside a LOOP** — same shape, but each case label matches a different value each iteration (e.g. `"valueList = /^BEG\*/"`, `"valueList = /^N1\*BY/"`, `"$default"`), used to dispatch different handling per array element type.

### `node.ndf` step hints for BRANCH/EXPRESSION/EXIT can be minimal

Unlike INVOKE/TRANSFORM/LOOP (which need `nodePathNew`/`nodeIndex`/`lineNumberForDebug` etc.), a working, platform-generated sample showed BRANCH/EXPRESSION/EXIT hints can be just:

```xml
<record name="/0/2" javaclass="com.wm.util.Values">
  <value name="name">BRANCH</value>
  <value name="itemType">CONTROLS</value>
</record>
```

No extra fields needed. Use this minimal shape for BRANCH, EXPRESSION (each case), and EXIT hints; use the fuller shape (below) for INVOKE, TRANSFORM, and LOOP.

## 5. `stepNodeHints` — required for the flow to open cleanly in the browser editor

This is the part that, if wrong, produces **"Associated action is not chosen"** when a step is opened — the flow.xml itself can be syntactically perfect and still show this error if the matching `node.ndf` hint is missing or wrong.

Every step in `flow.xml`, in document order, needs a corresponding entry in `stepNodeHints`, keyed by a slash-path matching its position (e.g. `/0`, `/0/0`, `/0/1/0` for nesting). **A container step itself (SEQUENCE, the outer TRY/CATCH) gets a `<null name="/N"/>` placeholder**, not a real record — only its children get real hint records. **Any step with no nested children still needs a `<null name="/N/0"/>` placeholder** for its would-be first child slot.

### INVOKE hint (SERVICES) — this is the one that fixes "Associated action is not chosen"

```xml
<record name="/0/0" javaclass="com.wm.util.Values">
  <null name="name"/>
  <value name="itemType">SERVICES</value>
  <record name="serviceInfo" javaclass="com.wm.util.Values">
    <value name="groupDisplayName">String</value>
    <value name="groupName">string</value>
    <null name="services"/>
  </record>
  <record name="serviceName" javaclass="com.wm.util.Values">
    <value name="displayName">tokenize</value>
    <value name="serviceName">pub.string:tokenize</value>
    <value name="transformerSupport">true</value>
  </record>
  <number name="ui_step_index" type="Long">1</number>
  <value name="nodePathNew">0.0</value>
  <number name="nodeIndex" type="Long">0</number>
  <number name="lineNumberForDebug" type="Long">1</number>
  <null name="mapSet"/>
  <null name="outputMapSet"/>
</record>
```

Both `serviceInfo` (folder grouping, cosmetic but must be present) **and** `serviceName` (the actual real service name) are required — missing either produces the error.

### TRANSFORM hint (CONTROLS)

```xml
<record name="/0/1" javaclass="com.wm.util.Values">
  <value name="itemType">CONTROLS</value>
  <value name="name">TRANSFORM</value>
  <number name="ui_step_index" type="Long">2</number>
  <value name="nodePathNew">0.1</value>
  <number name="nodeIndex" type="Long">1</number>
  <number name="lineNumberForDebug" type="Long">2</number>
  <array name="map" type="object" depth="1" javaclass="java.lang.Object"></array>
  <array name="mapSet" type="object" depth="1" javaclass="java.lang.Object"></array>
  <array name="mapDelete" type="object" depth="1" javaclass="java.lang.Object"></array>
  <array name="transformer" type="object" depth="1" javaclass="java.lang.Object"></array>
  <array name="lookupTransformer" type="object" depth="1" javaclass="java.lang.Object"></array>
</record>
```

### LOOP hint (CONTROLS)

```xml
<record name="/0/3" javaclass="com.wm.util.Values">
  <value name="name">REPEATINPUTOUTPUT</value>
  <value name="itemType">CONTROLS</value>
  <record name="in_array" javaclass="com.wm.util.Values">
    <value name="name">/valueList</value>
    <value name="field_name">valueList</value>
    <value name="type">string</value>       <!-- or "record" for a document-list -->
    <value name="dim">1</value>
    <Boolean name="isLink">false</Boolean>
    <null name="wrapper_type"/>
  </record>
  <record name="out_array" javaclass="com.wm.util.Values"><!-- same shape, omit if no OUT-ARRAY --></record>
  <value name="nodePathNew">0.3</value>
  <number name="nodeIndex" type="Long">3</number>
  <number name="lineNumberForDebug" type="Long">4</number>
</record>
```

### Connector/adapter APPS hint (e.g. a Slack action) — much heavier

Cloud connector steps (Slack, and any `WmXxxProvider`-backed operation) need a fuller hint than a plain INVOKE:

```xml
<record name="/1/0" javaclass="com.wm.util.Values">
  <value name="name">Slack®</value>
  <value name="itemType">APPS</value>
  <record name="operation" javaclass="com.wm.util.Values">
    <value name="connectorID">com.webmethods.cloudstreams.slack_v2</value>
    <value name="isCustom">false</value>
    <value name="isDynamicOperation">false</value>
    <value name="isImportedOperation">true</value>
    <value name="nsName">wm.com_webmethods_cloudstreams_slack_v2.operations:ChatPostMessage</value>
    <value name="operationName">ChatPostMessage</value>
    <value name="operationType">actions</value>
    <value name="packageName">WmSlackProvider</value>
    <value name="projectName">project.yourproject</value>
  </record>
  <value name="id">com.webmethods.cloudstreams.slack_v2</value>
  <value name="providerName">WmSlackProvider</value>
  <record name="connection" javaclass="com.wm.util.Values">
    <value name="connectionName">YourConnectionAlias</value>
    <value name="connectionType">sagcloud</value>
    <null name="description"/>
  </record>
  <value name="connectionNSName">project.yourproject.WmSlackProvider.com_webmethods_cloudstreams_slack_v2.connections:YourConnectionAlias</value>
  <!-- plus packageName, description, extensionName, certified, registry, category, registeredDate, installed, latest, and empty map/mapDelete/transformer/lookupTransformer/inputMap/outputMap arrays -->
</record>
```

**Do not hand-author the connector *connection* asset itself** (the OAuth/credentials node under `ns/project/<pkg>/<Provider>/<connector>/connections/<Alias>/node.ndf`) — that must be created once through the browser UI (it holds `passmanHandles` references to securely-stored secrets, not plaintext). Once it exists, you can freely reference its `connectionNSName` from hand-written flow steps like above.

Also add the flow's own `node_hints` an `appList` entry when it uses a connector:

```xml
<array name="appList" type="record" depth="1">
  <record javaclass="com.wm.util.Values">
    <value name="providerName">WmSlackProvider</value>
    <value name="id">com.webmethods.cloudstreams.slack_v2</value>
  </record>
</array>
```

## 6. MAPTARGET/MAPSOURCE field declaration conventions

Every field entry in a `Values`/`rec_fields` block uses one of these shapes, depending on its role:

| Role | `node_type` | Notes |
|---|---|---|
| The flow's own `sig_in` field, or a service's own fixed input parameter (e.g. `tokenizeInput`'s `inString`) | `field` | Include the `node_hints` (`field_largerEditor`/`field_password`/`field_usereditable`) + `node_comment` sub-block. |
| Any other pipeline-local scalar (created mid-flow, or a `sig_out` field being set) | `unknown` | No `node_hints`/`node_comment` needed. |
| An array field (string list or record list) | `field` (or `unknown` if pipeline-local) | Add `field_dim=1` and `is_soap_array_encoding_used=false`; omit `node_hints`. |
| A nested record/document field (e.g. `document`) | `record` for `field_type`, `node_type=record` | `rec_fields` can be left as an **empty array** — the Designer does not require it to be fully expanded for the mapping to work. |

The wrapping outer `Values`/`record name="xml"` block itself gets a `field_name` set **only** when it represents a service's own multi-field input group (e.g. `field_name=tokenizeInput`, matching the `<serviceName>Input` convention) — omit it when the block just represents "current pipeline state" (a MAPSOURCE, or a TRANSFORM's own target).

## 7. Composable patterns that come up constantly

**String accumulator across a loop** (building an XML/EDI/delimited string one element at a time):
1. TRANSFORM before the loop: `MAPSET` the accumulator field to `""` or an opening literal.
2. Inside the loop: `MAPSET` the same field to `%accumulatorField%<literal or %currentItem%>`, `VARIABLES="true"`.
3. TRANSFORM after the loop: `MAPSET` to append a closing literal.

**Try/Catch/Notify-on-failure**: not a nested container — it's **two sibling `SEQUENCE` steps**, one `FORM="TRY"` and the next `FORM="CATCH"`, both direct children of the same parent. The runtime pairs them by adjacency. Put your validation `BRANCH` (§4) with an `EXIT ... SIGNAL="FAILURE"` inside the TRY sequence, and a notification step (e.g. Slack) inside the CATCH sequence, using `%originalInputField%` to include context in the failure message.

**Counting/summing without full arithmetic confidence**: `pub.list:sizeOfList` (`fromList` → `size`) is reliable for counting array elements. For anything beyond +1 (e.g. real sums), verify the exact service (`pub.math:addInts` takes `num1`/`num2` → `value` — **not** `value1`/`value2`, a wrong guess we made once) before trusting it; if in doubt, use a clearly-flagged static placeholder and tell the user it's an approximation rather than silently guessing.

## 8. Git workflow specifics

- The platform (IWHI) itself auto-commits to the same remote whenever a DAFS is saved/edited in the browser, or when it auto-registers a connector/adapter dependency. Expect **frequent remote-side commits appearing between your own local commits.**
- Before every local commit: `git fetch origin` then `git -c user.name="..." -c user.email="..." rebase origin/main` (or `git stash` first if there are uncommitted changes blocking the rebase, then `git stash pop` after). This avoids "Updates were rejected... remote contains work you do not have."
- Stale `.git/*.lock` files periodically block commits (`index.lock`/`HEAD.lock`) — safe to `rm -f .git/*.lock` before each commit attempt if a previous operation didn't clean up.
- **Never hand-edit `config/scaffolding/*.yml`** to add package dependencies — if the platform also auto-writes to it (which it will, the first time you exercise a connector step in the browser), you'll get a "Failed pulling down remote changes: Checkout conflict" on that file. Let the platform own that file; just make sure the user actually opens/exercises the relevant step once in the browser so the platform registers the dependency itself.
- You will not have push credentials in a sandboxed environment — do local `git add`/`commit`/`fetch`/`rebase` only, and tell the user to run `git push origin main` themselves.

## 9. Error message → root cause cheat sheet

| Error seen in the browser | Root cause | Fix |
|---|---|---|
| "Associated action is not chosen" | `stepNodeHints` entry for that INVOKE is missing, or missing `serviceInfo`/`serviceName` | Add the full INVOKE hint shape (§5) |
| "The mappings listed are not displayed correctly... service metadata modification" | The hand-written `MAPTARGET` field declaration for an INVOKE's input doesn't match the *real* service's current signature (wrong `field_type`, e.g. declaring an Object-typed input as `string`; or no `MAPTARGET`/`MAPSOURCE` at all) | Look up the real service's documented signature and match `field_type` exactly (e.g. `pub.json:jsonToDocument`'s `jsonData` is `object`, not `string`); always include full `MAPTARGET`/`MAPSOURCE` |
| Runtime `Missing Parameter: <field>` despite a MAPSET that looks right | The `MAPSET FIELD=` (or a `MAPCOPY` `TO=`) is missing its `;X;Y` type suffix, so the value silently never lands in the pipeline | Add the suffix (§4) |
| A field computed from an array comes back as `0` or empty (e.g. a count) | The `MAPCOPY FROM=` referencing that array used the wrong type code — record arrays need `;2;0`, not `;1;0` | Use `;2;0` for any record/document-typed source, `;1;0` for string |
| Project tile shows "Updated: Invalid date" | `node.ndf`'s `node_hints` is missing `createdDate`/`lastModifiedDate` | Add both as epoch-millisecond integers |
| "Failed pulling down remote changes: Checkout conflict with files: config/scaffolding/..." | You hand-edited a platform-managed file that the platform also wrote to independently | Revert your edit to that file; let the platform's own version stand; have the user exercise the connector step in the browser once so the platform re-registers correctly |
| "[service] must specify schema or template" (seen with `wm.b2b.edi:convertToValues`) | Either the referenced schema/document-type name doesn't exist on the tenant, or a required B2B package dependency isn't registered | Confirm the package dependency is registered (§8); if it persists, this may require a real schema asset that doesn't exist yet — consider a simpler string-based workaround instead of the B2B connector (see §10) |

## 10. When a built-in service's real signature is uncertain

We hit this repeatedly with genuinely-real but unfamiliar services (`pub.list:appendToStringList`, `pub.math:addInts`, `pub.json:jsonToDocument`). Before wiring one up:

1. Search for the service's official Software AG / IBM webMethods documentation page (`documentation.softwareag.com` or `docs.webmethods.io`) and confirm the **exact** input/output field names and types — don't guess based on naming conventions from a different but similar-sounding service.
2. If docs are unreachable, look for the pattern in an existing real sample in the project's own git history (a service the user or the platform generated through the UI) — these are ground truth for exact XML encoding, more reliable than documentation prose.
3. If you must guess, say so explicitly to the user, and expect one round of live-test correction — this is normal, not a failure. Every genuinely new service or step-type combination in this codebase needed at least one round of "tried it, got this exact error, fixed this one field."
4. When something doesn't have a real generatable equivalent yet in the tenant (e.g. a JDBC Adapter connection, a B2B EDI schema asset), don't fabricate its on-disk encoding blind — it typically requires an interactive, wizard-driven setup in the browser that produces credentials/complex generated code we can't safely reproduce from documentation alone. Tell the user to create that one asset via the UI once, then continue building everything else in code from there.

## 11. General workflow that worked

1. Build the flow's logic as a Python generator script (not hand-typed XML) once it's more than ~5 steps — define small reusable render functions for each step type (INVOKE, TRANSFORM, LOOP, BRANCH case), track the growing list of "fields visible in the pipeline at this point" as you go, and generate both `flow.xml` and the matching `node.ndf` `stepNodeHints` from the same step list so they can't drift out of sync.
2. Validate every generated file with `xml.dom.minidom.parse()` before writing it into the repo — this catches encoding bugs (unescaped `<`/`>`/`&`, mismatched tags) before the user ever opens it in the browser.
3. Commit locally, tell the user to push, and wait for their test result (a screenshot of the error dialog, or the run output) — don't guess at a second fix before seeing the first error.
4. Treat every error message as a direct pointer to one specific, fixable line — the cheat sheet in §9 covers the recurring categories.

---

# Part II — Reference implementations & EDI cookbook

> **Added after building five production-style EDI/EDIFACT flow services entirely on-disk in `FlowServices_gitProject`.** This part turns the Part I theory into copy-paste-able recipes. Everything here follows the Part I rules; where this part and Part I disagree, Part I wins.
>
> **Verification status (read this):** the five services below were authored on-disk and their *logic* was simulated in Python (segment counts, branch outcomes, JSON/EDI shape). They have **not yet been through the in-browser open/save/run loop**, so treat the newer hint recipes — especially TRY/CATCH containers and BRANCH-case placeholders (§15) — as *best-known* and expect the usual **one round of live-test correction** (Part I §9, §11). The built-in service *signatures* in §13 are the ones actually wired up; still confirm any unfamiliar one against docs on first run (Part I §10).

## 12. On-disk path correction (supersedes the illustration in §2)

In this repo the service folder is keyed by the **project short name**, not the package name:

```
ns/project/<project_short_name>/integrations/<ServiceName>/
        └── e.g.  ns/project/flowservices_git/integrations/JsonToEdifactInvoice/
                     ├── node.ndf
                     └── flow.xml
```

Here the package is `FlowServices_gitProject` but the namespace folder is `flowservices_git` (see `ns/project/flowservices_git/node.idf` → `node_nsName = project.flowservices_git`). Individual services use **`node.ndf`**; the `node.idf` files exist only at the `project` / `project.<short>` / `.../integrations` container levels and are auto-generated — don't hand-author `node.idf` for a service.

## 13. Built-in services used across these builds (wire-up reference)

All are stock webMethods `pub.*` services (no connector/schema asset needed). Field names below are what the flows actually map to.

| Service | Inputs (field -> type) | Outputs | Notes |
|---|---|---|---|
| `pub.json:jsonToDocument` | `jsonData` -> **object** | `document` (record) | Input is **object**, not string (Part I §9). Parsed doc lands as `document`; reference nested values as `%document/a/b%`, arrays as record lists (`;2;0`). |
| `pub.list:sizeOfList` | `fromList` -> object/record list | `size` (string) | Count array elements reliably (Part I §7). Source a **record array** with `;2;0`. |
| `pub.math:addInts` | `num1`, `num2` -> **string** | `value` (string) | Numbers are String-typed. Used as a running counter inside a loop (init via MAPSET, `+N` each pass). |
| `pub.string:tokenize` | `inString` -> string; `delim` -> string; `useDelimsAsSet` -> string `"true"` | `valueList` (string[]) | `delim="~"` splits EDI into segments. `delim="*~"` + `useDelimsAsSet="true"` splits on **both** `*` and `~` into one flat token list. Empty tokens are dropped — beware for positional parsing (§14.4). |
| `pub.xml:xmlStringToXMLNode` | `xmldata` -> string | `node` (object) | Doubles as a **well-formedness validator**: it throws on malformed XML, so inside a TRY it routes control to the CATCH. |
| `pub.flow:getLastError` | *(none required)* | `lastError` (record) | In a CATCH, gives `%lastError/error%` (the message), plus `errorType`, `service`, `time`, `pipeline`. |
| `pub.flow:debugLog` | `message`, `function`, `level` -> string | *(none)* | The "show a log message" step. `level` e.g. `"Error"`; `function` is a free label (put the service name). |

## 14. The four reusable patterns (each backed by a shipped service)

### 14.1 JSON/document -> delimited-string **builder** (string accumulator)
*Reference: `JsonToEdifactInvoice` (JSON invoice -> UN/EDIFACT D96A INVOIC).*

Skeleton: `jsonToDocument` -> `sizeOfList` (line count) -> TRANSFORM (init accumulator + header) -> LOOP over the line array (append per-line segments; `addInts` to grow a segment counter) -> TRANSFORM (append summary/trailer using the counter).

Key moves:
- **Header in one MAPSET** (`VARIABLES="true"`) concatenating literals + `%document/...%` fields, each segment ended with the EDIFACT terminator `'` and a newline for readability.
- **Per-line append inside the LOOP**: `MAPSET /edidata;1;0 = %edidata%LIN+...'` — the accumulator persists across iterations because it is **not** part of IN-ARRAY (Part I §4).
- **Trailer counts without multiplication**: init `segCount` to the fixed-segment total via a literal MAPSET, then `addInts(segCount, "5")` once per line — final `UNT` count = `fixed + 5*lineCount`, using only the verified `addInts`. `CNT`/line count comes from `sizeOfList`.
- Loop over a **record array**: `IN-ARRAY="/document/invoice/lines"`; inside, fields are at the *same nested path* (`%document/invoice/lines/quantity%`).

### 14.2 EDI -> XML with validation + **Try/Catch + log**
*Reference: `Edi850ToXml`, `Edi855ToXml` (X12 -> generic XML).*

Structure = **two sibling SEQUENCEs** under FLOW, paired by adjacency (Part I §7):

```
FLOW
  SEQUENCE FORM="TRY"  EXIT-ON="FAILURE"
    BRANCH  (is-it-the-right-doc? -> EXIT FAILURE on $default)     <- one per rule
    INVOKE pub.string:tokenize   (edidata by "~" -> segmentList)
    MAP    (STANDALONE)          xmldata = "<EDIxxx>"
    LOOP   IN-ARRAY="/segmentList"
        MAP  xmldata = %xmldata%<segment>%segmentList%</segment>
    MAP    xmldata = %xmldata%</EDIxxx>,  status="VALID_xxx"
    INVOKE pub.xml:xmlStringToXMLNode  (throws if not well-formed)
  SEQUENCE FORM="CATCH" EXIT-ON="DONE"
    INVOKE pub.flow:getLastError  -> lastError
    MAP    logMessage="...%lastError/error%", status="INVALID_xxx", xmldata=""
    INVOKE pub.flow:debugLog      (message=%logMessage%, function, level="Error")
```

Validation BRANCH (regex label + throw on the default case):

```xml
<BRANCH LABELEXPRESSIONS="true">
  <COMMENT></COMMENT>
<SEQUENCE NAME="edidata = /ST\*850/" EXIT-ON="FAILURE"><COMMENT></COMMENT></SEQUENCE>
<SEQUENCE NAME="$default" EXIT-ON="FAILURE">
  <COMMENT></COMMENT>
<EXIT NAME="EXIT" FROM="$parent" SIGNAL="FAILURE" FAILURE-MESSAGE="Not a valid 850 (ST*850 not found).">
  <COMMENT></COMMENT>
</EXIT>
</SEQUENCE>
</BRANCH>
```

- Stack **multiple BRANCHes** for multi-rule validation (e.g. `Edi855ToXml` checks `/ST\*855/` **and** `/BAK\*/`, each its own BRANCH+EXIT). Any EXIT FAILURE — or a throw from `xmlStringToXMLNode` — drops into the CATCH.
- The regex is a substring match; escape the X12 element separator as `\*` in the label.
- **Escaping caveat:** the generic `<segment>%raw%</segment>` wrap does not XML-escape `&`/`<`/`>`. Real X12 rarely contains them; if present, the XML is malformed and `xmlStringToXMLNode` correctly kicks it to the CATCH. For strict output, escape upstream or build element-level XML (see "going further" below).

### 14.3 Scalars -> delimited-string **builder** (single step)
*Reference: `Edi997Ack` (build an X12 997 Functional Acknowledgment).*

When there's no repetition, the whole thing is **one STANDALONE MAP with one MAPSET** (`VARIABLES="true"`) concatenating literals + `%scalarField%`. No INVOKE, no LOOP — the lowest-risk service shape. Fixed trailer counts (e.g. `SE*6` for a 6-segment 997) are just literals. Drive small variations (accept vs reject -> `AK9*...*<numberAccepted>`) from an input field rather than a BRANCH when you can.

### 14.4 EDI -> JSON via **flat tokenize + positional substitution**
*Reference: `Edi997ToJson`.*

For a **small, fixed-layout** message: `tokenize(delim="*~", useDelimsAsSet="true")` -> `tokens[]`, then one MAPSET builds JSON with `%tokens[N]%` by element position:

```
{"functionalGroup":"%tokens[4]%","acknowledgedTransactionSet":"%tokens[7]%",
 "groupAckCode":"%tokens[12]%","numberAccepted":"%tokens[15]%"}
```

- Array-index substitution `%field[N]%` is a real MAPSET feature (Part I §4).
- **Fragility warning:** positions shift if optional segments appear (997: `AK3/AK4` error detail, repeated `AK2` loops) or if there are empty elements (tokenize drops empty tokens). Use this only for messages you control end-to-end (e.g. round-tripping your own generator's output). For arbitrary partner data, use the **segment-classifier-loop** instead: LOOP over segments, BRANCH on each segment's prefix (`valueList = /^AK1\*/`, `/^AK2\*/`, `$default`), tokenize the matched segment by `*`, and pull named fields (Part I §4, "segment/type classifier branch inside a LOOP").

**Going further — element-level XML/JSON:** the generic `<segment>` wrap (§14.2) is one loop. To emit `<segment id="ST"><element>997</element>...>`, nest a second LOOP over `tokenize(segment,"*")` inside the segment loop and read the id as `%elemList[0]%`. More faithful, deeper nesting, more hint bookkeeping — build it via the generator (§11) and expect a correction round.

## 15. `stepNodeHints` recipe, consolidated (all constructs used here)

One entry per `flow.xml` step in document order; keys are slash-paths mirroring nesting. Applied rules from these builds:

| Construct | Hint at its own path | Child placeholder(s) |
|---|---|---|
| **INVOKE** | full **SERVICES** record incl. `serviceInfo` *and* `serviceName` (Part I §5) | `<null name="/N/0"/>` (it's a leaf) |
| **TRANSFORM** (STANDALONE MAP) | **CONTROLS** record, `name=TRANSFORM` (Part I §5) | `<null name="/N/0"/>` |
| **LOOP** | **CONTROLS** record, `name=REPEAT` + `in_array` (`type=string` for a string list, `record` for a doc list) | its real child steps follow (no null) |
| **SEQUENCE** used as TRY/CATCH wrapper | `<null name="/N"/>` (container -> placeholder, Part I §5) | its real child steps follow |
| **BRANCH** | minimal record: `name=BRANCH`, `itemType=CONTROLS` (Part I §4) | its case children follow |
| **BRANCH case SEQUENCE** (regex label or `$default`) | `<null name="/N"/>` | if the case body is empty, still `<null name="/N/0"/>`; if it holds an EXIT, that EXIT gets its own record |
| **EXIT** | minimal record: `name=EXIT`, `itemType=CONTROLS` | `<null name="/N/0"/>` |

Every leaf (INVOKE/TRANSFORM/EXIT, and an empty SEQUENCE) needs the `/N/0` null placeholder for its would-be first child. Cosmetic fields (`ui_step_index`, `nodeIndex`=last path component, `nodePathNew`=dotted path, `lineNumberForDebug`) are filled sequentially and don't affect runtime.

TRY/CATCH concretely: `SEQUENCE FORM="TRY" EXIT-ON="FAILURE"` then `SEQUENCE FORM="CATCH" EXIT-ON="DONE"`, both direct FLOW children; hints `<null name="/0"/>` and `<null name="/1"/>` with their children as `/0/0...` and `/1/0...`.

## 16. EDI message quick reference (the ones built here)

**UN/EDIFACT D96A INVOIC** (built by `JsonToEdifactInvoice`): `UNA` (delims) - `UNB` interchange - `UNH+...+INVOIC:D:96A:UN` - `BGM+380` invoice no. - `DTM+137` date - `NAD+SU`/`NAD+BY` supplier/buyer - `CUX+2` currency - per line `LIN`/`IMD`/`QTY+47`/`MOA+203`/`PRI+AAA` - `UNS+S` - `MOA+79`(net)/`MOA+124`(tax)/`MOA+9`(payable) - `TAX+7+VAT` - `CNT+2` line count - `UNT+<seg count>+1` - `UNZ`. Terminator `'`, element `+`, component `:`.

**X12 850 Purchase Order** (parsed by `Edi850ToXml`): `ISA`/`GS` envelope - `ST*850` - `BEG` beginning - `N1` parties - `PO1` line items - `CTT` totals - `SE`/`GE`/`IEA`. Validated by presence of `ST*850`.

**X12 855 PO Acknowledgment** (parsed by `Edi855ToXml`): `ST*855` - **`BAK`** beginning (required — the 855's tell) - `PO1` - `ACK` line ack - `CTT` - `SE`. Validated by `ST*855` **and** `BAK`.

**X12 997 Functional Acknowledgment** (built by `Edi997Ack`, parsed by `Edi997ToJson`): `ST*997` - `AK1*<fg>*<groupCtrl>` group response - `AK2*<ts>*<tsCtrl>` transaction-set response - `AK5*<A|E|R>` TS result - `AK9*<A|E|R>*<#included>*<#received>*<#accepted>` group result - `SE`. Delimiters `*` element / `~` segment.

## 17. Service catalog — `ns/project/flowservices_git/integrations/`

| Service | Direction | In -> Out | What it does |
|---|---|---|---|
| `JsonToEdifactInvoice` | build | `jsonData` (JSON string) -> `edidata` | Invoice JSON -> EDIFACT **D96A INVOIC**; loops line items, computes `UNT`/`CNT`. |
| `Edi850ToXml` | parse+validate | `edidata` (X12 850) -> `xmldata`, `status` | Validates `ST*850`, wraps segments as XML, checks well-formedness; TRY/CATCH -> `debugLog` on failure. |
| `Edi855ToXml` | parse+validate | `edidata` (X12 855) -> `xmldata`, `status` | Like 850 but validates `ST*855` **and** `BAK`. |
| `Edi997Ack` | build | 7 scalar acks -> `edidata` | Builds a 997 transaction set (`ST/AK1/AK2/AK5/AK9/SE`) from `ackCode`, control numbers, `numberAccepted`. |
| `Edi997ToJson` | parse | `edidata` (X12 997) -> `jsonData` | Flat-tokenizes and maps element positions into named JSON (positional; standard single-group 997). |

Sample payloads live at repo root: `sample_invoice.json`, `sample_850.edi`, `sample_855.edi`, `sample_997.edi`, `sample_997_input.json`.

## 18. Workflow reminders specific to this environment

- Each service was generated by a small **Python generator per service** (Part I §11) that emits `flow.xml` and `node.ndf` from one step list and validates both with `xml.dom.minidom` before writing — this is what keeps the two files' step order and hints from drifting. Reuse that approach for anything over ~5 steps.
- **Delimiter assumption:** the X12 parsers assume `~` segment / `*` element separators. Real interchanges declare their own in `ISA`; make the delimiters inputs (or read them from `ISA`) if you must accept arbitrary partners.
- Commit locally, then **you push** (`git push origin main`) — the sandbox has no push credentials (Part I §8). If a stale `.git/*.lock` blocks a commit and can't be removed, it's a delete-permission gate on the mounted folder, not a git problem; clear the lock and retry.
