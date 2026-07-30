dont look













Project: Agentify

One-liner: A tool that lets any website become "agentic" — with user-granted permissions, sites can read/write files and take other agent-style actions on the user's machine, via a local daemon.

License: MIT + a mandatory visible-credit clause for anything built on/with it

Language: Go — fast, small static binaries, no runtime dependency headaches, good concurrency for many websocket connections, easy-to-read codebase.

No Electron — daemon + webview-based permission dialog instead, keeps everything in the multi-MB range.

Architecture — 3 layers
Daemon (local background process, Go binary, few MB)
Runs a websocket server
Holds permission state (SQLite)
Enforces scopes before doing any file I/O
Checks origin against blacklist and paths against sensitive-path registry before acting
Client SDK (JS library for site developers)
Wraps the websocket protocol into simple calls, e.g. agentify.readFile(), agentify.writeFile()
This is the "comfortable, easy integration" layer aimed at devs building AI sites
Permission dialog (native popup via webview, not a webpage-rendered dialog — can't trust the page to render its own trust prompt)
Triggered by the daemon when a site requests a capability it doesn't already have
Built with HTML/CSS via webview/webview, bound back to Go via Bind()
Websocket protocol (rough shape)
Every request carries the origin, checked against permissions before any action
Structured request/response with request IDs for promise-based SDK calls
Capability-negotiation handshake on connect: site declares desired capabilities → daemon returns currently granted → gaps trigger permission dialog
Daemon only accepts expected origins (manual origin check, since browsers don't block ws cross-origin by default)
Example message shape: {type: "request", capability: "fs.read", path, origin} / {type: "response", granted, data}
Permission model

Three grant durations:

Once — one-shot, expires immediately after use (visually the default/safest option in the UI)
Session — lasts until tab/site closes
Persistent ("Always") — lasts until manually revoked, scoped per-origin

Scoped by path, not just capability — e.g. write access to a specific sandboxed folder vs. broad filesystem access are treated as very different grants. New sites default to a sandboxed directory unless broader access is explicitly requested and granted.

Permission dialog contents:

Origin (unspoofable, prominent)
Capability in plain language ("wants to read a file" / "wants to write a file")
Exact literal path being requested
Extra warning + reason text if the path hits the sensitive registry
Once / Session / Always buttons, plus an equally prominent Deny
Sensitive-path registry
A .md file, human-readable, git-diffable, structured as a table: path pattern | reason | date added | report link
Applies universally to all sites regardless of grant tier — always forces one-time re-approval no matter what
Seeded proactively at launch with known-sensitive patterns (not just built reactively from abuse reports):
~/.ssh/*, ~/.aws/credentials, browser cookie/login stores, ~/.gnupg/*, wallet files, /etc/passwd, /etc/shadow, **/*.env, etc.
Abuse reports can extend this list over time
Blacklist (origin-based)
Anyone can report a site for abuse
Report requires video proof + full logs
You manually review and verify before action
If confirmed:
Site's origin → added to blacklist; daemon ignores all future requests from it
If the abuse involved reading specific sensitive files → those file patterns get added to the sensitive-path registry (manual-auto-approve list) so every site now needs fresh one-time approval for them
Distribution: daemon polls/fetches the blacklist (simplest v1 = a hosted JSON or the .md file itself)
Open question flagged for later: fail-open vs. fail-closed if your blacklist service is unreachable; ideally logs come from the daemon's own signed audit trail rather than user-submitted logs, to reduce fakeability
Public demo site
Static frontend hosted on GitHub Pages, served on your own custom domain
Requires the daemon to be installed — full capability demo, no mocked/sandboxed fallback, since install is lightweight (a few MB)
Flow: page checks for a running local daemon (ws://localhost:PORT) → if not found, shows platform-detected download → polls/retries connection until daemon comes online → auto-continues into the demo → first connection immediately triggers the real permission dialog (doubles as a feature showcase)
Uses free-tier LLM models only, so you have zero API cost
Rate limited to 50 requests/day per user — since GitHub Pages is static, real enforcement needs a lightweight backend (e.g. Cloudflare Worker + KV) sitting between the demo and the model API; can't rely on client-side-only limiting since it's trivially bypassed
Worth checking the free-tier model provider's ToS, since some free tiers disallow powering a public multi-user demo
Suggested Go project structure
