<div align="center">

# ⚡ LaraNuxt SaaS

**Skip the boring 80%. Ship the part only you can build.**

A production-shaped, multi-tenant SaaS foundation: **Laravel 13** API + **Nuxt 4** app, with teams, roles, Claude-powered AI, Stripe billing, two-factor auth and a support-ready admin console, all wired up and covered by ~880 tests.

[![CI](https://github.com/sanchaichie/laranuxt-saas/actions/workflows/ci.yml/badge.svg)](https://github.com/sanchaichie/laranuxt-saas/actions/workflows/ci.yml)
![Laravel](https://img.shields.io/badge/Laravel-13-FF2D20?logo=laravel&logoColor=white)
![Nuxt](https://img.shields.io/badge/Nuxt-4-00DC82?logo=nuxt.js&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.4-777BB4?logo=php&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-ready-4169E1?logo=postgresql&logoColor=white)
![Stripe](https://img.shields.io/badge/Stripe-Cashier-635BFF?logo=stripe&logoColor=white)
![Claude](https://img.shields.io/badge/AI-Claude-D97757)

[Features](#-whats-in-the-box) · [Quick start](#-quick-start) · [Architecture](#-architecture) · [Screenshots](#screenshots) · [Configuration](#%EF%B8%8F-configuration) · [Reference](#-api-and-pages-reference)

<img src="docs/screenshots/project-summary.png" alt="A project page with its approved AI summary and key points" width="860">

</div>

---

## 💡 Why this exists

Every B2B SaaS rebuilds the same things before it ships its first real feature: workspaces, invitations, permissions, billing, audit trails, account security. Those are also the things that hurt most when they're wrong, because one leaked tenant or one unchecked permission is enough to lose a customer.

LaraNuxt SaaS gives you those pieces already built, tested and documented. **Projects** is the sample feature: copy its pattern for your own domain, or delete it.

## 📦 What's in the box

| | Feature | The short version |
|:-:|---|---|
| 🏢 | **Multi-organization workspaces** | Sign up and you own a workspace. Join or create others and switch from the sidebar. Data never crosses between them. |
| 👥 | **Teams and invitations** | Invite by email, assign a role, remove members. There's always at least one Owner. |
| 🔐 | **Custom roles** | Owner, Admin and Member out of the box, and build your own from a permission list. No one can grant more than they hold. |
| 🤖 | **AI, done responsibly** | Claude summaries (draft → approve), streaming "Improve with AI" writing help, and a private "Ask AI" chat per project. |
| 💸 | **AI cost control** | Every call is metered (tokens and $). Each organization has a monthly budget, and AI stops when it runs out. |
| 💳 | **Stripe billing** | Free and Pro plans, one-time AI credit packs, the Customer Portal, and live invoices. No card data touches your server. |
| 🛡️ | **Account security** | Email verification, password reset, TOTP two-factor with recovery codes, device list with remote sign-out. |
| 📜 | **Audit log** | Who did what, when, and from where, with before/after diffs. Append-only and pruned on a schedule. |
| 🧑‍✈️ | **Platform admin console** | Look up users and organizations, reset 2FA, suspend, rescue ownerless organizations, and **view as user** (read-only). Every action is logged with a reason. |
| 🧪 | **Test-first** | ~880 Pest tests, an org-A-vs-org-B isolation test for every tenant feature, and fakes for AI and Stripe. CI runs on every push. |

## 🧭 A day in the app

> **Maya** signs up and lands in her new workspace, *Acme Studio*. She invites two teammates as Members and creates a "Contributor" role for a freelancer.
>
> She adds a project. Within seconds Claude drafts a summary, and her teammate approves it. She clicks **Improve with AI** on the description and watches a sharper version stream in.
>
> Halfway through the month the AI budget runs low, so she buys a $10 credit pack on Stripe Checkout and carries on. Every step is in the audit log.
>
> When the freelancer loses their phone, a platform admin resets their two-factor with a written reason. The freelancer gets an email, and the reason goes into the platform log.

## 🚀 Quick start

**You need:** PHP 8.4+ (with `pdo_pgsql`), Composer, Node 22.22.3+ (`web/.nvmrc`), and PostgreSQL running locally.

```bash
# 1. Databases: one for the app, one the tests can rebuild
createdb -h 127.0.0.1 -U postgres saas_starter
createdb -h 127.0.0.1 -U postgres saas_starter_test

# 2. API
cd api
composer install
cp .env.example .env            # set DB_* to match your Postgres
php artisan key:generate
php artisan migrate

# 3. Web
cd ../web
nvm use && npm ci
cp .env.example .env            # NUXT_PUBLIC_API_BASE=http://localhost:8000
```

**Run it** (three terminals):

```bash
cd api && PHP_CLI_SERVER_WORKERS=4 php artisan serve --no-reload   # API  → http://localhost:8000
cd api && php artisan queue:work                                    # emails + AI summaries
cd web && npm run dev                                               # app  → http://localhost:3000
```

Open **http://localhost:3000** and register, and your first organization is created for you.

> [!TIP]
> **No API keys? No problem.** Set `AI_PROVIDER=fake` and `BILLING_PROVIDER=fake` in `api/.env` to run the whole app offline with canned AI answers and simulated billing.

> [!NOTE]
> The API needs several workers because a streaming AI answer holds one open until it finishes. `--no-reload` is required for workers, so restart the server after editing `.env`. Locally, emails (verification, resets, invitations) are written with their links to `api/storage/logs/laravel.log`.

> [!IMPORTANT]
> Database-wiping commands (`migrate:fresh`, `db:wipe`, …) refuse to run unless the database name ends in `_test`, so your dev data can't be lost by accident.

**Become a platform admin:** register, turn on two-factor in Account settings, then run:

```bash
cd api && php artisan admin:grant you@example.com     # admin:revoke undoes it
```

In production, also run the scheduler (`php artisan schedule:work`, or a cron entry for `schedule:run`) to prune old audit entries.

## 🏗 Architecture

```
          ┌──────────────┐   Bearer token   ┌────────────────────┐
Browser ─▶│  Nuxt 4 app  │ ───────────────▶ │   Laravel 13 API   │──▶ PostgreSQL
          │   (web/)     │ ◀── JSON / SSE ── │      (api/)        │
          └──────────────┘                  └───┬───────────┬────┘
                                                │           │
                                   Claude API ◀─┘           └─▶ Stripe Checkout / Portal
                                (server-side only)                    │
                                                  signed webhook ◀────┘
                    Queue worker: emails, AI summaries · Scheduler: audit pruning
```

Seven rules hold the design together:

1. **The browser is just a client.** Identity (Sanctum), the active organization and permissions (policies) are all enforced in the API.
2. **Tenancy is server-side.** The current organization lives on the user record, never in the URL or the request. Another organization's record is simply a 404.
3. **Permissions, not role names.** Policies ask "can this user edit projects?", never "is this user an Admin?".
4. **AI goes through one door.** A single `AiClient` interface that meters usage, enforces the budget and is faked in tests. The provider key never leaves Laravel.
5. **Stripe is the source of truth.** Checkout and the Portal handle payment, and only the signed webhook changes a plan.
6. **Audits are transactional.** An audit entry is written in the same transaction as the change it describes.
7. **There's one way in.** Every token comes from a single place that enforces two-factor and refuses suspended accounts.

### Roles at a glance

| Can… | Owner | Admin | Member |
|---|:---:|:---:|:---:|
| View and create projects, edit their own | ✅ | ✅ | ✅ |
| Use AI features | ✅ | ✅ | ✅ |
| Edit and delete any project | ✅ | ✅ | — |
| Invite, change roles, remove members (roles they cover) | ✅ | ✅ | — |
| View the audit log and AI usage | ✅ | ✅ | — |
| Create, edit and delete roles | ✅ | — | — |
| Change the monthly AI budget | ✅ | — | — |
| Manage plan, payment details and invoices | ✅ | — | — |

Owner is built in: it has every permission, can't be edited, and an organization always keeps one.

<details>
<summary><b>📖 Glossary</b></summary>

| Term | Meaning |
|---|---|
| **Organization** | A workspace that owns its data: projects, roles, audit log, AI usage and billing. |
| **Current organization** | The one you're working in, stored on the server and switched from the sidebar. |
| **Draft and approve** | AI output is saved as a draft for a human to accept. AI never changes or deletes anything on its own. |
| **AI budget** | Dollars of AI an organization can spend per UTC month. It's set by the plan, and owners can lower it or set it to 0. |
| **AI credits** | A one-time purchase that tops up this month's budget. |
| **Plan** | Free or Pro, changed only by Stripe's signed webhook. |
| **Platform admin** | Someone who runs the app (not an organization role). Granted from the command line, two-factor required, never sees project content. |
| **View as user** | A read-only, time-limited admin session as a user. The user is emailed. |

</details>

## Screenshots

<table>
<tr>
<td width="50%"><img src="docs/screenshots/projects.png" alt="Projects list in the Acme Studio organization"><br><b>Projects</b>: everything in the current workspace</td>
<td width="50%"><img src="docs/screenshots/writing-help.png" alt="Edit project form with an AI suggestion ready to accept"><br><b>Improve with AI</b>: a better description streams in</td>
</tr>
<tr>
<td><img src="docs/screenshots/project-chat.png" alt="Ask AI panel with two questions and answers about the project"><br><b>Ask AI</b>: a private chat about a project</td>
<td><img src="docs/screenshots/ai-usage.png" alt="AI usage page with budget, totals and usage by feature"><br><b>AI usage</b>: spend against the monthly budget</td>
</tr>
<tr>
<td><img src="docs/screenshots/members.png" alt="Members page with two members and two pending invitations"><br><b>Members</b> and pending invitations</td>
<td><img src="docs/screenshots/roles.png" alt="Roles page with Owner, Admin, Contributor, Member and Viewer"><br><b>Roles</b> built from permissions</td>
</tr>
<tr>
<td><img src="docs/screenshots/role-permissions.png" alt="Edit role dialog with the Contributor role's permissions ticked"><br><b>Permissions</b> grouped by area</td>
<td><img src="docs/screenshots/audit-log.png" alt="Audit log with project and AI summary entries"><br><b>Audit log</b> with before/after values</td>
</tr>
<tr>
<td><img src="docs/screenshots/billing.png" alt="Billing page on the Pro plan with a card on file and a $10 AI credit pack"><br><b>Billing</b>: plans and AI credit packs</td>
<td><img src="docs/screenshots/invoices.png" alt="Invoices page with a Pro subscription invoice and two AI credit invoices, all paid"><br><b>Invoices</b>, read live from Stripe</td>
</tr>
<tr>
<td><img src="docs/screenshots/account.png" alt="Account settings with two-factor authentication on and three signed-in devices"><br><b>Account</b>: profile, 2FA and devices</td>
<td><img src="docs/screenshots/two-factor-setup.png" alt="Two-factor setup dialog with a QR code, the key and a 6-digit code input"><br><b>Two-factor setup</b> with any authenticator app</td>
</tr>
<tr>
<td><img src="docs/screenshots/select-organization.png" alt="Organization picker after login, listing three organizations with the last used one marked"><br><b>Pick a workspace</b> after login</td>
<td><img src="docs/screenshots/admin-users.png" alt="Platform admin user search with unverified, suspended, two-factor and admin badges"><br><b>Admin</b>: search every user</td>
</tr>
<tr>
<td><img src="docs/screenshots/admin-user-actions.png" alt="A user's admin page with the View as user, Reset two-factor, Sign out everywhere and Suspend actions"><br><b>Admin actions</b>, each with a reason</td>
<td><img src="docs/screenshots/admin-view-as.png" alt="A project page viewed as a user, with the read-only banner and its content hidden"><br><b>View as user</b>: read-only, content hidden</td>
</tr>
<tr>
<td colspan="2" align="center"><img src="docs/screenshots/admin-organization.png" alt="Platform admin page of an organization on the Pro plan with its AI budget, counts and members" width="60%"><br><b>An organization through an admin's eyes</b>: plan, budget, counts, members. No project content.</td>
</tr>
</table>

## ⚙️ Configuration

<details>
<summary><b>Environment keys in <code>api/.env</code></b></summary>

| Key | Default | What it's for |
|---|---|---|
| `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` | `saas_starter`, `postgres`, empty | The app database |
| `FRONTEND_URL` | `http://localhost:3000` | The only origin CORS allows, and the base of invitation links |
| `QUEUE_CONNECTION` | `database` | Invitation emails and AI summaries are queued |
| `MAIL_MAILER` | `log` | Locally, emails (with their links) go to `storage/logs/laravel.log` |
| `ANTHROPIC_API_KEY` | empty | Claude API key, used only by the API. Without it, summaries are marked failed |
| `AI_PROVIDER` | `anthropic` | `fake` returns canned answers with no network calls (no key needed) |
| `AI_MODEL` | `claude-haiku-5-5` | Model for all AI features |
| `AI_RATE_LIMIT_PER_MINUTE` | `5` | AI requests per organization per minute |
| `AI_MONTHLY_BUDGET_USD` | `5` | Monthly AI budget per organization on Free |
| `AUDIT_RETENTION_DAYS` | `365` | Days to keep audit entries (organization and platform); `0` keeps them forever |
| `ADMIN_TWO_FACTOR_TTL_MINUTES` | `720` | Admin pages ask for a fresh 2FA code once the last one on that device is older than this |
| `ADMIN_ACTION_TWO_FACTOR_TTL_MINUTES` | `15` | Admin actions (suspend, reset 2FA, …) ask for a fresher code |
| `ADMIN_IMPERSONATION_MINUTES` | `30` | Length of a read-only "view as user" session (not renewed) |
| `BILLING_PROVIDER` | `stripe` | `fake` makes no Stripe calls: checkout returns straight to the billing page and changes nothing |
| `STRIPE_KEY`, `STRIPE_SECRET` | empty | Stripe API keys (test mode locally), used only by the API |
| `STRIPE_WEBHOOK_SECRET` | empty | The webhook signing secret. Without it every webhook is refused |
| `STRIPE_PRICE_PRO` | empty | Price id of the recurring Pro price. Without it, upgrading is refused |
| `BILLING_PRO_AI_BUDGET_USD` | `50` | Monthly AI budget on Pro |
| `STRIPE_PRICE_CREDITS_10` | empty | Price id of the one-time "$10 of AI credits" price. Without it, buying credits is refused |

Everything else (AI timeouts, displayed prices, currency, mail sender, …) is documented in `api/.env.example`.

</details>

<details>
<summary><b>💳 Setting up Stripe (test mode)</b></summary>

Billing uses [Laravel Cashier](https://laravel.com/docs/billing) with the **organization** as the Stripe customer. Without Stripe, use `BILLING_PROVIDER=fake`: every organization stays on Free, and with any placeholder `STRIPE_PRICE_*`, Upgrade and Buy return to the billing page and change nothing.

1. Install the [Stripe CLI](https://docs.stripe.com/stripe-cli) and run `stripe login`.
2. Get the webhook secret for `STRIPE_WEBHOOK_SECRET`: `stripe listen --print-secret`.
3. Create the two prices and copy each `price_...` id:
   ```bash
   # STRIPE_PRICE_PRO: $19 a month
   stripe prices create --unit-amount=1900 --currency=usd -d "recurring[interval]=month" -d "product_data[name]=Pro"
   # STRIPE_PRICE_CREDITS_10: $10 one time
   stripe prices create --unit-amount=1000 --currency=usd -d "product_data[name]=AI credits"
   ```
4. Turn on the Customer Portal (Dashboard → Settings → Billing → Customer portal): allow updating payment methods, viewing invoices and canceling at period end, with plan switching off.
5. Fill in `api/.env`:
   ```dotenv
   BILLING_PROVIDER=stripe
   STRIPE_KEY=pk_test_...
   STRIPE_SECRET=sk_test_...
   STRIPE_WEBHOOK_SECRET=whsec_...
   STRIPE_PRICE_PRO=price_...
   STRIPE_PRICE_CREDITS_10=price_...
   ```
6. Restart the API and forward webhooks while you test:
   ```bash
   stripe listen --forward-to localhost:8000/api/stripe/webhook \
     --events customer.subscription.created,customer.subscription.updated,customer.subscription.deleted,customer.updated,customer.deleted,payment_method.automatically_updated,invoice.payment_action_required,invoice.payment_succeeded,checkout.session.completed,checkout.session.async_payment_succeeded
   ```
7. Open http://localhost:3000/billing as an owner and pay with `4242 4242 4242 4242` (any future date, any CVC).

**Production:** use live keys and prices, and add a webhook endpoint for `https://<api>/api/stripe/webhook` with the same events. (`php artisan cashier:webhook` registers only Cashier's defaults, without the two `checkout.session.*` events that grant credits.) Configure the Customer Portal the same way, and under *Manage failed payments* set "If all retries for a payment fail" to cancel the subscription.

</details>

## ✅ Tests and CI

One command runs every check, and it finds PHP 8.4 and the right Node version for you:

```bash
.claude/skills/verify/verify.sh          # or: verify.sh api | verify.sh web
```

Or step by step:

```bash
cd api && php artisan test                     # Pest, against saas_starter_test
cd api && vendor/bin/pint --test app routes tests database config bootstrap/app.php bootstrap/providers.php
cd web && npm run lint && npm run typecheck && npm run build
```

- **~880 Pest tests**, and every tenant-owned feature proves org A can't read or write org B.
- **No real network in tests.** `FakeAiClient` and `FakeBillingGateway` stand in for Claude and Stripe.
- **GitHub Actions** runs the same checks on every push to `main` and every PR, with no secrets needed. To block merging until they pass, require `API (Pint, Pest)` and `Web (lint, typecheck, build)` in branch protection.

## 🗂 Project layout

```
api/                         Laravel API
├─ app/Actions/              One class per business action (thin controllers)
├─ app/Actions/Admin/        The only cross-organization code
├─ app/Ai/                   AiClient interface, Claude client, budget, prompts
├─ app/Billing/              Plans, credit packs, Stripe gateway (+ fake)
├─ app/Http/                 Controllers, Form Requests, Resources, middleware
├─ app/Models/               Eloquent; tenant models use BelongsToOrganization
├─ app/Policies/             Permission checks
├─ app/Support/              Current organization, TOTP, recovery codes, impersonation
├─ config/                   ai.php · billing.php · admin.php
└─ tests/Feature/            Pest, including org A vs org B isolation
web/app/                     Nuxt app
├─ pages/                    Projects, members, roles, audit log, AI usage, billing, account, admin/…
├─ components/               Org switcher, AI summary card, chat, two-factor, admin actions…
├─ composables/useApi.ts     The single gateway to the API (JSON + streaming)
└─ stores/                   Pinia: current user and organization
docs/plans/                  Task-by-task build history (Proposed → WIP → Done)
.claude/                     Claude Code skills, agents and hooks
.github/                     CI workflow and Dependabot
```

### 🧩 Add your own feature

Follow how **Projects** is built: a model with `BelongsToOrganization`, a policy, Form Requests, an Action, a Resource, a page, and an isolation test. The `/tenant-feature` skill walks you through it.

## 📚 API and pages reference

<details>
<summary><b>Every endpoint and page, by feature</b></summary>

| Feature | API | Web |
|---|---|---|
| Register, log in, log out, current user | `POST /api/register`, `/login`, `/logout`, `GET /api/me` | `/register`, `/login`, `/select-organization` (for people in more than one organization) |
| Email verification. Until it's done, a banner shows and inviting, joining, billing and AI are refused (403 `email_unverified`) | `POST /api/email/verify/{id}/{hash}` (signed), `POST /api/email/verification-notification` | Banner on every page, `/verify-email` |
| Password reset: a link valid once for 60 minutes (same answer whether or not the account exists). Resetting signs out every device | `POST /api/forgot-password`, `POST /api/reset-password` | `/forgot-password`, `/reset-password` |
| Two-factor (TOTP): QR setup, 8 recovery codes, a code step after login or reset (5 min, 5 tries, single use). Emails on every change | `POST /api/account/two-factor`, `…/confirm`, `…/recovery-codes`, `DELETE /api/account/two-factor`, `POST /api/two-factor-challenge` | Two-factor card on `/account`, code step on `/login` and `/reset-password` |
| Account settings: name, email, password, devices, delete account (refused while you're the last owner of an organization with other members or a paid plan) | `PATCH /api/account`, `PUT /api/account/email`, `PUT /api/account/password`, `DELETE /api/account`, `GET/DELETE /api/account/devices…` | `/account` |
| Organizations: list, create, switch | `GET/POST /api/organizations`, `POST /api/organizations/{id}/switch` | Sidebar switcher |
| Members: list, change role, remove (always one owner) | `GET/PATCH/DELETE /api/members` | `/members` |
| Roles and permissions (only grant what you hold) | `GET/POST/PATCH/DELETE /api/roles`, `GET /api/permissions` | `/roles` |
| Invitations: invite, revoke, accept | `GET/POST/DELETE /api/invitations`, `GET/POST /api/join/{token}` | `/members`, `/join/{token}` |
| Projects: search, paginate, create, edit, delete | `/api/projects` (REST) | `/projects`, `/projects/{id}` |
| Audit log: filter by action, person and date. IPs shown to owners only. Pruned after `AUDIT_RETENTION_DAYS` | `GET /api/audit-logs`, `GET /api/audit-log/actions` | `/audit-log` |
| AI summaries: a draft on create (or on request), then approve (optionally edited) or reject, flagged when the project changes. Backfill with `php artisan ai:summarize-projects` (`--organization=ID`, `--limit=50`, `--dry-run`) | `GET/POST /api/projects/{id}/summary`, `POST …/summaries/{summaryId}/approve`, `…/reject` | Summary card on `/projects/{id}` |
| AI writing help: improve, expand or shorten. Streams in; stop, accept, retry or discard. Nothing is saved until the project is | `POST /api/projects/writing-help`, `POST /api/projects/{id}/writing-help` (SSE) | Project create and edit forms |
| Project chat: a private, streamed "Ask AI" about one project. Clearable, and deleted when you leave the organization | `GET`, `POST` (SSE), `DELETE /api/projects/{id}/chat` | Project page |
| AI usage and budget: calls, tokens and cost by feature, person and day. The budget comes from the plan plus credits, and owners can lower it | `GET /api/ai-usage?month=YYYY-MM`, `PUT /api/ai-budget` | `/ai-usage` |
| Billing: Free and Pro, Checkout, Portal, live invoices, $10 AI credit packs. Only the webhook changes plans, and plan changes are audited | `GET /api/billing`, `POST /api/billing/checkout`, `POST /api/billing/credits/checkout`, `POST /api/billing/portal`, `GET /api/billing/invoices`, `POST /api/stripe/webhook` | `/billing`, `/billing/invoices` |
| Platform admin lookups: users and organizations (25 a page). Detail views are logged in the platform log. Routes answer 404 to non-admins and need a recent 2FA code | `GET /api/admin/users[/{id}]`, `GET /api/admin/organizations[/{id}]`, `GET /api/admin/audit-log`, `POST /api/account/two-factor/reconfirm` | `/admin/users`, `/admin/organizations`, `/admin/audit-log` |
| Platform admin actions, each with a reason and a fresh code: reset 2FA, sign out everywhere, suspend or unsuspend, make Owner. Never on platform admins | `POST /api/admin/users/{id}/two-factor/reset`, `…/sign-out`, `…/suspend`, `…/unsuspend`, `POST /api/admin/organizations/{id}/owners` | Actions card on `/admin/users/{id}`, "Make Owner" on `/admin/organizations/{id}` |
| View as user: read-only for `ADMIN_IMPERSONATION_MINUTES`. Every change is refused except switching organization, private content is hidden, and the user is emailed | `POST /api/admin/users/{id}/impersonate`, `POST /api/logout` ends it | "View as user" button and "Stop viewing" banner |

Organization routes act on the user's **current** organization (`users.current_organization_id`). Records from other organizations return 404.

</details>

## 🤖 Built with Claude Code

This repo was built with [Claude Code](https://claude.com/claude-code) and ships the setup it used. [CLAUDE.md](CLAUDE.md) holds the rules.

- **Skills:** `/plan-task` · `/tenant-feature` · `/verify` · `/e2e-smoke` · `/ship-plan`
- **Read-only reviewers:** `code-reviewer`, `security-auditor`, `tenant-isolation-reviewer`, `database-reviewer`, `ui-reviewer`
- **Workers:** `flow-tester` (runs real user flows, emails included) and `laravel-test-writer` (limited to `api/tests/`)
- **Guard rails:** hooks block commands that wipe or drop database data
- **MCP:** Playwright, plus the Nuxt and Nuxt UI docs

The loop for a typical change: `/plan-task` → approve → build with tests → `/verify` → reviewers → `/ship-plan`.

## 🤝 Contributing

Start with [CLAUDE.md](CLAUDE.md) for the conventions, then browse [docs/plans/](docs/plans/README.md) to see how the project was built, one task at a time, including known follow-ups.

<div align="center">
<sub>Built with Laravel, Nuxt, Claude and Stripe · <a href="https://github.com/sanchaichie/laravel-react-multitenant-saas">Prefer React? See the React edition →</a></sub>
</div>
