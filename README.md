# Claude Code Skills: CodeIgniter 4

A trio of Claude Code skills for building with CodeIgniter 4. Designed to give Claude a solid, accurate understanding of the CI4 framework — including Shield authentication.

- **`ci4`** — Core CI4 framework: MVC conventions, routing, controllers, models, query builder, views, migrations, filters, Spark CLI.
- **`ci4-api`** — Building robust REST APIs with CI4: response envelopes, versioning, token auth, validation, rate limiting, CORS, and webhooks.
- **`ci4-shield`** — Shield authentication & authorization: session auth, access tokens, HMAC, JWT, groups, permissions, email activation, 2FA, magic links.

Each skill has a lean SKILL.md (~200-350 lines) that loads automatically, plus a `references/` directory with comprehensive deep-dive docs that are pulled on demand — keeping context usage efficient.

---

## Installation

Clone this repo directly into your Claude Code skills directory:

```bash
git clone https://github.com/enlivenapp/Codeigniter4-App-and-API-Skills-for-Claude-Code.git ~/.claude/skills/codeigniter4
```

Or if you already have other skills and want to add just these:

```bash
cd ~/.claude/skills
git clone https://github.com/enlivenapp/Codeigniter4-App-and-API-Skills-for-Claude-Code.git
cp -r Codeigniter4-App-and-API-Skills-for-Claude-Code/ci4 .
cp -r Codeigniter4-App-and-API-Skills-for-Claude-Code/ci4-api .
cp -r Codeigniter4-App-and-API-Skills-for-Claude-Code/ci4-shield .
```

Restart Claude Code. The skills are now available. Claude will automatically activate the relevant skill when it detects CI4-related work.

---

## What's Included

### ci4 (12 reference docs)

| Reference | Coverage |
|---|---|
| `routing.md` | Groups, placeholders, named routes, resource routes, CLI routes |
| `controllers.md` | ResourceController, request data, file uploads, validation, redirects |
| `models.md` | CRUD, soft deletes, pagination, scopes, callbacks, entities |
| `query-builder.md` | SELECT, WHERE, JOIN, batch ops, subqueries, transactions |
| `views.md` | Layouts, sections, partials, view cells, caching |
| `filters.md` | Creating, registering, global filters, arguments |
| `database.md` | Migrations, seeds, forge, field types |
| `validation.md` | All built-in rules, custom rules, file upload validation |
| `services-helpers.md` | Services, helpers, caching, email, sessions, encryption, HTTP client |
| `spark-cli.md` | All spark commands, generators, custom commands |
| `testing.md` | PHPUnit, feature tests, database tests, mocking |
| `gotchas.md` | Critical pitfalls, CI4 vs Laravel comparison |

### ci4-api (6 reference docs)

| Reference | Coverage |
|---|---|
| `base-controller.md` | BaseApiController, response helpers, auth helpers |
| `authentication.md` | Login/register/logout endpoints, token lifecycle, filters |
| `webhooks.md` | Receiving (Stripe, GitHub), sending, signature verification |
| `rate-limiting.md` | Throttle filter, per-route limits, inline throttling |
| `cors.md` | CORS filter, preflight, production config |
| `error-handling.md` | Status code guide, error envelope, global exception handler |

### ci4-shield (8 reference docs)

| Reference | Coverage |
|---|---|
| `configuration.md` | Auth.php, AuthGroups.php, password validators, JWT config |
| `session-auth.md` | Login/logout flow, remember me, checking auth state |
| `token-auth.md` | Access tokens, HMAC tokens, JWT, scopes, mobile login |
| `groups-permissions.md` | Groups, permissions, matrix, wildcards, direct permissions |
| `user-model.md` | User entity, UserModel, extending, CRUD, banning, force reset |
| `filters.md` | All Shield filters, route protection, filter order, chain filter |
| `actions.md` | Email activation, Email 2FA, magic links, password handling |
| `events-customization.md` | Events, custom views, extending controllers, routes, testing |

---

## Resources

- [CodeIgniter 4 User Guide](https://codeigniter.com/user_guide/)
- [CodeIgniter 4 Installation](https://codeigniter.com/user_guide/installation/installing_composer.html)
- [CodeIgniter Shield Docs](https://shield.codeigniter.com/)
- [CodeIgniter 4 on GitHub](https://github.com/codeigniter4/CodeIgniter4)
- [CodeIgniter Shield on GitHub](https://github.com/codeigniter4/shield)
