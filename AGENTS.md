# AGENTS.md

Guidance for AI coding agents (and humans) working in this repository.

> Asked to install or run this on someone's machine? Follow [`docs/setup.md`](docs/setup.md) and the
> [Setting up a host](#setting-up-a-host) rules below.

## Project

PlayGTA (a gtalab project) lets internet visitors take turns controlling a game console from the
browser. A Bun server runs the line and relays input to a console backend (a simulator, or a
patched chiaki-ng); MediaMTX serves the video over WebRTC through the server's WHEP proxy. Read
`README.md` first, then `docs/architecture.md` and `docs/security.md`.

## Commands

| Task                    | Command                                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| Install                 | `bun install` (CI: `bun install --frozen-lockfile`)                                              |
| Run (simulated console) | `bun start` (http://localhost:3000)                                                              |
| Run, restart on change  | `bun run dev`                                                                                    |
| MediaMTX                | `bun run video` (`vendor/mediamtx/mediamtx` from `scripts/build-mediamtx.sh`; see docs/setup.md) |
| Overlay preview         | `bun run overlay /tmp/overlay.png --once`                                                        |
| chiaki-ng               | `bun run chiaki:build`, then `bun run chiaki:test-netinput` and `bun run chiaki:test-stream-out` |
| Admin CLI               | `bun run admin <command>` (needs `ADMIN_TOKEN`)                                                  |
| Type check              | `bun run typecheck`                                                                              |
| Lint (oxlint)           | `bun run lint` (fix with `bun run lint:fix`)                                                     |
| Format (oxfmt)          | `bun run format` (verify with `bun run format:check`)                                            |
| Tests                   | `bun run test` (bun:test, only `test/`)                                                          |
| All checks              | `bun run check` (typecheck, lint, format check, tests)                                           |
| Shell scripts           | `shellcheck -x -S warning deploy/macos/*.sh scripts/*.sh`                                        |

Bun 1.4+ runs the TypeScript directly: there is no build step and no `dist/`. The code sticks to
`node:*` modules (Bun implements them) rather than `Bun.*` APIs. The lockfile is `bun.lock`.

## Layout

```
src/domain/        pure rules, no I/O, no runtime APIs except what is needed for data
src/application/   use cases and ports (interfaces the adapters implement)
src/adapters/      I/O: config, HTTP, WebSocket, consoles, MediaMTX, Kick, files
src/main.ts        composition root only
public/            reference web client: plain HTML, one CSS file, ES modules (no build)
scripts/           admin CLI, overlay renderer, build and test scripts
deploy/macos/      isolation kit: user, Seatbelt, pf, LaunchAgents, run scripts
mediamtx/          MediaMTX configuration
chiaki/            chiaki-ng patches and their tests (AGPL-3.0, see chiaki/NOTICE)
test/              bun:test suites
docs/              architecture, protocol, security, streaming, configuration
```

## Conventions

- TypeScript is strict (`noUncheckedIndexedAccess`, `verbatimModuleSyntax`, `erasableSyntaxOnly`).
  Only erasable syntax: no enums, no namespaces, no parameter properties. Import local files with
  the `.ts` extension and types with `import type`.
- Dependencies point inward: `domain` imports nothing from the other layers; `application` imports
  `domain`; `adapters` may import both. New I/O goes behind a port defined in `application`.
- Inject time (`Clock`) and I/O (`fetch`, sockets, fs) into stateful classes so tests stay
  deterministic. Never call `Date.now()` in domain code.
- Private state uses `#fields`. Public APIs get TSDoc comments that explain why, not what.
- Treat every browser value as untrusted: parse, bound and clamp it at the edge.
- Never log secrets or tokens. Store secrets only as hashes.
- Configuration comes only from `src/adapters/config/env-config.ts`. A new variable needs
  validation there, a test in `test/config.test.ts` and a row in `docs/configuration.md`.
- Keep the project name in `src/app-info.ts`, `package.json` and `deploy/macos/lib.sh`.
- Code, comments, docs and UI text are in English. Use conventional commits (`feat:`, `fix:`,
  `docs:`, `refactor:`, `test:`, `chore:`).

## Setting up a host

When a user asks you to install or run this on their machine, follow `docs/setup.md`:

- Go in its stage order (simulated console, builds, real console on localhost, tunnel) and run
  each stage's "You should now see" check before moving on.
- The human does the console side and the secrets: PS5 settings, PSN Account ID, the Link Device
  code, Cloudflare login and Kick. Tell them which mode-600 file a secret goes in; never ask them
  to paste it into the chat.
- Do not skip or weaken security steps (MediaMTX credentials, `PUBLIC_URL`, `TRUSTED_PROXY`, the
  isolation kit). On an error, stop and explain it instead of improvising a workaround.

## Rules for agents

- Run `bun run check` before you say a change is done; add or update tests for behavior changes.
- Tests must use loopback addresses and ephemeral ports (`listen(0, '127.0.0.1')`). Never bind a
  public interface and never call real third-party services from tests.
- Do not commit `.env`, `server.env`, `kick.env`, state files or anything under `vendor/`.
- Do not add dependencies unless they clearly pay off; prefer the `node:*` standard library.
- Do not run `deploy/macos/install.sh`, `pf-isolate.sh` or anything with `sudo` or `launchctl`
  unless the user explicitly asks: they change the host. `install.sh --dry-run DIR` is safe.
- Files in `chiaki/` are AGPL-3.0 and are patches against a pinned chiaki-ng commit: keep them
  applying cleanly (`git apply --check`) and do not move MIT code into them.
- Keep the server game-agnostic: no game names, art or game-specific logic. Game-tuned controls
  live only in the page's keyboard layouts (`public/js/keymap.js`, chosen with `KEYBOARD_LAYOUT`),
  next to the `generic` one.
- The backend is the product; `public/` stays a minimal reference client. A change to a WebSocket
  message, the input frame or the admin API must update `docs/protocol.md`.
- Security-sensitive spots: the admin gate and client identity (`src/adapters/http/client-origin.ts`,
  `TRUSTED_PROXY`), the SDP sanitizer (`src/adapters/mediamtx/sdp.ts`), paused mode in
  `src/adapters/http/http-server.ts`, and the pf anchor. Update `docs/security.md` with them.
- Secrets never go on a command line (other local accounts can read it with `ps`); pass them in the
  environment or a file. The two exceptions are documented in `docs/security.md`.
