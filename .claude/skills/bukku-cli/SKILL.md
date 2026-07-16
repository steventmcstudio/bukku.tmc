---
name: bukku-cli
description: Use the Bukku CLI in this repo to create/manage quotations, invoices, contacts, and other Bukku accounting records for the tmcstudio company. Trigger on requests like "create a quotation for X", "list unpaid invoices", "add a customer to Bukku", or anything involving Bukku sales/purchase documents.
---

# Bukku CLI usage

This repo (`packages/cli`) is a CLI for the Bukku accounting API. It is not
pre-installed — build it once per session, then call it with credentials from
environment variables.

## Setup (once per session)

```bash
npm install
npm run build
```

This produces `packages/cli/build/index.js`, invoked as `node packages/cli/build/index.js <command>`.

## Credentials

The CLI needs `BUKKU_API_TOKEN` and `BUKKU_COMPANY_SUBDOMAIN` (subdomain is
`tmcstudio` for this account). **Never write the token into a file that gets
committed to this repo** (skill files, scripts, `.env` committed to git,
etc.) — that would leak a live credential into git history on a public/shared
remote.

The correct place for these is the cloud environment's **environment
variables** setting (same dialog as network access), so every session gets
them automatically without asking the user to paste the token again. Check
first:

```bash
echo "token set: $([ -n "$BUKKU_API_TOKEN" ] && echo yes || echo no)"
echo "subdomain: $BUKKU_COMPANY_SUBDOMAIN"
```

If unset, tell the user to add `BUKKU_API_TOKEN` and
`BUKKU_COMPANY_SUBDOMAIN` in the environment's **Environment variables**
field (edit environment → same place as Network access), rather than pasting
the token into chat/files. If they do paste it in chat as a one-off, use it
only via `export` in the shell for that session — do not persist it to disk.

## Required: Node proxy flag

**Always export `NODE_USE_ENV_PROXY=1` before running the CLI** in this
sandboxed environment:

```bash
export NODE_USE_ENV_PROXY=1
```

Node's built-in `fetch` does not read `HTTPS_PROXY` by default, so without
this flag every API call silently fails with a 403 body like
`Host not in allowlist: api.bukku.my` — this looks like a network-policy
block but is actually just Node bypassing the sandbox's egress proxy. Setting
`NODE_USE_ENV_PROXY=1` fixes it (confirmed working on Node 22.22). Only chase
an actual network-policy fix (adding `api.bukku.my` to the environment's
Custom allowed domains) if requests still fail with this flag set.

## Network access

If `api.bukku.my` truly isn't reachable (e.g. a fresh environment that was
never configured), tell the user: open the environment for editing (cloud
icon → settings icon), set Network access to **Custom**, add `api.bukku.my`
to Allowed domains (keep "include default list" checked for npm/GitHub
access needed to build this repo).

## Known reference data (tenant: tmcstudio)

Verify with a search before trusting these, since new contacts/products get
added over time — but these ids were confirmed in this tenant:

- Contact "SOLVENTUM MALAYSIA SDN BHD" → `contact_id: 2`
- Contact "3M MALAYSIA SDN. BHD." → `contact_id: 3`
- Account "Sales Income" (code 5000) → `account_id: 20`

```bash
node packages/cli/build/index.js contacts contacts list --search "<name>"
node packages/cli/build/index.js products products list --search "<name>"
node packages/cli/build/index.js accounting search-accounts --search "Sales"
```

## Creating a sales quotation

```bash
node packages/cli/build/index.js sales quotes create --data '{
  "contact_id": <id>,
  "date": "YYYY-MM-DD",
  "currency_code": "MYR",
  "exchange_rate": 1,
  "tax_mode": "exclusive",
  "status": "draft",
  "form_items": [
    {
      "account_id": 20,
      "description": "<item description>",
      "quantity": <qty>,
      "unit_price": <price>
    }
  ]
}'
```

`exchange_rate`, `tax_mode`, and `status` are required by the API even though
the CLI's `--help` doesn't say so — omitting them returns a 422 validation
error. Use `--dry-run` first to sanity-check the payload without submitting.

If no matching product exists, a line item with just `account_id` +
`description` (no `product_id`) works fine as a free-text line.

### ⚠️ Draft quotes can silently disappear

Observed in this tenant: creating a second **draft** quote before the first
draft is finalized can overwrite/delete the first one (both get assigned the
same provisional number, e.g. `QT-00012`, and the older record then 404s on
`get`). Bukku appears to hold only one provisional "next number" slot for
unfinalized drafts.

To avoid losing a quote you care about, finalize it right after creating it:

```bash
node packages/cli/build/index.js sales quotes status <id> --status ready
```

Note: `draft → ready` is one-way — a ready quote can only go to `void`
after that, never back to `draft`. Ask the user before flipping status on
something they might still want to edit, but warn them clearly about the
overwrite risk if they choose to leave it as a draft.
