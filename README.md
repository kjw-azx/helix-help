# helix-help

A **docs-grounded chatbot** for people building Helix apps, and an example of two
gateway surfaces cooperating with no backend of its own: the app loads the
platform documentation **at runtime through the fetch proxy** and answers
questions over the **OpenAI-compatible LLM surface**.

Live at <https://help.azx.helix.azxlabs.io> (Helix app slug `help`).

```
boot:   browser ──GET /_api/fetch/https://azx-pbc-oss.github.io/helix/apps/quickstart.md──▶
        edge (manifest origin grant) ──▶ egress (SSRF checks) ──▶ GitHub Pages

turn:   browser ──POST /_api/openai/v1/chat/completions──▶
        edge (allowlist + budget, meters) ──▶ egress (injects vendor key) ──▶ gpt-5-mini
              ◀──────────────────── SSE deltas streamed back ──────────────────────┘
```

- **No secrets in the bundle, no RAG, no embeddings.** The docs site serves a
  markdown twin of every page; the app fetches five of them (the "Building apps"
  core + local dev), concatenates them into the system prompt, and re-sends them
  each turn. The whole context is **~38 KB (~10K tokens)**.
- **Keyless origin grant.** The fetch proxy needs only
  `capabilities.fetch.origins: [{ origin: "https://azx-pbc-oss.github.io" }]` in
  the manifest — no connection secret, no admin approval. Egress still applies
  its SSRF controls; the edge meters the call.
- **Session-gated.** Both `/_api/fetch/*` and `/_api/openai/v1/*` ride the same
  session as the rest of the app; an unauthenticated fetch gets a 401 (the app
  prompts to sign in).
- **Metered + allowlisted.** The model must be in the manifest
  `capabilities.llm.models`, and calls count against `dollarsPerDay`.
- **Degrades gracefully.** Docs load in parallel and partial failure is kept
  (the status chip shows `4/5 pages`); if none load, the bot says it's answering
  from general knowledge. Replies render as markdown-lite, built with
  `textContent` only — nothing from the model is ever parsed as HTML.

## Deploying

Every push to `main` deploys a **preview** through
[`.github/workflows/helix-deploy.yml`](.github/workflows/helix-deploy.yml), which
calls the Helix reusable workflow
[`AZX-PBC-OSS/helix-actions`](https://github.com/AZX-PBC-OSS/helix-actions).
The workflow builds the app in a job with no token permission, then a separate
job uploads `dist/` with its own GitHub OIDC token. The repo holds no secret.

A preview goes live only when someone promotes it — on the app's **Versions**
tab in the portal, or with `helix promote <n>`.

The connection lives on the app's **Repository** tab
(`kjw-azx/helix-help`, branch `main`). To require deploys to come from this
workflow only, set **Only this workflow** there to:

```
AZX-PBC-OSS/helix-actions/.github/workflows/deploy.yml@refs/tags/v1
```

`helix.json` names the app's slug, the portal and the build directory; both the
workflow and `helix deploy` read it.

## Capabilities

The app needs an LLM grant and a fetch-proxy origin grant in its manifest, set
once on the app's **Capabilities** tab or through the portal API:

```bash
curl -fsS -X PUT "https://portal.azx.helix.azxlabs.io/api/v1/apps/help/manifest" \
  -H "authorization: Bearer $HELIX_TOKEN" \
  -H "content-type: application/json" \
  -d '{"capabilities":{"llm":{"models":["gpt-5-mini"],"dollarsPerDay":2},"fetch":{"origins":[{"origin":"https://azx-pbc-oss.github.io"}]}}}'
```

## Local development

```bash
npm ci
npm run typecheck
npm run build      # writes dist/ (not committed)
```

`npm run dev` serves the UI, but the `/_api/*` calls only work when the app is
served by a Helix edge. To try a change against the platform without pushing,
sign in and deploy a preview from your machine:

```bash
npx @azx-pbc/helix-cli login
npx @azx-pbc/helix-cli deploy
```
