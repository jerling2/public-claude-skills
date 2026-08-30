---
name: onboard-figma
description: Navigate a Figma file and pull its JSON out over the REST API. Use when a figma.com URL appears, when someone asks what is inside a Figma file or how it is structured, when extracting node JSON for a handoff or for generating code from a design, or when a question touches Figma seat types, plans, or REST API rate limits.
---

# Onboard: Figma

Two jobs. **Read a file well enough to discuss it with the designer who built it**, and **get that
file's JSON onto disk.** The second is metered — on the Starter plan at six calls a *month* — so the
request is planned before it is sent, and the response is written down the first time.

## References

Read the one that matches the job. Don't read all three by reflex.

| File | Read it when |
| --- | --- |
| `references/design-url-decoding.md` | Turning a URL into a file key and a node id — and any time the URL isn't the plain `/design/<key>/<name>?node-id=` shape. Branch URLs especially: they fail *silently*, returning the wrong file with a `200` |
| `references/hierarchy-vernacular.md` | Reading the JSON that came back, or talking to a designer. Node types, nesting rules, frame vs group vs section, the words that mean two things |
| `references/rest-api.md` | Budgeting calls. The tier tables, the seat × plan grid, and why Starter is the expensive plan |

## What you need from the user

Four inputs. Three gate a fetch; one gates rate-limit talk. **Ask for every missing one in a single
message** — never a question per turn, and never ask for something a link already carries.

### 1. The design URL — required

Everything addressing the file comes out of it. Copy it from the browser bar or from **Share →
Copy link**; the file key survives renames and folder moves, so an old link still works.

Decode it yourself rather than asking the user for parts:

- **file key** — path position 4, e.g. `abttA1F3GxZdZlkI9t5s99`. Case-sensitive; read to the next
  `/` rather than assuming a length.
- **node id** — the `node-id` query parameter, e.g. `32-9`. **Swap `-` for `:`** before sending it:
  the API wants `32:9`. Applies to the opaque in-instance form too (`I422-10713;1082-2236` →
  `I422:10713;1082:2236`).
- **branch** — if path position 5 reads `branch`, the key you want is position **6**, not 4.
- Strip the `t=` sharing token before storing or comparing the URL.

If there's no `node-id`, the link names a file, not a layer — say so, and expect to enumerate pages
first. If the `node-id` turns out to address a `CANVAS`, it's a page, not a screen.

### 2. Authentication — required

**Assume a personal access token (PAT) unless the user says otherwise.** It's what Figma recommends
for "individual use, such as scripts or local tooling against your own Figma account."

```
X-Figma-Token: <token>
```

The user mints one at **account menu → Settings → Security → Personal access tokens → Generate new
token**, setting an expiration and scopes in the modal. It's shown once. The scopes decide which
endpoints the token can reach:

| Scope | Buys |
| --- | --- |
| `file_content:read` | `GET file`, `GET file nodes` — the design data |
| `file_metadata:read` | `GET file metadata` |
| `file_dev_resources:read` | Dev resources on the file |

The other two methods, and when they're actually the answer:

- **OAuth 2.0** — `Authorization: Bearer <token>`. For an app acting on behalf of *other* users.
  Not the answer for local tooling; don't propose it for a one-off extraction.
- **Plan access token** — also `X-Figma-Token`, but scoped to an Organization or Enterprise plan
  rather than a person, created by an org admin with MFA. Up to a year's expiry, supports resource
  allowlisting. The right answer for automation that must outlive one employee — PATs cap at 90
  days and carry that person's access.

Handling: ask for it as an environment variable (`FIGMA_TOKEN`) and reference it as `$FIGMA_TOKEN`.
Never echo it, never interpolate it into a command you print back, never write it into the output
file, never commit it. If the user pastes it into chat anyway, use it and tell them to rotate it.

### 3. Destination — required, but has a default

Where the JSON lands. **If the user didn't say, write it to your scratchpad directory** and tell
them the path. Don't stop to ask.

This isn't bookkeeping. The response *is* the rate budget: write the raw body to disk on the first
call and read it from there afterwards, so re-reading the design costs nothing.

### 4. Seat type and plan — optional, but it's what makes the usage report useful

**No endpoint reports the caller's seat.** Ask for it when a question turns on how many calls are
available or what an upgrade would buy — and ask once before the first fetch of a session, because
without it the mandatory usage report below can only say what was spent, never what remains. Don't
block a fetch waiting for the answer; report the degraded version instead.

- **Seat**: Full, Dev, Collab, or View. Many people don't know theirs; an admin reads it under
  **Admin → People → Seat type**.
- **Plan of the file** — Starter, Professional, Organization, or Enterprise. Ask for *the plan the
  file lives in*, not the plan of their account. The limit follows the file: a Starter-plan file is
  metered at 6 Tier 1 calls a month even for a Full seat holder on an Enterprise plan elsewhere.

A `429` answers both after the fact — it carries `X-Figma-Plan-Tier` for the requested resource,
plus `Retry-After` and `X-Figma-Upgrade-Link`.

## Extracting the JSON

Base URL `https://api.figma.com`.

**Confirm the file with a cheap call before spending an expensive one.** `GET file metadata` is
Tier 3 (10/min even on Starter); the endpoints that return design data are Tier 1 (6/**month** on
Starter). Depth and `ids` shrink the *payload*, not the rate cost — a probe costs the same as the
real thing, so there is exactly one Tier 1 call to spend and it has to be the right one.

```bash
# Tier 3 — confirms the key resolves, and to which file
curl -s -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/files/$FILE_KEY/meta"
```

Then pick one Tier 1 endpoint:

| Endpoint | Returns | Use when |
| --- | --- | --- |
| `GET /v1/files/:key/nodes?ids=32:9` | Just those subtrees | **The default.** The URL named a node — take that node and nothing else |
| `GET /v1/files/:key?depth=1` | Pages only | No `node-id`, and you need to see what pages exist before choosing |
| `GET /v1/files/:key?depth=2` | Pages plus each page's top-level frames | Surveying the screens in a file |
| `GET /v1/files/:key` | The entire document tree | Rarely. Whole-file audits only — it is enormous and it is the same single call you could have spent precisely |

Parameters worth setting deliberately:

- `ids` — comma-separated, colon form: `ids=1:2,1:3`.
- `depth` — a positive integer, how far down the tree to traverse. Omit for everything below the
  requested nodes.
- `geometry=paths` — vector path data. Only when someone needs the geometry; it inflates the
  response hard.
- `version` — pins a version id, so a re-fetch reproduces rather than picking up edits.
- `branch_data=true` — on `GET file`, returns `mainFileKey`, which is how you confirm after the
  fact that a key was a branch key.

Write the body to the destination unparsed, then work from the file:

```bash
curl -s -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/files/$FILE_KEY/nodes?ids=$NODE_ID" \
  -o "$DEST/$FILE_KEY-$NODE_ID.json"
```

Two adjacent endpoints worth knowing: `GET /v1/images/:key` renders nodes to png/jpg/svg/pdf —
Tier 1, so it competes with the fetch above for the same budget — and `GET /v1/files/:key/images`
returns the URLs of image fills already in the file, which is Tier 2 and comparatively cheap.

## Report remaining usage after every call — mandatory

**After every API call, tell the user what it cost and what is left.** Not on request, not only
when it's tight. Every call, unprompted.

Figma gives you nothing to build this on. There is no `X-RateLimit-Remaining`, no quota field, no
counter on a successful response — the rate headers (`Retry-After`, `X-Figma-Plan-Tier`,
`X-Figma-Rate-Limit-Type`, `X-Figma-Upgrade-Link`) appear **only on a `429`**, which is to say the
API tells you your budget exactly once: at the moment you've already blown it. For an API that
meters some plans at six calls a *month*, that is a poor design, and working around it is on you.

So keep the count yourself. Append one line per call to a ledger beside the output:

```bash
printf '%s\t%s\t%s\t%s\t%s\n' "$(date -u +%FT%TZ)" "tier1" "GET /v1/files/:key/nodes" \
  "$FILE_KEY" "$STATUS" >> "$DEST/figma-api-usage.tsv"
```

Then report, in the message that delivers the result:

- **What tier that call spent** — Tier 1 for `GET file`, `GET file nodes`, `GET image`; Tier 2 or 3
  for everything else. Tiers have separate budgets; a Tier 3 call costs nothing from Tier 1.
- **How many of each tier have been spent** against this file, per the ledger.
- **What remains** — the seat × plan cell from `references/rest-api.md` minus the ledger count. If
  the seat or the file's plan is unknown, say the count spent and name the missing input rather
  than guessing a ceiling.
- **The reset shape**, when it matters: Tier 1 on Starter is a *monthly* quota, not a per-minute
  one, so a spent budget is gone until the month turns. Elsewhere the limits are per-minute and
  refill continuously — Figma runs a leaky bucket, so "remaining" drifts back up rather than
  resetting on a clean boundary.

State the arithmetic honestly. The ledger counts only calls *you* made — the user's own scripts,
plugins, other tools, and other sessions draw on the same budget invisibly, and View and Collab
figures are ceilings Figma may lower under load ("the actual limit may be lower"). So the number
you report is a floor on what's been used and an optimistic ceiling on what's left. Say that once
per session; don't repeat the caveat on every call.

A worked report, in one line:

> Spent 1 Tier 1 call (`GET file nodes`). 2 of 6 Tier 1 calls used this month on this Starter file
> — 4 left, and only by my count.

If the seat and plan were never supplied:

> Spent 1 Tier 1 call (`GET file nodes`); 2 made this session. I can't say what's left without the
> seat type and the plan the file lives in — Figma doesn't return either.

## Reading what came back

The JSON is the layers panel. `document` → pages → everything else; the first two levels are fixed
and the rest nests arbitrarily, often ten deep. Array order is z-order, frontmost first.

Report it in the designer's vocabulary, not the JSON's: a `CANVAS` is a **page**, a top-level
`FRAME` is a **screen**, an `INSTANCE` is a copy of a component, a `COMPONENT_SET` holds the
variants. `references/hierarchy-vernacular.md` has the mapping and the traps — the frame/group/
section distinction, and the words that carry two meanings.

## When it fails

| Status | Means | Do |
| --- | --- | --- |
| `403` | The file exists; this token can't have it | Check the scope first (`file_content:read` for design data), then access to the file itself. A PAT that used to work has likely hit its 90-day expiry |
| `404` | No such file — or the key is wrong | Re-decode the URL. Most often position 4 of a branch URL, or a truncated key |
| `400` | Malformed parameters, or too large a request | Check the node id is in colon form. If the tree is huge, add `depth` or narrow `ids` |
| `429` | Rate limited | Read `Retry-After`. Don't retry blind — on Starter, Tier 1 resets monthly, so a retry loop is pointless and the budget is already gone. Report it and say what's left |
| `500` | Usually a render that timed out | Ask for fewer or smaller nodes |

## Rules

- **Never spend a Tier 1 call to explore.** Confirm with `/meta`, then fetch once, precisely.
- **Report usage after every call, without being asked.** Figma won't tell you; the ledger will.
- **Never re-fetch what's already on disk.** If a previous response is in the destination, read it.
- **Say what a call will cost before making it** when the file is on Starter or the seat is View or
  Collab — six a month is a resource the user is entitled to spend knowingly.
- **Never assume position 4 is the file key** without checking position 5 for `branch`. That
  mistake returns real-looking JSON for the wrong file.
