---
name: vmotif
description: Design on a Vmotif canvas through its MCP server. Generate images, video, code, SVG and HTML screens, run saved workflows, and read or edit canvases that people are watching live. Use when a task mentions Vmotif, a vmotif.com canvas link, or asks for generated design assets that should land on a shared canvas instead of a terminal.
license: MIT
metadata:
  author: vmotif
  version: "1.0.0"
  mcp: https://vmotif.com/api/mcp
  docs: https://vmotif.com/docs/mcp
---

# Vmotif

Vmotif is a node-based design canvas. A person and an agent look at the same
graph: every generation you start lands on a canvas as a node, wired to the
prompt and references that made it, and the canvas link is the deliverable.

This skill is the working loop for the hosted MCP server at
`https://vmotif.com/api/mcp`. Tool reference and full docs:
https://vmotif.com/docs/mcp

## Connect

The server speaks streamable HTTP with OAuth 2.1 sign-in on first use. If the
`vmotif` tools are not in your tool list, connect first:

```sh
claude mcp add --transport http vmotif https://vmotif.com/api/mcp
```

```sh
codex mcp add --url https://vmotif.com/api/mcp --oauth-resource https://vmotif.com/api/mcp vmotif
codex mcp login vmotif
```

Clients that read an `mcpServers` file (Cursor, Windsurf, VS Code):

```json
{ "mcpServers": { "vmotif": { "url": "https://vmotif.com/api/mcp" } } }
```

A client with a small instruction budget can connect to
`https://vmotif.com/api/mcp?tools=compact`: three tools (`discover_tools`,
`read_tool`, `execute_tool`) that find and run the same actions on demand.
Keep the OAuth resource at the URL without the query.

## The loop

1. **Scope.** Call `list_workspaces` first. Paid work spends a personal Pro or
   Enterprise wallet or an active Team workspace's pooled wallet. If the result
   says `requires_choice`, ask the user which scope to bill and file under, then
   pass `workspace_id` (`"personal"` or the workspace UUID) on paid calls. Left
   out, the server uses the canvas's scope, the only usable scope, or the scope
   this client used in the last 30 minutes; otherwise it answers
   `scope_required`.
2. **Canvas.** Generations from one client in one scope share a canvas, with
   inputs wired in. To work on an existing canvas, pass `canvas_id` from
   `list_projects` (only rows with `can_edit`) or from a link the user gave you.
   `create_canvas` makes a new one, optionally seeded with a prebuilt workflow
   (`premade_workflow_id`) or attached to a brand (`client_brand_id`). On the
   generate tools, pass `new_canvas: true` to start a fresh shared canvas; a
   30-minute pause does the same.
3. **Generate.** `generate_image` (`prompt`, optional `model`), `generate_code`
   (`prompt`, optional `model`, `target_language`, `screen_html` for a full HTML
   screen) and `generate_video` (`prompt`, optional `model`, `images`, `videos`,
   `duration`, `resolution`, `sound_enabled`) start a job and return at once.
   Leave `model` out for the curated default; `list_models` shows every id with
   its real price. Send a fresh UUID as `client_request_id` on every paid call
   and reuse it on a retry: the server returns the original job instead of
   charging twice.
4. **Wait.** A started job returns `{ id, status: "pending", poll_after_seconds,
   canvas }`. Either subscribe to the settlement event (below) or read
   `get_generation` after `poll_after_seconds`. `completed` means generated;
   saved means `projectAssociation.status` is `associated`. If it is `pending` or
   `retry_required`, call the `retry` it returns with the same id: that repairs
   placement for free and never regenerates. `unavailable` means stop retrying
   and keep the result (code results carry a signed `download_url`).
5. **Hand back the link.** Every canvas tool returns `url`. Finish by giving it
   to the user; the canvas is where they will look at the work.

## Inputs

- Reference images: prefer HTTPS URLs. A request caps near 4.5 MB, so stage a
  local file of up to 10 MB with `create_image_upload` (returns a single-use
  `upload_url` and the `image_url` it will serve from), PUT the raw bytes, then
  pass `image_url`.
- `transform_image` converts, compresses to a byte target or trims an image
  you own. It is synchronous and free.
- SVG previews and downloads do not inherit website CSS. Use literal colors,
  SVG-local variable definitions, or concrete fallbacks such as
  `var(--bx-ink, #d6d6d6)`.

## Brand and guides

`read_canvas` returns the whole graph: nodes with positions and data, edges,
tagged brand assets as live URLs, and the DESIGN.md in force. The server
applies the canvas's DESIGN.md, brand kit and typefaces to every generation on
that canvas, so you do not restate them in prompts; prompt for what the asset
is, not for the house style. `list_client_brands` lists the reusable brand kits
a canvas can be attached to; attach one with `create_canvas` or
`update_canvas` (`client_brand_id`, `null` detaches).

## Edit the graph

- `add_canvas_nodes` adds up to 25 nodes of any type the toolbar offers (image,
  prompt with its output node wired, video, capture, site scrape, code, text
  input, sticky note, group, social post and more). Nodes you create carry
  your client's name as provenance.
- `update_canvas` is one atomic edit: patch node data, move or resize, add or
  remove edges, delete nodes, rename the canvas. An unknown id fails the whole
  update without changing anything, so read ids from `read_canvas` first.
- A canvas that is executing or read-only rejects writes. Tell the user rather
  than retrying.

## Workflows

`list_workflows` and `get_workflow` read saved graphs. `list_premade_workflows`
and `add_premade_workflow` place a prebuilt recipe on a canvas. `run_workflow`
(`workflow_id`) runs a saved graph headlessly from an immutable snapshot taken
at enqueue; completed outputs are written back onto the canvas. Read progress
and per-output results with `get_workflow_run`, or subscribe to the run's
settlement event.

## Browser nodes

`run_capture` records a capture node's URL or element in a cloud browser and
writes the frames and animation context back onto the node. `run_site_scrape`
crawls a site scrape node's configured site and seeds the redesign graph beside
it: screenshots, brand assets, a measured DESIGN.md, direction previews and
per-page screen prompts. It is metered per page and marked destructive. Both
return once the node is claimed and finish in the background; read
`get_node_status`. A second call while one runs returns that run instead of
starting and charging another.

A bot check ends a capture. The server never tries to get past one, for any
site, and a blocked page is neither captured nor charged (the error code is
`site_blocked_by_bot_check`). Do not retry, change user agents or look for a
way around it; offer the user the option of uploading screenshots instead.

## Events: skip the polling

An agent that runs for hours should not have to poll. The server implements the
MCP Events draft (`events/list`, `events/subscribe`, `events/unsubscribe`) with
webhook delivery. The handshake advertises an `events` capability when it is
offered.

Events, all settlements: `generation.completed`, `generation.failed`,
`workflow_run.completed`, `workflow_run.failed`. `events/list` describes each
one, its filter arguments and its payload fields.

Subscribe with a public HTTPS URL you control and a `whsec_` signing secret:

```json
{
  "name": "generation.completed",
  "arguments": {},
  "delivery": { "mode": "webhook", "url": "https://hooks.example.com/vmotif", "secret": "whsec_..." },
  "ttlMs": 86400000
}
```

- The URL is challenged once before the subscription exists: the server POSTs
  `{ "type": "verification", "challenge": "..." }` and expects the challenge
  echoed back as JSON within five seconds.
- Each delivery is one signed POST with Standard Webhooks headers
  (`webhook-id`, `webhook-timestamp`, `webhook-signature`) plus
  `X-MCP-Subscription-Id`. Verify the signature and reject anything older
  than five minutes. The body is `{ eventId, name, timestamp, data,
  cursor: null }`; `eventId` is your idempotency key.
- `data` carries identifiers and status, never asset URLs. Read the result with
  `get_generation` or `get_workflow_run`.
- A 2xx acknowledges; other answers retry with backoff for a day. A
  subscription lapses at `refreshBefore` (one day by default, `ttlMs` up to
  seven); re-subscribing with the same name, arguments and URL refreshes it in
  place.
- A subscription hears everything the account starts, from any surface, and
  nothing another member starts.

With the Vercel AI SDK, `experimental_createMCPClient({ transport,
experimental_events: { store } })` plus `client.experimental_events.subscribe(...)`
and `experimental_createMCPEventWebhook({ store, onEvent })` handle the
challenge, the signatures and the refresh for you:
https://ai-sdk.dev/docs/ai-sdk-core/mcp-events

An interactive CLI session without a public endpoint keeps polling; that is
fine.

## Cost and care

- Reading canvases, models and usage is free. Generations, captures, site
  scrapes and workflow runs spend the GPU wallet of the scope they bill to.
  `get_usage` shows the balance (`null` means unlimited, not empty); a spent
  wallet is an expected outcome to report, not an error to retry.
- Start one job per intent. A job that already ran is not undone by a second
  one; read the result before deciding to generate again.
- A failed job is not charged unless the model had already produced output.
- Never embed secrets, tokens or private data in prompts or node content; the
  canvas is shared with everyone who can open it.
