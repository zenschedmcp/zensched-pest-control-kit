# Quickstart

Setup is about 15 minutes, once. After that everything is plain English to your AI. Each step below tells you what to do and, where relevant, exactly what to type to the AI.

You need: Claude Desktop (or Cursor) and [Node.js LTS](https://nodejs.org/) installed. Nothing else.

Before you start, read the "What this kit is not" section of `README.md`. Short version: this is GPS-verified route proof plus a local extract of the Treatment Record. It is **not** a state pesticide-use log and **not** a WDO graph.

## 1. Make a data folder

Create a folder such as `C:\Users\YourName\pest-ops` (Windows) or `/Users/yourname/pest-ops` (Mac). Note the full path.

## 2. Add the two tools to your AI's config

Open the config file:

- **Claude Desktop, Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Claude Desktop, Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Cursor:** Settings → MCP → Add new global MCP server

Paste this in and fix only the `SQLITE_PATH` line to match your folder from step 1:

```json
{
  "mcpServers": {
    "zensched": {
      "url": "https://mcp.zensched.com/mcp",
      "headers": { "Authorization": "Bearer zsc_your_key_here" }
    },
    "pest-ops-db": {
      "command": "npx",
      "args": ["-y", "easy-sqlite-mcp"],
      "env": { "SQLITE_PATH": "/Users/yourname/pest-ops/pest-ops.db" }
    }
  }
}
```

- On Windows, double every backslash: `"C:\\Users\\YourName\\pest-ops\\pest-ops.db"`.
- Leave `zsc_your_key_here` as it is. You get the real key in the next step.

Save, then **fully quit and reopen** the AI app.

## 3. Create your ZenSched account

Type to the AI:

> Call zensched_guide, then account_create with org_name "GreenShield Pest". Show me the zsc_ key.

Copy the key into the config file in place of `zsc_your_key_here`. Save. Quit and reopen the app once more. (You can also ask the AI to call `account_use_key` with the key to continue right away, but update the file anyway so it sticks.)

## 4. Create the database tables

Copy the full contents of `schema.sql` and paste it into the chat with this line above it:

> Create these tables in my pest-ops database. Run each statement one at a time with the SQLite tool, then list the tables to confirm.

## 5. Give the AI its instructions

Paste `SKILL.md` into the AI as standing instructions (Claude Desktop: a Project's instructions; Cursor: a rule). Then:

> My business is GreenShield Pest in Tampa, Florida, Eastern time. Save that to settings and create the Treatment Record form.

The AI saves your settings and calls `form_create` once (free) to build the Treatment Record your techs fill in: pests seen, products used, amount applied, areas, up to 3 photos, conditions, notes, follow-up. No signature. It stores the form id so every stop gets it. Termite inspections use the same form (pests seen = Termites).

## 6. Add your first two customers

> Add Rosa Delgado, rosa@example.com, 813-555-0144, 1842 Palmetto Court, Tampa FL 33609. Monthly general pest $89 starting Monday 2026-09-07 at 9. Gate code 3319, dog in the backyard. Ants at the kitchen and patio.

> Add a one-off termite inspect for Maya Chen, maya.chen@example.com, 813-555-0190, 410 Bayshore Blvd, Tampa FL 33606, Tuesday 2026-09-08 at 11, $150. Crawl access through the east hatch.

Behind the scenes the AI inserts each customer and property, calls `location_create` (geocode, $0.03, may trigger the $5 activation deposit the first time), creates a 60-day `event_create` for the property, attaches the Treatment Record with `form_assign`, and saves the IDs. Gate / crawl notes go only into the local database. You just see a confirmation.

## 7. Invite your technician

> Invite Luis Ortega at luis@example.com as a tech and make him my default.

Luis gets an email ($0.25), installs the app ([Android](https://play.google.com/store/apps/details?id=com.zensched.app) / [iOS TestFlight](https://testflight.apple.com/join/Wp51m5Yq)), and activates. Give him the gate code and crawl hatch yourself; the AI will not put them in ZenSched.

## 8. Schedule the week

> Schedule this week for Luis.

The AI reads `customers_due`, creates one shift per stop on ZenSched, and summarizes by day. Luis gets a push notification for each, with the Treatment Record attached. It will confirm each stop is about $0.35 once he punches and you read the photo record.

## 9. After the work is done

> Record this week's jobs, show me the chemical log, then draft invoices for anyone with uninvoiced work.

The AI pulls the completed, GPS-verified shifts and the Treatment Records from ZenSched (reading records is metered, so it tells you the cost first), saves a per-job summary, advances Delgado to next month and clears Chen's on-demand date, shows the chemical-log extract (your copy, not a state form), creates invoice records, and writes out each invoice as text you can paste into an email.

> Rosa paid INV-2026-0001.

Marks it paid.

## What next

- `README.md` for the full explanation, the regulatory boundary, troubleshooting table, and developer notes
- `example-workflow.md` to see the exact tool calls behind each step above
