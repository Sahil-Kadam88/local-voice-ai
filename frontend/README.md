# Local Voice Agent — browser interface

The browser UI for Local Voice Agent: a Next.js app exported as static files
and served by the Python backend at `http://localhost:8080`.

Derived from LiveKit's
[agent-starter-react](https://github.com/livekit-examples/agent-starter-react)
template (MIT), adapted for this project's branding and connection flow.

## How it connects

1. On **Start call** the app `POST`s to `/api/connection-details` on the
   backend (same origin, or `NEXT_PUBLIC_BACKEND_URL` when set) and receives a
   LiveKit server URL plus a short-lived participant token.
2. `livekit-client` joins the LiveKit room over WebRTC; microphone audio flows
   to the agent and back.
3. The first-boot screen polls `GET /api/status` every 2 s to show per-service
   readiness and model-download progress.

There are no server-side routes: `next.config.ts` uses `output: 'export'`, so
the whole UI is static assets.

## Development

```bash
pnpm install
pnpm dev          # http://localhost:3000
pnpm build        # static export → out/ (served by the backend)
pnpm lint
pnpm format:check
```

When running against a backend on another machine:

```bash
NEXT_PUBLIC_BACKEND_URL=http://192.168.1.40:8080 pnpm dev
```

## Configuration

UI identity (title, description, logo, accent colors, start button) lives in
[`app-config.ts`](./app-config.ts). Theme colors are CSS variables in
`styles/globals.css`; fonts are Public Sans (Google) and Commit Mono (local,
`fonts/`).
