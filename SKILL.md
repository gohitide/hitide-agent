---
name: hitide
description: HiTide is social SMS list growth — it turns your Instagram, TikTok & Facebook followers into SMS & email subscribers using automated DMs (keyword auto-responders). This CLI drives a HiTide workspace with a Workspace API key — read and manage your workspace, campaigns, connected accounts & integrations, collected subscribers, members, and metrics, all as JSON.
homepage: https://gohitide.com
metadata: {'requires': {'bins': ['hitide'], 'env': ['HITIDE_API_KEY']}}
---

# HiTide CLI

[HiTide](https://gohitide.com) is **social SMS list growth**: it converts Instagram, TikTok & Facebook followers into SMS & email subscribers using automated DMs. `hitide` is a thin, JSON-emitting wrapper over the HiTide API — every command is scoped by the server to the single workspace that owns your API key and prints JSON, so output pipes cleanly into `jq`.

## 1. Install the CLI if it doesn't exist

```bash
command -v hitide >/dev/null 2>&1 || npm install -g @hitide/cli
```

## 2. Authenticate first (hard rule)

Commands (except `auth:*`) need a **Workspace API key** (starts with `ws_`). The key identifies the workspace — there is no workspace argument.

**Get a key:** at [gohitide.com](https://gohitide.com), log in → **Settings → API Keys → Manage API Keys → Create API Key**.

```bash
export HITIDE_API_KEY=ws_xxx          # preferred for agents / CI
hitide auth:login --api-key ws_xxx    # or store it once instead of the env var
hitide auth:status                    # verify the key & see which workspace it connects to
```

`auth:status` calls the API and returns `{authenticated, source, workspace, key}`. If `authenticated` is `false`, stop and resolve auth. When it succeeds it prints the **workspace name** — confirm it's the account you expect before doing real work.

## 3. Commands

All commands accept `--json` for compact machine output (default is pretty-printed JSON). Paths (like `<campaignPath>`) come from the `path` field of the corresponding list command.

**Workspace & metrics**
| Command | Returns / does |
| --- | --- |
| `hitide workspace:get` | Workspace details (name, members, onboarding, integrations) |
| `hitide workspace:checkup` | Integration & account health errors |
| `hitide workspace:update --data <json\|@file>` | Update workspace settings (send the full settings object) |
| `hitide metrics:dashboard` | Dashboard metrics |
| `hitide metrics:usage [--month YYYY-MM]` | Subscriber + revenue usage |
| `hitide billing:subscription` | Active subscription / plan |

**Campaigns (keyword auto-responders)**
| Command | Returns / does |
| --- | --- |
| `hitide campaigns:list` | **All** published campaigns (auto-paginated across every page) as `{items}`, each with `name` |
| `hitide campaigns:status` | **All** published campaigns grouped into `{active, past}` (matches dashboard), each with name, path & metrics `summary` |
| `hitide campaigns:drafts` | Draft campaigns |
| `hitide campaigns:get <campaignPath>` | One campaign |
| `hitide campaigns:optin-links` | SMS opt-in links for all campaigns |
| `hitide campaigns:save <campaignPath> --data <fields>` | Update a campaign — pass only the fields to change, e.g. `'{"name":"New"}'`. Use `--workspace <path>` (no path arg) to create a new one |
| `hitide campaigns:enable <campaignPath>` | Activate a campaign |
| `hitide campaigns:disable <campaignPath>` | Deactivate a campaign |
| `hitide campaigns:duplicate <campaignPath>` | Duplicate a campaign |
| `hitide campaigns:remove <campaignPath>` | Delete a campaign |

**Accounts, integrations & content**
| Command | Returns / does |
| --- | --- |
| `hitide accounts:list` | Connected social accounts (no tokens) |
| `hitide content:recent <accountPath>` | Recent posts/stories/reels for an account |
| `hitide integrations:subscribers` | Connected SMS/email integrations (secrets redacted) |
| `hitide integrations:advertising` | Connected Meta/TikTok ad accounts (secrets redacted) |
| `hitide integrations:get <integrationPath>` | One integration (secrets redacted) |
| `hitide integrations:revenue` | Connected revenue (Shopify) integration — Settings "Revenue" (`{integration:null}` if none) |
| `hitide integrations:save-config <integrationPath> --data <json>` | Save an integration's routing config |

**Subscribers & collected contacts**
| Command | Returns / does |
| --- | --- |
| `hitide subscribers:search --phone <p> \| --email <e>` | Search collected subscribers |
| `hitide subscribers:get --phone <p> \| --email <e>` | Merged subscriber profile |
| `hitide collected:list <campaignPath>` | Contacts collected by a campaign |
| `hitide collected:pending <campaignPath>` | Contacts waiting for approval |
| `hitide collected:winners <campaignPath> [--regenerate]` | Pick random winners |

**Members & keys**
| Command | Returns / does |
| --- | --- |
| `hitide members:list` | Team members, each with `role` ("Workspace Owner" / "Regular Member") |
| `hitide members:remove <uid>` | Remove a member |
| `hitide keys:list` | API-key metadata (never secrets) |

Discover interactively: `hitide --help`, `hitide campaigns --help`.

## 4. Metrics glossary — the numbers match the UI

`metrics:dashboard`, `campaigns:get` and `campaigns:status` include a computed **`summary`** block whose numbers equal what you see at app.gohitide.com. **Prefer `summary`**; raw fields are kept for detail.

Per-campaign `summary` (`campaigns:get` / `campaigns:status`) — mirrors the campaign **breakdown** screen (`/breakdown/<id>`):
| `summary` field | UI label | computed from raw fields |
| --- | --- | --- |
| `keywordsSubmitted` | SOCIAL "Keywords Submitted" | `metrics.uniqueContacts` |
| `accountsReached` / `likes` / `comments` | SOCIAL card | `metrics.social.*` |
| `uniqueSubscribers` | "Unique Subscribers" | `netNew + reactivated + newFromCustomers + existing + legacyExistingOrReactivated` |
| `newSubscribers` / `existingSubscribers` / `unsubscribed` | "Net New" / "Existing" / "Unsubscribed" | new = `netNew + reactivated + newFromCustomers`; existing = `existing + legacyExistingOrReactivated` |
| `phonesAndEmailsCollected` | "Phones & Emails Collected" | `metrics.totalCollected` |
| `optInRate` | "Opt-In Rate" | `uniqueSubscribers / keywordsSubmitted` |
| `orders` / `revenue` | "Orders" / "Revenue" | `revenue.newCustomers + returningCustomers`, or `null` if revenue isn't tracked |
| `averageOrderValue` | "Average Order Value" | `revenue / orders` (`null` if no orders / not tracked) |

Workspace `summary` (`metrics:dashboard`):
| `summary` field | UI label | computed from raw fields |
| --- | --- | --- |
| `keywordsSubmitted` | SOCIAL "Keywords Submitted" | `social.uniqueContacts` |
| `accountsReached` / `likes` / `comments` | same | `social.*` |
| `campaignSubscribers` | "Campaign Subscribers" | `netNew+reactivated+newFromCustomers+existing+legacyExistingOrReactivated` |
| `newSubscribers` / `existingSubscribers` | "New" / "Existing" | `(netNew+reactivated+newFromCustomers)` / `(existing+legacyExistingOrReactivated)` |
| `revenue` / `orders` | REVENUE total | sum of new + returning customers; `null` when not tracked (UI shows "N/A") |

⚠️ **`uniqueContacts` is NOT a subscriber count** — it is the UI's "Keywords Submitted". For subscribers use `summary.uniqueSubscribers`.

⚠️ **Revenue `null` ≠ `0`.** When a summary shows `revenue`/`orders`/`averageOrderValue` as `null`, revenue is **not tracked** (no Shopify integration) and the UI shows **"N/A"** — do not report it as "$0 earned". A real `0` (buckets present) means tracked-but-zero.

**`legacyExistingOrReactivated`** is a historical raw subscriber bucket. The UI (and `summary`) fold it into **"Existing"** — it is already included in `existingSubscribers` / `uniqueSubscribers`; never add it again.

`summary.tasks` = the UI's **"Tasks"** (the setup checklist, e.g. "Add the Klaviyo list ID"), mirroring the campaign's raw `flags`. Each task may be `optional: true` (a suggestion) or required (blocking). These can lag the editor, which recomputes them live.

**Vocabulary (CLI ↔ UI):** campaign groups `active`/`past` = "Active"/"Past" (per-campaign toggle "Enabled"); `campaigns:remove` = Delete; `collected:winners` = "Random Winner Selector"; `integrations:advertising` = "Ad Managers"; `integrations:revenue` = Settings "Revenue" (Shopify). Note: `integrations:subscribers` = the Settings "Subscribers" section (SMS/email _providers_), which is different from `subscribers:search`/`subscribers:get`, which find individual _people_.

## 5. Examples

```bash
# Campaign paths, then read one and its collected contacts
hitide campaigns:list --json | jq -r '.items[].path'
P=$(hitide campaigns:list --json | jq -r '.items[0].path')
hitide campaigns:get "$P" --json
hitide collected:list "$P" --json

# Patch a campaign: send only the fields that change
hitide campaigns:save "$P" --data '{"name":"Renamed"}'
hitide campaigns:disable "$P"     # flip it off
hitide campaigns:enable "$P"      # flip it back on
```

## 6. Gotchas

- **Keys start with `ws_`.** `403 "Invalid API key"` = wrong/revoked key.
- **Workspace scope only.** The key sees exactly one workspace; admin surfaces (bulk email, customer-success dashboards, cross-workspace metrics) are not exposed and return `403 Forbidden`.
- **Paths, not ids.** `<campaignPath>` / `<integrationPath>` / `<accountPath>` are full paths from the matching list command.
- **Write payloads.** `campaigns:save <path> --data` patches only the fields you pass; `workspace:update --data` needs the full settings object (fetch `workspace:get`, edit, send back).
- Non-zero exit + one-line error on failure; everything else is JSON on stdout.
