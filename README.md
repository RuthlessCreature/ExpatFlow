# ExpatFlow Browser Bridge

Cloudflare Browser Run + Playwright MCP experiment for remote browser control and human-in-the-loop login.

## Goal

This repository validates one concrete workflow first:

1. Start a Cloudflare Browser Run session.
2. Navigate the remote browser to a login page.
3. Generate a Cloudflare Live View URL.
4. Open the Live View URL locally and complete login / MFA manually.
5. Reconnect to the same Browser Run session by `sessionId`.
6. Continue browser automation through Playwright.
7. Expose the browser through MCP for later ChatGPT integration.

## Architecture

```text
ChatGPT / MCP client
        |
        v
Cloudflare Worker
  |             |
  |             +--> /mcp  (Playwright MCP)
  |
  +--> /api/live/start
  +--> /api/live/inspect
  +--> /api/live/goto
  +--> /api/live/close
        |
        v
Cloudflare Browser Run
        |
        v
Live View <--> Human login
```

## Quick start

Prerequisites:

- Cloudflare account with Browser Run enabled.
- Node.js 20+.
- Wrangler authenticated with your Cloudflare account.

```bash
npm install
npx wrangler login
npm run deploy
```

After deployment, start a remote login session:

```bash
curl -X POST "https://<worker>.workers.dev/api/live/start" \
  -H "content-type: application/json" \
  -d '{"url":"https://www.facebook.com/"}'
```

The response contains:

- `sessionId` — Browser Run session to reconnect to.
- `liveViewUrl` — open this URL in your normal browser and interact with the remote Cloudflare browser.
- `url` / `title` — current remote page state.

After manual login, inspect the same session:

```bash
curl "https://<worker>.workers.dev/api/live/inspect?sessionId=<SESSION_ID>"
```

Navigate the same logged-in session:

```bash
curl -X POST "https://<worker>.workers.dev/api/live/goto" \
  -H "content-type: application/json" \
  -d '{"sessionId":"<SESSION_ID>","url":"https://www.facebook.com/"}'
```

Close it when finished:

```bash
curl -X POST "https://<worker>.workers.dev/api/live/close" \
  -H "content-type: application/json" \
  -d '{"sessionId":"<SESSION_ID>"}'
```

## MCP

The Worker also exposes:

- `/mcp` — Streamable HTTP MCP endpoint.
- `/sse` — compatibility endpoint.

This is backed by `@cloudflare/playwright-mcp` and Browser Run.

## Important limitation

Browser Run is a cloud browser. It does **not** inherit the cookies, IP, fingerprint, extensions, or device history of your local Chrome. The purpose of this experiment is to test whether manual Live View login plus session reuse is stable enough for the target websites before building more automation.

Do not commit credentials, cookies, API tokens, or storage-state files to this repository.
