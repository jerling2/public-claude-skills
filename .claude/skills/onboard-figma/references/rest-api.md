# Reference: Figma REST API

Base URL `https://api.figma.com`. Authentication is out of scope here — tokens, headers, and the
scopes each endpoint needs are documented at
[Authentication](https://developers.figma.com/docs/rest-api/authentication/) and
[Scopes](https://developers.figma.com/docs/rest-api/scopes/).

Checked against Figma's documentation on 2026-08-30. Figma states it reserves the right to change
rate limits, so treat every number here as a snapshot and re-check the linked page.

## How often you can call it

Three things decide any single limit, together: **the seat type of the user**, **the rate limit
tier of the endpoint**, and **the plan of the resource being requested**.

Tier is a cost bucket, not a permission level. Figma groups endpoints by what they cost to serve —
tiers are "determined based on a number of factors, including the infrastructure and cost required
to support the endpoint" — so Tier 1 holds the expensive endpoints and carries the *smallest*
allowances. A higher tier number means a cheaper endpoint and a more generous limit.

**Tier 1** — `GET file`, `GET file nodes`, `GET image`

| Seat | Starter | Professional | Organization | Enterprise |
| --- | --- | --- | --- | --- |
| View, Collab | 6/month | 6/month | 6/month | 6/month |
| Dev, Full | 6/month | 10/min | 15/min | 20/min |

**Tier 2** — comments, dev resources, discovery, `GET image fills`, folders, projects, variables (GET), version history, webhooks

| Seat | Starter | Professional | Organization | Enterprise |
| --- | --- | --- | --- | --- |
| View, Collab | 5/min | 5/min | 5/min | 5/min |
| Dev, Full | 5/min | 25/min | 50/min | 100/min |

**Tier 3** — activity logs, components & styles, developer logs, `GET file metadata`, folder metadata, library analytics, payments, users, `POST variables`

| Seat | Starter | Professional | Organization | Enterprise |
| --- | --- | --- | --- | --- |
| View, Collab | 10/min | 10/min | 10/min | 10/min |
| Dev, Full | 10/min | 50/min | 100/min | 150/min |

> On Figma's page the Starter column is one cell spanning both seat rows, so the `Dev, Full` row
> renders with one fewer cell than the header has columns. Read naively, every number shifts a
> column left and Organization's numbers appear under Professional. The tables above are already
> corrected for this.

Two qualifications that change how much the numbers are worth:

- **View and Collab figures are ceilings, not allowances.** Figma: "requests are limited up to the
  given amount. Depending on traffic and demand, the actual limit may be lower."
- **The limit follows the file, not the token.** Personal access tokens are account-wide rather
  than plan-bound, so the plan that matters is the one the *requested file* lives in.

Exceeding a limit returns `429`, carrying `Retry-After` (seconds to wait), `X-Figma-Plan-Tier`
(the plan tier of the resource requested), `X-Figma-Rate-Limit-Type`, and `X-Figma-Upgrade-Link`.

Full tables, the per-token-type accounting, and Figma's guidance on batching, caching, and retries:
**[REST API — Rate Limits](https://developers.figma.com/docs/rest-api/rate-limits/)**

### Why the free plan is the expensive one

These limits are recent: Figma's changelog announced "Published and adjusted REST API rate limits
will go into effect on November 17, 2025," and the rate limits page now reads "As of November 17,
2025 the updated rate limits are in effect." The cost of that change is not spread evenly, and it
lands hardest on Starter — the plan Figma bills as Free — in a way worth understanding before
planning around it.

On Starter, **Tier 1 is the only tier that drops to a monthly quota**: up to 6 calls per month,
where Tiers 2 and 3 keep working per-minute allowances (5/min and 10/min). That single capped tier
is exactly `GET file`, `GET file nodes`, and `GET image` — the endpoints that return design data,
and so the ones anything reading a design has to call. The cheaper metadata endpoints remain
callable at a per-minute cadence on the free plan; the ones carrying the design itself do not.

Seat upgrades do not help here. On Starter the 6/month figure covers both seat rows, so a Dev or
Full seat buys no additional Tier 1 capacity. Professional is the first plan on which Tier 1
becomes a per-minute budget at all, at 10/min for Dev and Full seats.

Because the limit attaches to the file's plan rather than the caller's seat, upgrading does not
retroactively cover files left behind. Figma's own example: "if you use a personal access token to
get the content of a file in a Starter plan, requests to that file are limited to up to 6 per month
even if you have a Full seat in a different plan." A file still sitting in a Starter-plan folder
after an upgrade is still metered at 6/month.

## What data it returns

`GET /v1/files/:key` returns the file as a JSON document. The response shape, the node fields, and
the query parameters that narrow it are all documented by Figma and change with the product — read
them at the source rather than from a copy here:

**[REST API — File endpoints](https://developers.figma.com/docs/rest-api/file-endpoints/)**

For the vocabulary that documentation assumes — node types and how the tree nests — see
`references/hierarchy-vernacular.md`. For getting a `file_key` out of a URL in the first place, see
`references/design-url-decoding.md`.
