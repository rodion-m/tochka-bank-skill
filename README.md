<h1 align="center">
  Tochka Bank API Skill for Claude Code
</h1>

<p align="center">
  <strong>A Claude Code skill for the Russian Tochka Bank (Точка.Банк) REST API — invoices (счета), payments, SBP QR codes, Open Banking statements, closing documents, webhooks</strong>
</p>

<p align="center">
  <a href="#installation">Installation</a> •
  <a href="#features">Features</a> •
  <a href="#usage">Usage</a> •
  <a href="#safety-hook">Safety Hook</a> •
  <a href="#documentation">Documentation</a> •
  <a href="#license">License</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Claude_Code-Skill-blueviolet?style=flat-square" alt="Claude Code Skill">
  <img src="https://img.shields.io/badge/Tochka_Bank-API-red?style=flat-square" alt="Tochka Bank">
  <img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="MIT License">
  <img src="https://img.shields.io/badge/Python-stdlib_only-blue?style=flat-square" alt="stdlib-only">
</p>

---

## Overview

This skill extends [Claude Code](https://claude.ai/code) with specialized knowledge and a stdlib-only Python CLI for the [Tochka Bank REST API](https://developers.tochka.com/docs/tochka-api/) — the most developer-friendly of Russian business banks. Covers both personal JWT and OAuth 2.0 + Consent authentication, with 19 CLI subcommands for the most common tasks: invoicing, statements, payments, SBP QR codes, webhooks. Verified end-to-end on production as of 2026-04-20.

## Features

- **Two authentication flows** — personal JWT (simple, read + payment drafts) or OAuth 2.0 + Consent (full invoicing + Open Banking statements)
- **Keychain-backed credentials** — tokens stored in macOS Keychain / Linux secret-tool / Windows Credential Manager, with file fallback; OAuth access_token auto-refreshed
- **Interactive wizards** — `init` and `init --oauth` with TTY preflight, app-registration guidance, and local HTTPS callback server (mkcert-aware)
- **19 CLI subcommands** covering:
  - **Accounts & balances** — `list-accounts`, `get-balance`
  - **Incoming payments** — `list-incoming` (SBP + card via acquiring)
  - **Outgoing drafts** — `list-for-sign` («На подпись»)
  - **Full bank statement** — `list-statement` (Open Banking async flow)
  - **Invoices (счета)** — `create-invoice`, `send-invoice` with PDF auto-download
  - **Closing documents** — акт / УПД / ТОРГ-12 / счёт-фактура via `create-closing-doc`
  - **Payment links** (интернет-эквайринг) — `create-payment-link`
  - **Webhooks** — `register-webhook` / `test-webhook` / `list-webhooks` / `delete-webhook`
  - **Acquiring registry** — daily reconciliation via `list-registry`
  - **OAuth consent introspection** — diagnose 403 scope issues with `list-consents` / `get-consent`
- **`--format {json,id,url}`** on `create-*` commands — get a single ID or URL on stdout for shell pipelines, envelope on stderr (no `jq` needed)
- **18-row error → fix cheatsheet** in SKILL.md — maps the most common prod errors to root cause and remedy
- **Optional safety hook** — Claude Code permission prompt with `[PROD]` / `[SANDBOX]` labels before every state-changing call
- **Offline OpenAPI spec** — `references/swagger.json` for greppable schema lookup via `jq` (OpenAPI 3.1.0, Tochka.API v1.90.4-stable)

## Installation

### Option 1 — Project scope (this repo only)

```bash
mkdir -p .claude/skills
git clone https://github.com/rodion-m/tochka-bank-skill.git .claude/skills/tochka-bank-api
```

### Option 2 — User scope (all projects)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/rodion-m/tochka-bank-skill.git ~/.claude/skills/tochka-bank-api
```

Restart Claude Code after cloning for the skill to be detected.

### First-time setup

The skill ships with interactive wizards. Pick ONE based on your goal:

```bash
# Simple: personal JWT from ЛК Точки. Good for reading + payment drafts.
# Does NOT cover invoices or full statement (those return 501 under JWT).
! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init

# Full: OAuth 2.0 + Consent. Needed for invoices, closing documents, Open Banking statement.
# Requires a one-time app registration at https://i.tochka.com/bank/services/m/integration/new.
! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init --oauth
```

The `!` prefix is **mandatory** — the wizard reads secrets via `getpass`, which needs a real TTY that Claude Code's Bash tool can't provide. The preflight check exits with this exact reminder if run without a TTY.

Adjust the path to `~/.claude/skills/tochka-bank-api/...` if you installed to user scope.

## Usage

Once installed, Claude Code automatically invokes this skill on Russian-banking tasks. Examples:

```
> Выставь счёт через Точку на 50000 рублей для ООО «Ромашка»
> Получи выписку из Точки за последнюю неделю
> Зарегистрируй webhook на incomingPayment к https://receiver.example.com/tochka
> Какие платёжки ждут подписания в Точке?
> Проверь баланс счёта в Точке
```

The agent consults [SKILL.md](SKILL.md) for the task → subcommand mapping, checks the `--format` options for pipeline-friendly output, and falls back to [references/endpoints.md](references/endpoints.md) or [ReDoc](https://enter.tochka.com/doc/v2/redoc) when it hits schema questions.

## Safety Hook

State-changing API calls (issuing invoices, dispatching payment orders, registering SBP QR codes, configuring webhooks) touch real money or persistent bank records. This repo ships an **optional but strongly recommended** Claude Code hook at [`hooks/tochka-require-confirmation.sh`](hooks/tochka-require-confirmation.sh) that forces a confirmation prompt before each.

Install:

```bash
# 1. Copy the hook script to your project's hooks directory
mkdir -p .claude/hooks
cp .claude/skills/tochka-bank-api/hooks/tochka-require-confirmation.sh .claude/hooks/
chmod +x .claude/hooks/tochka-require-confirmation.sh
```

2. Register it in `.claude/settings.json` (add the `hooks` block alongside your existing config):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/tochka-require-confirmation.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Now every `curl -X POST|PUT|PATCH|DELETE` to `enter.tochka.com` and every state-changing subcommand (`create-invoice`, `send-invoice`, `create-closing-doc`, `send-closing-doc`, `delete-closing-doc`, `register-webhook`, `delete-webhook`, `test-webhook`, `create-payment-link`) triggers a Claude Code permission prompt labelled:

- `[PROD] create-invoice — real money / bank records may be affected`
- `[SANDBOX] create-invoice — sandbox only, safe to allow`

Read-only calls (`list-*`, `get-*`, `config`, `init` validation) pass through silently.

## Documentation

- [`SKILL.md`](SKILL.md) — the router agents read on trigger: Quickstart (goal-branched), capability matrix (JWT vs OAuth), task → subcommand table, critical gotchas, day-to-day commands, hook explanation, error → fix cheatsheet
- [`references/auth.md`](references/auth.md) — JWT vs OAuth deep-dive, full authoritative permissions list (19 values), wizard internals, OAuth app-registration pitfalls (incl. the `localhost` vs `127.0.0.1` gotcha)
- [`references/endpoints.md`](references/endpoints.md) — body schemas, validation rules, field reference for all 19+ endpoints; ИП vs ООО field differences for invoicing
- [`references/webhooks.md`](references/webhooks.md) — signature verification (via OIDC discovery), retry semantics, idempotency guidance
- [`references/swagger.json`](references/swagger.json) — offline OpenAPI 3.1.0 spec (447 KB) for greppable `jq` lookup. Source: `https://enter.tochka.com/doc/openapi/swagger.json`

## Caveats

- **Not a multi-tenant SaaS helper.** OAuth mode *supports* multi-tenant, but the wizards are optimised for "I own the account" automation.
- **Not for other Russian banks.** Tinkoff, Sber, VTB have different schemas — don't port examples.
- **Tochka API deviates from АФТ/ОБР standards** (ЦБ Accounts v2.0/v3.0, Payment Initiation v1.0.0, ФАПИ Advanced v2.0). Expect a schema watershed around **01.10.2026** when the official v2.0 standards kick in. Don't copy OBR/OBIE examples hoping they work on Tochka.
- Some endpoint paths and field names rotate without strict semver — always verify against live [ReDoc](https://enter.tochka.com/doc/v2/redoc) before shipping code.

## Contributing

Issues and PRs welcome. Especially useful:

- New error-message rows for the cheatsheet (with exact error string + root cause + fix).
- Schema diffs if Tochka ships breaking changes.
- SBP operations verification (none of the SBP endpoints are live-tested in this skill — schemas are from swagger + ReDoc, not end-to-end prod calls).

## License

[MIT](LICENSE) — use, fork, adapt freely.
