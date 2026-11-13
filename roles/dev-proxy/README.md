# dev-proxy

Stable HTTPS names for dev servers on sandy, reachable from any tailnet device and nowhere else:

| Checkout               | URL                                            |
| ---------------------- | ---------------------------------------------- |
| main checkout          | `https://<name>.dev.unframework.net`           |
| worktree on `<branch>` | `https://<name>--<branch>.dev.unframework.net` |

No ports in URLs, no per-project proxy config, nothing to install on the phone or laptop.

## How it works

```
tailnet client ─▶ *.dev.unframework.net (Cloudflare, DNS-only, → sandy's 100.x / fd7a:…)
               ─▶ Caddy on tailnet :443 (wildcard Let's Encrypt cert via Cloudflare DNS-01)
               ─▶ Host rewritten to <branch>.<name>.localhost
               ─▶ portless proxy on :1355 ─▶ dev server on its assigned PORT (4000–4999)
```

- portless (`portless run …`) picks a free port, passes it as `PORT`, and registers `<name>.localhost` (or `<branch>.<name>.localhost` in a linked worktree) for as long as the process lives.
- Caddy maps the flat public name onto that portless name; one static Caddyfile covers every project.
- Off the tailnet the names still resolve, but `100.x` is unroutable. Certificate Transparency logs only show `*.dev.unframework.net`.
- Caddy also owns the node's ts.net root (`https://sandy.tail3cc64b.ts.net` → OpenClaw gateway), because a Tailscale Serve rule on 443 would hold the port.

## Naming rules

- `<name>` comes from `portless run --name <name>` (or `portless.json` / the package name). Lowercase letters, digits and single hyphens only; never `--`.
- Everything after the first `--` in a public name is the branch, which portless derives from the worktree's git branch.
- Extra services of one project get their own name, prefixed with the project: `herd` and `herd-storybook`.
- Apps see `Host: <…>.localhost`, not the public name. This only matters for OAuth redirects, domain cookies and absolute URLs.

## Opting a project in

1. Run the dev command through portless: `portless run --name <name> <cmd>`.
2. The server must listen on `$PORT`. Vite and a few other frameworks get `--port`/`--host` injected automatically; for others pass it through a shell, e.g. `sh -c 'storybook dev -p "$PORT" --host 127.0.0.1 --no-open --ci'`.
3. Fail on a taken port instead of drifting (Vite: `server.strictPort: true`).
4. Several processes behind one page (UI + API): put only the UI under portless and proxy the API through the UI dev server on the same origin (Vite `server.proxy`), so the browser never needs a second port.
5. Keep a portless-free script around (`PORTLESS=0 <script>` also bypasses it).

Examples: herd's root `yarn dev` (`scripts/dev.mjs`) and herd-ui's `storybook` script.

## Checking and troubleshooting

- `portless list` shows live routes; `~/.portless/routes.json` is the same data.
- `404 Unknown dev host` comes from Caddy: the name doesn't match the naming rules.
- A plain 404 from portless means no live route: the dev server isn't running, or its name/branch differs (compare with `portless list`).
- `systemctl status caddy`, `journalctl -u caddy`; `systemctl --user status portless-proxy`.
- Vite rejecting the host: shouldn't happen, since `.localhost` hosts are allowed by default; if a tool checks Host strictly, allow `.localhost`.

## Operations

- Applied by the `dev-proxy` play in `main.yml` (sandy only). DNS records follow `tailscale ip` on each run.
- The Cloudflare token (`cloudflare_dns_api_token`, vault) can edit all of `unframework.net`; it lives in `/etc/caddy/caddy.env` (root, 0600).
- Rollback: set `openclaw_tailscale_serve: true` in `host_vars/sandy.local.yml`, `sudo systemctl disable --now caddy`, re-run the playbook. Tailscale Serve takes 443 back.

## Agent blurb

Copy into a user-level `CLAUDE.md`, a skill, or a project's `CLAUDE.md`:

```markdown
## Dev servers on the tailnet

Dev servers on sandy get stable tailnet URLs through portless + Caddy:

- run them as `portless run --name <project> <cmd>`; the server must listen on `$PORT` (pass it explicitly for tools that ignore it, e.g. `storybook dev -p "$PORT"`)
- URL: `https://<project>.dev.unframework.net`, or `https://<project>--<branch>.dev.unframework.net` from a git worktree; check `portless list` for the live route
- extra services of one project: `--name <project>-<service>` (e.g. `herd-storybook`)
- names: lowercase, digits, single hyphens, never `--`
- prefer the project's own portless script when it has one (e.g. `yarn dev`); fall back to `tailscale serve` only for things that can't run under portless
```
