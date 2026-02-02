# OpenClaw - Project Memory

## What is OpenClaw?
A personal AI assistant platform you run on your own devices. It integrates with messaging channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Teams, Matrix, etc.) and connects to AI providers (Claude, ChatGPT, etc.). Open-source, single-user, always-on.

**Version**: 2026.1.29 | **License**: MIT | **Runtime**: Node 22+ / TypeScript ESM

## Quick Reference

### Commands
```bash
pnpm install          # Install deps
pnpm build            # Build with tsdown
pnpm lint             # Lint with oxlint
pnpm format           # Format with oxfmt
pnpm test             # Run vitest
pnpm test:coverage    # With V8 coverage (70% threshold)
pnpm openclaw ...     # Run CLI in dev mode
```

### Project Structure
```
src/                   # Main TypeScript source
  cli/                 # CLI wiring & commands (commander.js)
  commands/            # High-level command implementations
  gateway/             # WebSocket gateway server
  channels/            # Channel management & plugins
  providers/           # AI model provider integrations
  agents/              # AI agent implementations
  infra/               # Infrastructure & system utilities
  config/              # Configuration loading/management
  plugins/             # Plugin system core
  plugin-sdk/          # Public SDK for extensions
  media/               # Media processing pipeline
  routing/             # Message routing logic
  terminal/            # Terminal/TTY utilities (palette, table)
  wizard/              # Onboarding wizard
  entry.ts             # Main CLI entry point
extensions/            # ~30 plugin packages (channels, memory, auth, etc.)
apps/                  # Native apps (ios/, android/, macos/)
docs/                  # Mintlify docs (docs.openclaw.ai)
scripts/               # Build/dev/release helpers
```

### Key Conventions
- **TypeScript ESM**, strict typing, avoid `any`
- **Linting/formatting**: oxlint + oxfmt (run before commits)
- **Tests**: Vitest, colocated `*.test.ts`, 70% coverage thresholds
- **File size**: aim for ~500-700 LOC max
- **Naming**: `OpenClaw` for product/docs, `openclaw` for CLI/config/paths
- **Commits**: use `scripts/committer "<msg>" <file...>` for scoped staging
- **CLI progress**: use `src/cli/progress.ts` (osc-progress + @clack/prompts)
- **Terminal output**: use `src/terminal/table.ts` for tables, `src/terminal/palette.ts` for colors
- **Plugins**: keep plugin deps in extension `package.json`, not root
- **Tool schemas**: no `Type.Union` in tool inputs; use `stringEnum`/`optionalStringEnum`

### Architecture Highlights
- Gateway: WebSocket-based, multi-channel, model failover, hook system
- Plugin system: ESM modules loaded via jiti, SDK at `openclaw/plugin-sdk`
- Config: YAML/JSON at `~/.openclaw/config`, multi-profile support
- Build: tsdown (5-10x faster than tsc)
- Mobile: SwiftUI with Observation framework (prefer over ObservableObject)

### Extension Ecosystem
**Channels**: bluebubbles, discord, google-chat, imessage, line, matrix, mattermost, msteams, nextcloud-talk, nostr, signal, slack, telegram, tlon, twitch, whatsapp, zalo, zalouser
**Features**: memory-core, memory-lancedb, open-prose, llm-task, lobster, voice-call, diagnostics-otel, copilot-proxy
**Auth**: google-gemini-cli-auth, google-antigravity-auth, qwen-portal-auth

### Important Rules
- Never edit `node_modules`
- Never update the Carbon dependency
- Patched dependencies (`pnpm.patchedDependencies`) must use exact versions (no `^`/`~`)
- Patching requires explicit approval
- Release version changes require operator consent
- Never commit real phone numbers, videos, or live config values
- Web provider creds at `~/.openclaw/credentials/`
- See `AGENTS.md` for full operational guidelines
