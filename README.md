# ZenSched Pest-Control Reference Kit

A copy-pasteable setup for a 1–5 truck pest-control shop (general pest routes, rodent, mosquito, wildlife, commercial, plus termite inspections and treatments) that wants an AI assistant to run route scheduling, GPS-verified treatment records, a local chemical-log extract, and invoicing. ZenSched handles the live schedule, the technician's phone app, GPS check-ins at the property, and the photo-plus-checklist Treatment Record. A small local database on your computer holds your customers, properties, prices, cadence, job summaries, and invoices.

**You do not need to know how to program or write SQL to use this.** You type plain English to your AI assistant ("schedule this week", "add a monthly account", "what did Luis put down at Delgado", "who owes me money?") and the AI does the work using two tools you set up once. Setup takes about 15 minutes and is the only technical part.

If you *are* a developer, skip to [For developers](#for-developers).

## What this kit is not — read this first

**What it is:** GPS-verified proof that a tech was at the address, a Treatment Record (pests seen, products, amount, areas, photos, follow-up), a local extract of those records for your own files, and invoices built from completed stops.

**What it is not:**

- **Not an official pesticide-use log.** `chemical_log` is *your* copy of what the tech typed on the phone (date, property, product, amount, tech). It is not a state applicator report, not an EPA log, and not a substitute for whatever your department of agriculture requires you to keep. Florida FDACS, California DPR, and every other state still want *their* form. This kit does not produce it and does not claim you are compliant because you used it.
- **Not WDO graphing.** A termite inspection in this kit is the same Treatment Record with `pests_seen` = Termites and photos. It is not an NPMA-33, not a Florida WDO-10, not a graph of the structure, and not a real-estate wood-destroying-organism report. If a lender or a buyer needs a WDO, you still fill out the state form.
- **Not a signed legal document.** The Treatment Record has no signature field. On ZenSched a signature field replaces the Submit button, so adding one would make every stop look like the tech (or the customer) had signed something. Submitting the form is just submitting the form.

If any of those is a deal-breaker, this kit is not for you. If you want route cadence, door-GPS, and a local extract you can file next to your real state forms, read on.

## What lives where

**ZenSched (source of truth for what happened, when, and where):**

- Locations (customer properties with GPS coordinates; the check-in radius is a **policy** setting)
- Workers (technicians with the mobile app)
- Events (one "Pest control" job per property, renewed every 60 days)
- Shifts (each scheduled stop, with push notifications to the tech)
- GPS punches (check-in/check-out with distance-from-the-pin verification)
- The Treatment Record form (pests, products, amount, areas, up to 3 photos, conditions, notes, follow-up) and every submission
- Timesheets (verified hours worked)

**Local SQLite database (`pest-ops.db`, on your computer):**

- Customer contact, per-visit rate, frequency (weekly / biweekly / monthly / quarterly / on-demand), next service date
- Properties, including access notes (gate code, crawl hatch, dog) that **never leave your computer**
- Your price list (general pest, rodent, termite inspect, termite treat, mosquito, wildlife, commercial)
- Technicians, including applicator license numbers that **never leave your computer**
- Completed jobs with a summary of each Treatment Record, the chemical-log extract, and invoices
- Your settings (timezone, default tech, invoice prefix, Treatment Record form id)

**Never duplicated:** the live schedule, punches, timesheets, and report photos stay in ZenSched. The local database only stores *references* to them plus a short per-job summary so you can answer "what did we put down at Delgado" without paying to re-read reports.

### Privacy note

Gate codes, crawl-hatch locations, alarm words, and applicator license numbers are stored only in `properties.access_notes` and `technicians.license_no` in the local database. `SKILL.md` forbids the AI from putting them into any ZenSched field. Give them to your tech yourself, by whatever channel you trust. Customer names, phones, and emails also stay local: ZenSched locations and events are named by street address (`1842 Palmetto Court, Tampa`), so ZenSched only ever sees the street address and the GPS pin.

## How it works day to day

Your AI assistant has two sets of tools:

1. **ZenSched tools** (`location_create`, `shift_create`, `form_submissions`, `shift_list`, ...) that talk to ZenSched over the internet.
2. **A SQLite tool** (`sqlite_query`, `sqlite_execute`) that reads and writes `pest-ops.db` on your computer.

When you say "schedule this week," the AI reads who is due from the local database (`next_service_date` in the next 7 days), creates one shift per stop on ZenSched, and tells you what it did. Your tech sees the stops in the app, checks in at the property (GPS-verified), treats, fills in the Treatment Record with photos, and checks out. Later you say "record this week's jobs" and the AI pulls the completed shifts and records, saves a summary locally, advances each account's next date (monthly +1 month, quarterly +90 days, on-demand clears it), and flags anything marked for follow-up. "Chemical log for September" is a local query. You never run SQL yourself. `SKILL.md` in this repo is the instruction sheet that teaches the AI how to do all of this; you paste it into your AI tool once.

A typical stop costs about **$0.35** on ZenSched: GPS in $0.10 + GPS out $0.10 + reading a Treatment Record that has photos $0.15. Geocoding a new property is $0.03 once. The AI states the cost before it spends.

## Setup

### 0. What you need

- **An AI tool that supports MCP.** These instructions use Claude Desktop (Windows or Mac). Cursor works too.
- **Node.js 20 or newer.** The SQLite tool runs on it. Download the LTS installer from [nodejs.org](https://nodejs.org/) and run it with the defaults. This is the only software install.
- You do **not** need the `sqlite3` command-line program, Python, or Git.

### 1. Make a folder for your data

Create a folder where the database will live and write down its full path. Examples:

- Windows: `C:\Users\YourName\pest-ops`
- Mac: `/Users/yourname/pest-ops`

The database file will be created automatically inside this folder the first time the AI uses it.

### 2. Add both tools to your AI's config file

Open the MCP configuration file for your AI tool:

- **Claude Desktop, Windows:** `%APPDATA%\Claude\claude_desktop_config.json` (paste that into the File Explorer address bar)
- **Claude Desktop, Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json` (in Claude Desktop: Settings → Developer → Edit Config)
- **Cursor:** Settings → MCP → Add new global MCP server

Paste in the contents of `mcp.json.example` from this repo, then change one line, the `SQLITE_PATH`, to point at your folder from step 1 plus `\pest-ops.db` (Windows) or `/pest-ops.db` (Mac):

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

**Windows path gotcha:** inside a JSON file every backslash must be doubled. Write `"C:\\Users\\YourName\\pest-ops\\pest-ops.db"`, not `"C:\Users\..."`. A single backslash will silently break the config.

**Leave `zsc_your_key_here` exactly as it is for now.** You do not have a key yet. The ZenSched tools that create your account work without one, and you will fill this in during step 3.

Save the file and **fully quit and reopen** your AI tool (on Mac, Cmd-Q; on Windows, right-click the tray icon → Quit). It only reads this file on startup.

### 3. Create your ZenSched account

In a new chat, type:

> Call `zensched_guide`, then call `account_create` with org_name "My Pest Control Co" (use my real business name if I told you one). Show me the `zsc_` key it returns.

Copy the `zsc_` key. Go back to the config file from step 2, replace `zsc_your_key_here` with your real key, save, and fully quit and reopen the AI tool again.

Some clients can adopt the key mid-session with `account_use_key`; you can ask the AI to try that to keep going immediately, but still update the config file so the key survives restarts. Keep the key private; it is the password to your account.

### 4. Create the database tables

Open `schema.sql` from this repo in any text editor, copy the whole thing, and paste it into the chat with this message in front of it:

> Create these tables in my pest-ops database. Run each statement one at a time using the SQLite tool, then list the tables to confirm.

The AI will run the statements one at a time and confirm the tables exist. The `pest-ops.db` file now exists in your folder, pre-loaded with a starter price list you can change.

If you happen to have the `sqlite3` command-line tool, `sqlite3 pest-ops.db < schema.sql` does the same thing, but it is not required.

### 5. Teach the AI the workflow

Paste the contents of `SKILL.md` into your AI tool as standing instructions. In Claude Desktop, create a Project and put it in the project instructions; in Cursor, save it as a rule. Then tell it your basics once:

> My business is GreenShield Pest in Tampa, Florida (Eastern time). Save that in settings, and set up the Treatment Record form.

It writes those to the `settings` table, creates the Treatment Record form on ZenSched (free), and saves the form id so every stop gets it automatically.

**Check-in radius.** The default pin uses `checkin_radius_m=75` on `location_create`, but ZenSched **enforces** the radius through the account's policy, not per property. With geofencing on it raises anything under 100 m to about 91 m (300 ft), so 75 behaves as roughly a house-and-driveway circle. For a warehouse, a large lot, or a pin that lands on the road, ask the AI to "set the check-in radius to 200 m" (`policy_update`) or to move the pin onto the building (`location_update`, free). Do not ask it to widen the radius "on that location" — that field is informational only.

### 6. Funding (only when asked)

The first 200 ZenSched tool calls per day are free. Some things are metered: creating a location (geocoding, $0.03), inviting a technician ($0.25), each GPS-verified check-in or check-out ($0.10), and reading a Treatment Record ($0.05, or $0.15 when it has photos). When a metered call happens without funds, the AI will get a `payment_required` response and tell you how to add the $5 activation deposit, which is credited to your balance. You will not be charged without seeing this first.

A typical stop is about $0.35 (in + out + photo record). A tech doing 8 stops a day is about $2.80 in meters that day, plus $0.03 the first time you add each property. The AI states the cost before it spends.

## Using it

Everything after setup is plain English. Examples:

- "Add Rosa Delgado, rosa@example.com, 813-555-0144, 1842 Palmetto Court, Tampa FL 33609. Monthly general pest $89 starting Monday. Gate code 3319, dog in the backyard. Ants at the kitchen and patio."
- "Add a one-off termite inspect for Maya Chen at 410 Bayshore Blvd, Tampa, Tuesday 11, $150."
- "Invite Luis Ortega, luis@example.com, and make him the default tech."
- "Schedule this week for Luis."
- "Record this week's jobs."
- "Chemical log for last week."
- "Draft invoices for everyone with uninvoiced work."
- "Who still owes me money?"
- "Rosa paid INV-2026-0001."
- "Pause the Delgado account until November."
- "Add a quarterly commercial at the warehouse on 22nd."

See `QUICKSTART.md` for the first-week walkthrough and `example-workflow.md` for exactly which tools the AI calls behind each of these.

### What "invoice" means here

"Draft an invoice" records the invoice in your database (number, date, due date, amount, which jobs) and the AI writes out a plain-text invoice you can paste into an email or text message, with a line per stop and a note that the visit was GPS-verified. It does **not** generate a PDF, email it for you, or collect payment. Invoices do not list mix rates or license numbers unless you ask. When the customer pays, tell the AI ("Rosa paid INV-2026-0001") and it marks it paid. If you outgrow this, the invoice records are simple enough to import into any accounting tool.

## Mobile app for technicians

- **Android:** [Google Play](https://play.google.com/store/apps/details?id=com.zensched.app)
- **iOS:** [TestFlight](https://testflight.apple.com/join/Wp51m5Yq)

When you invite a technician, they get an email, install the app, and can immediately see their stops, check in and out with GPS verification, and fill in the Treatment Record with photos. The record is attached to each stop automatically. There is no signature step — they tap Submit.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| AI says it has no ZenSched tools | Config file not saved, or the app was not fully restarted | Check the JSON is valid (paste it into [jsonlint.com](https://jsonlint.com)), then quit and reopen the app |
| AI says it has no SQLite / `pest-ops-db` tools | Node.js not installed, or bad `SQLITE_PATH` | Install Node.js LTS; on Windows check every backslash is doubled |
| `SQLITE_PATH` points nowhere / "unable to open database" | Folder from step 1 does not exist | Create the folder; the file is created automatically but the folder is not |
| ZenSched tools return an auth error | Key still says `zsc_your_key_here`, or was pasted with a space | Re-paste the key, restart |
| `payment_required` | Metered call with no balance | Follow the instructions in the response; $5 deposit |
| AI creates shifts at the wrong hour | Timezone not set | "Set my timezone offset to -04:00 in settings" (use your own offset) |
| Shift creation fails for dates a couple of months out | The property's 60-day ZenSched event has expired | Say "renew the events"; the AI runs the roll-over in `SKILL.md` and retries |
| Tech's check-in not GPS-verified at a house | Geocoded pin is at the mailbox, tech parked far away, or a large lot | Ask the AI to widen `checkin_radius_m` with `policy_update` (not on the location), or run `location_update` / `location_refine` ($0.10) |
| Tech does not see the Treatment Record | Form not assigned to that property's event | "Attach the Treatment Record to Delgado's event" (`form_assign`) |
| "Chemical log" comes back empty | Jobs not recorded yet, or the tech marked products None (inspect-only) | "Record this week's jobs" first; inspect-only rows are omitted on purpose |
| AI asks you to run SQL yourself | It does not have `SKILL.md` loaded | Re-paste `SKILL.md` as project instructions |
| AI refuses to put a gate code in ZenSched | Working as intended | Give it to the tech directly |
| AI offers a WDO graph or a state chemical form | It shouldn't | This kit does not produce those; use your state form |

If something is confusing or broken in ZenSched itself, ask the AI to call `feedback_submit` with a description. It is free, needs no account, and a human reads every submission.

## For developers

**Architecture.** Two MCP servers, no application code. The agent is the integration layer; `SKILL.md` is the spec it follows. ZenSched is authoritative for operations (schedule, punches, forms); SQLite is authoritative for CRM, cadence, treatment summaries, the chemical-log extract, and billing; each side stores only the other's **integer** IDs, plus a per-job report summary cached locally because submission reads are metered.

**Data model decisions.**

- One ZenSched **location** per property, permanent, stored on `properties.zensched_location_id` as an integer. Created with `location_create(name="<street>, <city>", street_address=..., checkin_radius_m=75, idempotency_key=...)`; the name is the street address, never the customer's name (customer PII stays in SQLite). `checkin_radius_m` on `location_create` is informational; the enforced radius is `policy_update(0, '{"checkin_radius_m": N}')`, and with geofencing on the platform raises values under 100 m to 300 ft.
- **Events are capped at 60 days by ZenSched**, so an event cannot be a permanent job template the way it is in the original lawn kit. Each property holds its *current* event in `properties.zensched_event_id` and its last covered date in `properties.event_valid_until`. The agent creates a new event (`event_create(location_id, title="Pest control - <street>", start_date, end_date=start+59 days, idempotency_key="event-property-{property_id}-{YYYYMMDD}")`) whenever a shift date is later than `event_valid_until`, calls `form_assign(form_id, event_id=...)` on it, and updates the row. `customers_due` exposes `event_needs_roll` per row and `events_expiring` lists properties due for renewal within 14 days. Shifts already created on the old event remain valid. When recording a completed job whose `event_id` no longer matches a property, the agent falls back to `event_get(event_id).location_id` against `properties.zensched_location_id`.
- **Cadence is next-service-date, not a weekday mask.** `customers.service_frequency` is `weekly | biweekly | monthly | quarterly | on-demand`. `customers_due` is every active customer with `next_service_date <= today+7` joined to their active properties, emitting `start_iso` / `end_iso` (preferred start or `settings.default_shift_start`, duration from the service or `default_shift_minutes`) and the shift `idempotency_key`. A customer with two active properties produces two rows on the same due date.
- **The `advance_service_date_on_job` trigger** sets `last_service_date` and `next_service_date` on every job insert: +7 / +14 / +1 month / **+90 days** / NULL. Quarterly is +90 days, not `+3 months`, so the interval does not drift with month length. Recording a one-off on a recurring customer also moves the cadence; `SKILL.md` tells the agent to set the date back if the owner says so.
- Rate lives on the **customer** (`service_rate`) so a shop can charge off-list. `customers.service_id` is the default; `jobs.service_id` is what was actually done (a monthly general-pest account can still get a `termite_inspect` job).
- `jobs.zensched_shift_id` and `technicians.zensched_worker_id` are integer `UNIQUE`. `jobs.report_dc_id` holds the form `submission_id`. `areas`, `conditions`, and `follow_up` are `CHECK`-constrained to the form's option labels. `pests_seen` / `products_used` store option keys as JSON; `product_summary` is the human string used on invoices and `chemical_log`.
- `fill_job_technician` sets `technician_id` from `zensched_worker_id` when the agent leaves it NULL.
- `invoices.invoice_number` is auto-assigned by trigger as `{prefix}-{YYYY}-{0001}`.
- **`chemical_log`** is a view over treated jobs (omits `products_used` in `[]` / `["none"]` / `none`). One row per job, not per product, because `amount_applied` is a single text field. It does not transmit anything and is not a state form.
- `properties.access_notes` and `technicians.license_no` are the columns that must never be sent to ZenSched; `SKILL.md` rule 6 enforces it. `customers_due` still *selects* `access_notes` so the agent can tell the owner to pass them to the tech.
- `PRAGMA foreign_keys = ON` is in `schema.sql` and `SKILL.md` tells the agent to run it per session; SQLite does not persist it.

**Treatment Record form.** Created once with `form_create(title, fields_json, idempotency_key="form-treatment-record")`; the exact `fields_json` is in `SKILL.md` and `example-workflow.md` (byte-identical) and was validated against ZenSched's `_validate_fields`. Every field carries an explicit `identifier` so submission `data` keys are stable (`pests_seen`, `products_used`, `amount_applied`, `areas`, `evidence`, `conditions`, `notes`, `follow_up`; section `sec_treatment`). Option keys are derived by ZenSched from the labels (lowercase, non-alphanumerics → `_`, truncated at 30 characters); every option here is well under 30 characters, so nothing truncates. **No `signature` field** — the phone keeps a Submit button, and submitting is not a legal attestation. Termite inspect reuses this form (`pests_seen=termites`). Attaching is `form_assign(form_id, event_id=...)`.

**Idempotency keys.** Deterministic, derived from local IDs:

- location: `loc-property-{property_id}`
- event: `event-property-{property_id}-{YYYYMMDD window start}`
- shift: `shift-property-{property_id}-{YYYYMMDD}`
- worker: `worker-{email}`
- form: `form-treatment-record`; assignment: `assign-treatment-record-{event_id}`

ZenSched caches idempotent responses for 24 hours.

**Timestamps.** `shift_create` takes `start` and `end` in ISO 8601 with an explicit offset. Always use the business's local offset from `settings.timezone_offset` (e.g. `2026-09-07T09:00:00-04:00`), never `Z`. The view builds these strings so the agent does not have to. The offset is a fixed setting, not a zone name, so it must be updated when daylight-saving time starts or ends (`SKILL.md` rule 8; `example-workflow.md` shows the November flip to `-05:00`).

**Metered reads.** `form_submissions` and `form_export` bill $0.05 per submission read ($0.15 with media); `form_export` is preferred for a week at a time. The kit stores the summary and media URLs on `jobs` on first read so later chemical-log questions are answered from SQLite. `shift_list`, `shift_status`, `event_get`, and `timesheet_export(mode="hours"|"raw")` are free.

**SQLite MCP server.** `mcp.json.example` uses [`easy-sqlite-mcp`](https://github.com/chenkumi/easy-sqlite-mcp) (Node, `better-sqlite3`, `SQLITE_PATH` env var). Its `sqlite_execute` calls `prepare()`, so it accepts **one statement per call**; `schema.sql` is written so every statement stands alone and is idempotent. Any SQLite MCP server with read and write tools will work; adjust the tool names in `SKILL.md`.

**Schema test.** The schema was verified by splitting the file into its 45 statements with `sqlite3.complete_statement` and executing each individually (as the MCP server does) twice for idempotency (seed rows not duplicated), then exercising: all 7 tables, 5 views, and 6 triggers present; every view on an empty database; the `advance_service_date_on_job` trigger for weekly (+7), biweekly (+14), monthly (+1 month), **quarterly (+90 days, 2026-09-07 → 2026-12-06, not +3 months / Dec 7)**, and on-demand (NULL); `fill_job_technician` from `zensched_worker_id`; `UNIQUE` on `zensched_shift_id`; every `CHECK` (frequency, `preferred_start`, areas, conditions, follow-up); `customers_due` `start_iso` / `end_iso` / `idempotency_key` / worker / minutes; `event_needs_roll` flipping exactly when `event_valid_until < next_service_date`; inactive and +20-day rows excluded; `events_expiring`; `chemical_log` omitting inspect-only `["none"]`; invoice numbering (auto `INV-2026-0001`, explicit number kept); `jobs_to_invoice` / `invoices_outstanding` filters; cascade delete and technician set-null; `updated_at`; integer types on ZenSched ID columns. Form payload validated against `_validate_fields` (9 fields, no signature, SKILL.md byte-identical to example-workflow.md, every option key ≤ 30 characters). 104 checks, all passing.

## Support

- ZenSched docs: <https://www.zensched.com/docs/>
- Tool reference: <https://www.zensched.com/docs/tools/>
- Feedback: ask your AI to call `feedback_submit` (categories: `bug`, `friction`, `missing_capability`, `docs`, `billing`, `feature`, `other`)

## License

MIT. See `LICENSE`.
