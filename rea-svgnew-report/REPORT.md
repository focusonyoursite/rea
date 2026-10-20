# Reverse engineering report — svg.new

Target: <https://svg.new/> (public web app). All findings below come from REA
evidence captured in this session; the evidence IDs and retained artifacts live
in this directory.

## Method (REA workflow)

| Step | REA command | Result |
|---|---|---|
| Bounded browser scenario | `rea capture-browser-scenario` (launch mode) | `ev_029808…3220` — 83 network transactions, DOM, accessibility |
| CDP target discovery | `rea list-browser-targets` | `ev_c82faa…1625a` — target `519DA03…FBD` on `https://svg.new` |
| Passive page inspection | `rea inspect-web-page --include-script-sources` | `ev_1aefc2…c819` — 27 page scripts retained |
| Script export | `rea export-web-scripts` | `ev_b3a640…3ec5` — 27 scripts + `exported-scripts-manifest.json` |
| Static app analysis | `rea analyze-javascript-application` | graph `jag_eb88d…55d2` — 28 files, 341 773 AST nodes, 94 findings, 45 endpoint nodes |
| Runtime upload observation | `rea inspect-web-page --include-json-body-shapes` | `ev_2f1c21…f86aa` — `POST /api/image/vectorize` observed |
| Screenshot | `rea capture-web-screenshot` | `ev_c99206…bf67` / `ev_38fefa…0742` — PNG sha256 `e466c442…b8b` |

## Observations

### Stack and hosting
* Next.js App Router, served from Vercel (`?dpl=dpl_J3jorTQ1uucN3auC8HiadLvK6xZ6`),
  build id `_yBE_voFv0J8NfiWg5L9e`, Turbopack chunk pipeline.
* Auth: Supabase (`supabase.auth.session()`, `onAuthStateChange`,
  `localStorage["supabase.auth.token"]`); OAuth discovery
  `GET /.well-known/oauth-protected-resource` →
  `{"resource":"https://svg.new/api/mcp","authorization_servers":["https://xooyashzkuopxyyvhzdy.supabase.co/auth/v1"],"scopes_supported":["email"]}`.
* Monitoring: Bugsnag (`notify.bugsnag.com`, `sessions.bugsnag.com`).
* Third-party script: `https://what.vectorart.ai/script.js` (website id
  `ce12cb5d-affc-433f-b538-d76570e64a3c`) — the related product vectorart.ai.
### Server surface (41 routes, from the client route manifest)
```
/api/agent/vectorize            /api/agent/batch            /api/agent/batch/[id]
/api/agent/edit/convert         /api/agent/edit/recolor     /api/agent/edit/simplify
/api/agent/edit/remove-background  /api/agent/account      /api/image/vectorize
/api/image/claim                /api/image/remove-background /api/image/upload
/api/image/replicate/[promptId] /api/library               /api/mcp
/api/mcp/.well-known/oauth-protected-resource                /api/oauth-protected-resource
/api/og                         /api/prompt  /api/prompts   /api/prompts/rating
/api/purchase/credits           /api/purchase/pro          /api/purchase/starter
/api/purchase/ultimate          /api/purchase/webhook      /api/subscription/cancel
/api/subscription/change        /api/subscription/resume   /api/vectorizations
/api/vectorizations/[id]        /api/account/api-keys      /api/account/delete
/api/auth                       /api/user  /api/sessions    /api/webhook
/api/cron                       /api/draw/apply             /api/draw/completion
/api/generate
```

### The vectorization flow (runtime-confirmed)
1. Client picks a file (`input[type=file]`, accepts
   `image/png,image/jpeg,image/webp,image/avif,image/gif,image/bmp`).
2. `POST https://svg.new/api/image/vectorize`, `Content-Type: application/json`,
   body shape = object with one string property `image`
   (base64 data URL; `mode` added for non-general modes). Initiator:
   `/_next/static/chunks/2fa7623fb210f5d2.js`.
3. Response `200` with `Content-Type: text/event-stream` — the result is streamed
   over SSE.
4. On completion the app navigates to `/edit/<uuid>`; this run produced
   `https://svg.new/edit/7b278520-ed2d-4a8e-b6fd-3e442e42504d`.
5. The editor holds the traced vector: our 48×48 synthetic gradient PNG became a
   48×48 SVG with a `linearGradient` and 17 `<path>` elements
   (`evidence/vectorized-result.svg`).

### Product / commercial model (from the shipped pages)
* Free: 3 vectorizations per day, personal use. Pro: $9.99/month, 1 000
  vectorizations/month, REST API + MCP, commercial use, top-ups at 2¢/credit.
* Payments via Stripe. One conversion without an account is possible.
* Public developer surface: REST API (Bearer `svk_…`), hosted MCP server at
  `https://svg.new/api/mcp` (Streamable HTTP, OAuth or API key), local MCP
  package `@svgnew/mcp`, Claude Code plugin `svgnew/plugin`, TS SDK.
* MCP tools advertised: `create_upload`, `vectorize`, `vectorize_file`,
  `recolor_svg`, `simplify_svg`, `remove_background`,
  `remove_background_file`, `batch_vectorize`, `batch_vectorize_files`,
  `get_batch_status`, `list_vectorizations`.

## Inferences (not directly observed)
* The tracing engine itself is server-side: only the base64 image leaves the
  browser, so vectorization happens on their backend, not client-side.
* Credits are most likely debited server-side (the `/api/user` fetch after
  `supabase.auth.session()` and the pricing copy line up), but no credit-counter
  response body was captured.

## Unknowns / limitations
* No account was used, so authenticated routes, `/api/agent/*` behaviour and the
  credit ledger were not exercised.
* The editor's own client bundles were not all retained (capture was
  `policy_filtered` for scripts outside the allowed origin).
* SSE event contents were not retained: the response was still streaming and
  REA keeps no raw bodies for it.
* The static analysis result budget truncated one large file; coverage is
  `partial` (reported in the graph coverage section).
* A first attempt to inspect the page *while* reloading failed with a CDP
  `No script for id` protocol error; inspection was repeated without a reload.

## Reproducing
`rea capture-browser-scenario` needs a Chromium binary. This sandbox had none,
so Playwright's chromium was fetched and its shared libraries were extracted
into a local prefix (no root, no system install); `LD_LIBRARY_PATH` was pointed
at that prefix for the capture processes.

## Files
* `evidence/capture-load.json` — scenario capture of the initial page load
* `evidence/upload-capture.json` — passive capture during the upload
* `evidence/page-capture.json` — passive page inspection with script sources
* `evidence/exported-scripts-manifest.json` — 27 exported scripts with URLs/digests
* `evidence/vectorized-result.svg` — the traced SVG produced from our test PNG
* `evidence/test-upload.png` — the 48×48 synthetic image we submitted
* `page-developers.html`, `page-pricing.html` — shipped developer/pricing pages

