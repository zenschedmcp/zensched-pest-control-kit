# Example Workflow: What the AI Does Behind Each Request

This shows the exact tool calls the agent makes for a first week of operation, following `SKILL.md`. The owner only types the quoted lines; everything else is the agent's work. Assumes setup from `QUICKSTART.md` is complete (both MCP servers configured, `schema.sql` loaded, `SKILL.md` given as instructions).

IDs and responses are illustrative. ZenSched IDs are integers.

Shop: **GreenShield Pest**, Tampa, Florida, Eastern time (`-04:00` in September). Two customers (monthly general pest + one-off termite inspect), one technician, then invoice.

Access notes (gate code, crawl hatch) and any applicator license number stay in SQLite. They are never sent to ZenSched.

## Session start (every session)

```
sqlite_execute: PRAGMA foreign_keys = ON;
sqlite_query:   SELECT key, value FROM settings;
  → business_name = My Pest Control Co, timezone_offset = -05:00,
    default_worker_id = NULL, default_shift_start = 09:00,
    default_shift_minutes = 45, invoice_due_days = 14, invoice_prefix = INV,
    treatment_record_form_id = NULL, event_window_days = 60
```

## "My business is GreenShield Pest in Tampa, Florida, Eastern time."

```
sqlite_execute: UPDATE settings SET value = 'GreenShield Pest' WHERE key = 'business_name';
sqlite_execute: UPDATE settings SET value = '-04:00' WHERE key = 'timezone_offset';
```

> Saved. I'll schedule everything in Eastern time (-04:00). You don't have a Treatment Record form yet; it's free to create and every stop will come back with pests, products, amount, photos, and a follow-up flag. No signature — this is an internal stop record, not a state chemical form and not a WDO graph. Want me to set it up now?

Owner: *"yes"*

```
form_create:
  title: "Treatment Record"
  idempotency_key: "form-treatment-record"
  fields_json: (the JSON below, passed as one string)
```

```json
[
  {"type": "section", "label": "Treatment record", "identifier": "sec_treatment",
   "text": "Fill this in before you leave. Photos of activity or placement help. This is an internal stop record, not a state pesticide-use form and not a WDO graph."},
  {"type": "multi_select", "label": "Pests seen", "identifier": "pests_seen", "required": true,
   "options": ["Ants", "Roaches", "Rodents", "Termites", "Spiders", "Mosquitoes", "Other", "None"]},
  {"type": "multi_select", "label": "Products used", "identifier": "products_used", "required": true,
   "options": ["Alpine WSG", "Talstar", "Contrac", "Bait stations", "Glue boards", "None"]},
  {"type": "text", "label": "Amount applied", "identifier": "amount_applied",
   "placeholder": "e.g. 0.5 gal mixed / 0.3 oz Alpine"},
  {"type": "select", "label": "Areas treated", "identifier": "areas", "required": true,
   "options": ["Interior", "Exterior", "Both", "Attic", "Crawl"]},
  {"type": "photo", "label": "Evidence photos", "identifier": "evidence", "max_images": 3},
  {"type": "select", "label": "Conditions", "identifier": "conditions", "required": true,
   "options": ["Active", "No activity", "Monitor only"]},
  {"type": "textarea", "label": "Notes", "identifier": "notes"},
  {"type": "select", "label": "Follow-up", "identifier": "follow_up", "required": true,
   "options": ["None", "2 weeks", "30 days", "Quote treatment"]}
]
```

```
  → form_id: 401, fields: [ ...9 fields echoed with identifiers... ]

sqlite_execute: UPDATE settings SET value = '401' WHERE key = 'treatment_record_form_id';
```

> Treatment Record form created. I'll attach it to every property automatically. Termite inspections use this same form — the tech marks Termites. There is no signature step.

## "Add Rosa Delgado, rosa@example.com, 813-555-0144, 1842 Palmetto Court, Tampa FL 33609. Monthly general pest $89 starting Monday 2026-09-07 at 9. Gate code 3319, dog in the backyard. Ants at the kitchen and patio."

```
sqlite_query:   SELECT service_id, price, default_minutes FROM services WHERE code = 'general_pest';
  → 1, 85.00, 45

sqlite_execute:
  INSERT INTO customers (customer_name, contact_email, contact_phone, service_id, service_rate,
                         service_frequency, next_service_date, preferred_start)
  VALUES ('Rosa Delgado', 'rosa@example.com', '813-555-0144', 1, 89.00,
          'monthly', '2026-09-07', '09:00');
  → lastInsertRowid = 1

sqlite_execute:
  INSERT INTO properties (customer_id, address, city, state, zip, access_notes, pest_pressure)
  VALUES (1, '1842 Palmetto Court', 'Tampa', 'FL', '33609',
          'Gate 3319. Dog in backyard — wait for owner or crate.',
          'Ants at kitchen and patio');
  → lastInsertRowid = 1
```

Agent (first metered call this session, rule 11):

> Creating Rosa's address on ZenSched costs $0.03 for geocoding. If your account isn't funded yet you'll get a prompt for the $5 activation deposit. Go ahead?

Owner: *"yes"*

```
location_create:
  name: "1842 Palmetto Court, Tampa"
  street_address: "1842 Palmetto Court, Tampa, FL 33609"
  checkin_radius_m: 75
  idempotency_key: "loc-property-1"
  → location_id: 9101, lat: 27.9378, lng: -82.5154, pin_quality: "street"
    billing: { meter: "geocode", units: 1, price: 0.03 }
```

The gate code is **not** in `notes`, and Rosa's name is **not** in the location `name`: ZenSched gets the street address only; the customer record stays in SQLite. `checkin_radius_m` on the location is informational; the enforced radius is the policy (see later if Luis misses a punch).

```
event_create:
  location_id: 9101
  title: "Pest control - 1842 Palmetto Court"
  start_date: "2026-09-07"
  end_date:   "2026-11-05"          ← start + 59 days (60-day cap)
  idempotency_key: "event-property-1-20260907"
  → event_id: 7101

form_assign:
  form_id: 401
  event_id: 7101
  idempotency_key: "assign-treatment-record-7101"
  → status: "assigned", form_id: 401, event_id: 7101, policy_id: 0

sqlite_execute:
  UPDATE properties SET zensched_location_id = 9101, zensched_event_id = 7101, event_valid_until = '2026-11-05'
  WHERE property_id = 1;
```

> Added Rosa Delgado, 1842 Palmetto Court. Monthly general pest at $89, next stop Monday Sep 7 at 9:00. The gate code is saved on your computer only; give it to Luis directly. Her ZenSched event runs through Nov 5; I'll renew it automatically when scheduling gets that far.

## "Add a one-off termite inspect for Maya Chen, maya.chen@example.com, 813-555-0190, 410 Bayshore Blvd, Tampa FL 33606, Tuesday 2026-09-08 at 11, $150. Crawl access through the east hatch."

Same pattern. The agent does not re-ask about the $0.03 (owner already approved geocoding this session). Frequency is `on-demand`.

```
sqlite_query:   SELECT service_id, default_minutes FROM services WHERE code = 'termite_inspect';
  → 3, 90

sqlite_execute:
  INSERT INTO customers (customer_name, contact_email, contact_phone, service_id, service_rate,
                         service_frequency, next_service_date, preferred_start)
  VALUES ('Maya Chen', 'maya.chen@example.com', '813-555-0190', 3, 150.00,
          'on-demand', '2026-09-08', '11:00');
  → lastInsertRowid = 2

sqlite_execute:
  INSERT INTO properties (customer_id, address, city, state, zip, access_notes, pest_pressure)
  VALUES (2, '410 Bayshore Blvd', 'Tampa', 'FL', '33606',
          'Crawl access: east hatch. No gate.',
          'Buyer-requested termite inspect; not a WDO graph');
  → lastInsertRowid = 2

location_create:
  name: "410 Bayshore Blvd, Tampa"
  street_address: "410 Bayshore Blvd, Tampa, FL 33606"
  checkin_radius_m: 75
  idempotency_key: "loc-property-2"
  → location_id: 9102

event_create:
  location_id: 9102
  title: "Pest control - 410 Bayshore Blvd"
  start_date: "2026-09-08"
  end_date:   "2026-11-06"
  idempotency_key: "event-property-2-20260908"
  → event_id: 7102

form_assign: form_id 401, event_id 7102, idempotency_key "assign-treatment-record-7102"

sqlite_execute:
  UPDATE properties SET zensched_location_id = 9102, zensched_event_id = 7102, event_valid_until = '2026-11-06'
  WHERE property_id = 2;
```

> Added Maya Chen, 410 Bayshore Blvd. One-off termite inspect at $150, Tuesday Sep 8 at 11:00. Same Treatment Record — Luis will mark Termites. This is not a WDO graph; if the lender wants NPMA-33 / Florida WDO-10, that's still your state form. Crawl-hatch note is local only.

## "Invite Luis Ortega, luis@example.com, and make him the default."

> Inviting a technician costs $0.25. OK?

Owner: *"yes"*

```
worker_invite:
  email: "luis@example.com"
  first_name: "Luis"
  last_name: "Ortega"
  idempotency_key: "worker-luis@example.com"
  → worker_id: 601, status: "invited", invitation_sent: true
    billing: { meter: "worker_invite", units: 1, price: 0.25 }

sqlite_execute:
  INSERT INTO technicians (technician_name, email, zensched_worker_id)
  VALUES ('Luis Ortega', 'luis@example.com', 601);
  → lastInsertRowid = 1

sqlite_execute: UPDATE settings SET value = '601' WHERE key = 'default_worker_id';
```

> Invited Luis. He'll get an email with the app link and an activation code. He's the default tech now. Remember to give him Rosa's gate code and the east-hatch note at Chen's; I keep those off ZenSched on purpose.

Optional, if the owner wants a wider circle for large lots:

```
policy_get: policy_id 0
policy_update:
  policy_id: 0
  settings_json: '{"geofence_enabled": true, "require_on_site": true, "checkin_radius_m": 150}'
```

The radius is on the **policy**, not on either location.

## "Schedule this week for Luis."

```
sqlite_query: SELECT * FROM customers_due;
  → 2026-09-07 | Rosa Delgado | monthly | $89 | general_pest | 45 min
               | property 1 | evt 7101 | event_needs_roll 0 | worker 601 Luis
               | start_iso 2026-09-07T09:00:00-04:00 | end_iso 2026-09-07T09:45:00-04:00
               | shift-property-1-20260907
    2026-09-08 | Maya Chen   | on-demand | $150 | termite_inspect | 90 min
               | property 2 | evt 7102 | event_needs_roll 0 | worker 601 Luis
               | start_iso 2026-09-08T11:00:00-04:00 | end_iso 2026-09-08T12:30:00-04:00
               | shift-property-2-20260908
  (2 rows)

shift_create:
  event_id: 7101
  worker_id: 601
  start: "2026-09-07T09:00:00-04:00"
  end:   "2026-09-07T09:45:00-04:00"
  idempotency_key: "shift-property-1-20260907"
  → shift_id: 88011, status: "scheduled", forms_installed: [401]

shift_create:
  event_id: 7102
  worker_id: 601
  start: "2026-09-08T11:00:00-04:00"
  end:   "2026-09-08T12:30:00-04:00"
  idempotency_key: "shift-property-2-20260908"
  → shift_id: 88012, status: "scheduled", forms_installed: [401]
```

> Scheduled 2 stops for Luis this week:
> - Mon 9/7  9:00–9:45   Rosa Delgado, 1842 Palmetto Ct — general pest
> - Tue 9/8 11:00–12:30  Maya Chen, 410 Bayshore Blvd — termite inspect
>
> Luis has been notified in the app and both stops have the Treatment Record attached. Each stop is about $0.35 once he punches in and out and you read the photo record ($0.10 + $0.10 + $0.15). Pass him the gate code and the east-hatch note yourself.

Nothing is written to SQLite here. If the owner asks "schedule this week" again, the identical idempotency keys make ZenSched return the same two shifts instead of creating duplicates.

## Luis does the work

Monday 8:58, at 1842 Palmetto Court, in the ZenSched app: **Check in** (GPS: 18 m from the pin, verified, $0.10). Treats interior kitchen baseboards and exterior patio. Opens **Treatment Record**: Pests seen = Ants; Products used = Alpine WSG, Talstar; Amount = "0.5 gal mixed / 0.3 oz Alpine"; Areas = Both; two photos; Conditions = Active; Follow-up = None; Notes = "Heavy trail at patio slider, treated and wiped." Submits (no signature). **Check out** 9:41 ($0.10).

Tuesday 10:55, 410 Bayshore Blvd: check in at the east hatch. Inspects the crawl. Treatment Record: Pests seen = Termites; Products used = None; Amount blank; Areas = Crawl; three photos of the sill; Conditions = Active; Follow-up = Quote treatment; Notes = "Live subterranean activity at sill plate, east crawl. Recommend liquid treat." Check out 12:22.

## "Record this week's jobs."

```
shift_list:
  date_from: "2026-09-07"
  date_to:   "2026-09-13"
  status:    "checked_out"
  → shifts: [
      { shift_id: 88011, event_id: 7101, worker_id: 601, date: "2026-09-07", start: "09:00", end: "09:45", status: "checked_out" },
      { shift_id: 88012, event_id: 7102, worker_id: 601, date: "2026-09-08", start: "11:00", end: "12:30", status: "checked_out" }
    ], count: 2

sqlite_query: SELECT zensched_shift_id FROM jobs WHERE zensched_shift_id IN (88011, 88012);
  → (none; not yet recorded)

sqlite_query: SELECT p.property_id, p.customer_id, c.service_id, c.service_rate
              FROM properties p JOIN customers c USING (customer_id)
              WHERE p.zensched_event_id = 7101;
  → 1, 1, 1, 89.00
```

Agent (metered read, rule 11):

> There are 2 treatment records to pull, both with photos, so about $0.30 ($0.15 each). I'll read them once and keep the summaries locally. That's the last piece of the ~$0.35/stop meter (punches already billed when Luis checked in and out). OK?

Owner: *"go"*

```
form_export:
  form_id: 401
  since: "2026-09-07"
  until: "2026-09-13"
  format: "json"
  → row_count: 2, rows: [
      { submission_id: 4401, worker_id: 601, event_id: 7101, submitted_at: "2026-09-07 09:38:12",
        data: { pests_seen: ["ants"], products_used: ["alpine_wsg", "talstar"],
                amount_applied: "0.5 gal mixed / 0.3 oz Alpine", areas: "both",
                conditions: "active", notes: "Heavy trail at patio slider, treated and wiped.",
                follow_up: "none" },
        media: [ { cdn_url: "https://cdn.zensched.com/m/.../d1.jpg" },
                 { cdn_url: "https://cdn.zensched.com/m/.../d2.jpg" } ] },
      { submission_id: 4402, worker_id: 601, event_id: 7102, submitted_at: "2026-09-08 12:18:40",
        data: { pests_seen: ["termites"], products_used: ["none"],
                amount_applied: "", areas: "crawl",
                conditions: "active",
                notes: "Live subterranean activity at sill plate, east crawl. Recommend liquid treat.",
                follow_up: "quote_treatment" },
        media: [ { cdn_url: "https://cdn.zensched.com/m/.../c1.jpg" },
                 { cdn_url: "https://cdn.zensched.com/m/.../c2.jpg" },
                 { cdn_url: "https://cdn.zensched.com/m/.../c3.jpg" } ] }
    ]
    billing: { meter: "form_media", units: 2, price: 0.30 }

shift_status: shift_id 88011
  → status "checked_out", actual_in "2026-09-07T08:58:10-04:00", actual_out "2026-09-07T09:41:05-04:00",
    punches: [ { type: "in", gps_verified: true, distance_from_site_m: 18 },
               { type: "out", gps_verified: true, distance_from_site_m: 12 } ]
(free)

sqlite_execute:
  INSERT INTO jobs (customer_id, property_id, service_id, completed_date, amount,
                    zensched_shift_id, zensched_event_id, zensched_worker_id,
                    actual_in, actual_out, duration_minutes, gps_verified, report_dc_id,
                    pests_seen, products_used, product_summary, amount_applied,
                    areas, conditions, follow_up, notes, photo_urls)
  VALUES (1, 1, 1, '2026-09-07', 89.00,
          88011, 7101, 601,
          '2026-09-07T08:58:10-04:00', '2026-09-07T09:41:05-04:00', 43, 1, 4401,
          '["ants"]', '["alpine_wsg","talstar"]',
          'Alpine WSG, Talstar — 0.5 gal mixed / 0.3 oz Alpine',
          '0.5 gal mixed / 0.3 oz Alpine',
          'Both', 'Active', 'None',
          'Heavy trail at patio slider, treated and wiped.',
          '["https://cdn.zensched.com/m/.../d1.jpg","https://cdn.zensched.com/m/.../d2.jpg"]');
  -- trigger: customers.next_service_date for Rosa → 2026-10-07 (+1 month)
  -- trigger: jobs.technician_id → 1 (Luis)

sqlite_execute:
  INSERT INTO jobs (customer_id, property_id, service_id, completed_date, amount,
                    zensched_shift_id, zensched_event_id, zensched_worker_id,
                    actual_in, actual_out, duration_minutes, gps_verified, report_dc_id,
                    pests_seen, products_used, product_summary, amount_applied,
                    areas, conditions, follow_up, notes, photo_urls)
  VALUES (2, 2, 3, '2026-09-08', 150.00,
          88012, 7102, 601,
          '2026-09-08T10:55:02-04:00', '2026-09-08T12:22:18-04:00', 87, 1, 4402,
          '["termites"]', '["none"]', NULL, '',
          'Crawl', 'Active', 'Quote treatment',
          'Live subterranean activity at sill plate, east crawl. Recommend liquid treat.',
          '["https://cdn.zensched.com/m/.../c1.jpg","https://cdn.zensched.com/m/.../c2.jpg","https://cdn.zensched.com/m/.../c3.jpg"]');
  -- trigger: customers.next_service_date for Maya → NULL (on-demand)
```

> Recorded 2 jobs, both GPS-verified.
>
> **Follow-up — Tuesday, Chen termite inspect:** Luis marked Termites / Active / **Quote treatment**: *"Live subterranean activity at sill plate, east crawl. Recommend liquid treat."* 3 photos. This is not a WDO graph; if the buyer needs the state form, that's still on you.
>
> Delgado monthly: Alpine WSG + Talstar, both sides, 0.5 gal mixed / 0.3 oz Alpine. Next due Oct 7. Chen's on-demand date is cleared.

## "Chemical log for last week."

```
sqlite_query: SELECT * FROM chemical_log WHERE treatment_date BETWEEN '2026-09-07' AND '2026-09-13';
  → 2026-09-07 | 1842 Palmetto Court | Rosa Delgado
    | Alpine WSG, Talstar — 0.5 gal mixed / 0.3 oz Alpine
    | ["alpine_wsg","talstar"] | 0.5 gal mixed / 0.3 oz Alpine | Luis Ortega
    | ["ants"] | Both | Active
```

Chen's inspect (products = None) is omitted on purpose.

> Your copy from the Treatment Record — not a state form:
>
> | Date | Property | Product | Amount | Tech |
> |---|---|---|---|---|
> | Sep 7 | 1842 Palmetto Court | Alpine WSG, Talstar | 0.5 gal mixed / 0.3 oz Alpine | Luis Ortega |
>
> Chen's Tuesday inspect is not on this list (no product applied). Keep whatever FDACS wants on their form separately.

## "Draft invoices for everyone with uninvoiced work."

```
sqlite_query: SELECT * FROM jobs_to_invoice;
  → Maya Chen    | 1 job | 150.00 | 2026-09-08 .. 2026-09-08
    Rosa Delgado | 1 job |  89.00 | 2026-09-07 .. 2026-09-07

sqlite_execute:
  INSERT INTO invoices (customer_id, invoice_date, due_date, total_amount, line_items)
  SELECT j.customer_id, date('now'),
         date('now', '+' || (SELECT value FROM settings WHERE key = 'invoice_due_days') || ' days'),
         SUM(j.amount),
         json_group_array(json_object('job_id', j.job_id, 'date', j.completed_date, 'service', s.service_name,
                                      'amount', j.amount, 'shift_id', j.zensched_shift_id, 'products', j.product_summary))
  FROM jobs j JOIN services s ON s.service_id = j.service_id
  WHERE j.invoiced = 0 AND j.customer_id = 1 GROUP BY j.customer_id;

sqlite_execute: UPDATE jobs SET invoiced = 1 WHERE invoiced = 0 AND customer_id = 1;
sqlite_query:   SELECT invoice_number, invoice_date, due_date, total_amount FROM invoices WHERE invoice_id = last_insert_rowid();
  → INV-2026-0001 | 2026-09-08 | 2026-09-22 | 89.00     (number assigned by trigger)

(same three statements for customer 2 → INV-2026-0002, 150.00)
```

The agent then writes the invoices out as text:

> Here are your two invoices. Paste each into an email or text.
>
> ```
> GreenShield Pest
> Invoice INV-2026-0001
> To: Rosa Delgado (rosa@example.com)
> Date: Sep 8, 2026    Due: Sep 22, 2026
>
>   Sep 7    General pest service, 1842 Palmetto Court    $89.00
>
> Total due: $89.00
> Visit GPS-verified at the property.
> ```
>
> ```
> GreenShield Pest
> Invoice INV-2026-0002
> To: Maya Chen (maya.chen@example.com)
> Date: Sep 8, 2026    Due: Sep 22, 2026
>
>   Sep 8    Termite inspection, 410 Bayshore Blvd        $150.00
>
> Total due: $150.00
> Visit GPS-verified at the property.
> ```
>
> Say "sent" when they're out and I'll note the date. Want me to draft the termite-treat quote for Chen as a separate on-demand job?

## "Sent. Rosa already paid."

```
sqlite_execute: UPDATE invoices SET sent_date = date('now') WHERE invoice_number IN ('INV-2026-0001', 'INV-2026-0002');
sqlite_execute: UPDATE invoices SET paid = 1, paid_date = date('now') WHERE invoice_number = 'INV-2026-0001';
```

> Marked both sent and INV-2026-0001 paid. Chen's $150 (INV-2026-0002) is outstanding, due Sep 22.

## Eight weeks later: "Schedule this week."

Daylight-saving time ended on Nov 1, so the owner (or the agent, noticing the date) first updates the fixed offset (SKILL.md rule 8):

```
sqlite_execute: UPDATE settings SET value = '-05:00' WHERE key = 'timezone_offset';

sqlite_query: SELECT * FROM customers_due;
  → 2026-11-07 | Rosa Delgado | monthly | evt 7101 | event_valid_until 2026-11-05 | event_needs_roll 1
               | start_iso 2026-11-07T09:00:00-05:00 | ...

event_create:
  location_id: 9101
  title: "Pest control - 1842 Palmetto Court"
  start_date: "2026-11-07"
  end_date:   "2027-01-05"
  idempotency_key: "event-property-1-20261107"
  → event_id: 7201

form_assign: form_id 401, event_id 7201, idempotency_key "assign-treatment-record-7201"
sqlite_execute: UPDATE properties SET zensched_event_id = 7201, event_valid_until = '2027-01-05' WHERE property_id = 1;

shift_create: event_id 7201, worker_id 601, start "2026-11-07T09:00:00-05:00",
              end "2026-11-07T09:45:00-05:00", idempotency_key "shift-property-1-20261107"
```

> Scheduled Delgado for Sat Nov 7 9:00. Her ZenSched event was about to expire (Nov 5), so I renewed it through Jan 5. Chen has no next date (on-demand) so she is not on this week's list.

## Summary of who stored what

| Thing | Where | Why |
|---|---|---|
| Rosa's contact, $89 monthly cadence, Chen's one-off $150, prices | SQLite | CRM; ZenSched does not model rates or recurrence |
| Gate code, crawl hatch, applicator license, customer names / phones / emails | SQLite **only** | Privacy; never sent to ZenSched (locations and events are named by street address) |
| Each property's GPS location | ZenSched (integer ID in `properties`) | Needed for geofenced check-in |
| Each property's current ≤60-day event and its end date | ZenSched (integer ID + `event_valid_until` in `properties`) | Shifts hang off events; renewed by the agent |
| The Treatment Record form | ZenSched (ID in `settings`) | Installed on the tech's phone per shift |
| Luis, his invite, his app | ZenSched (integer ID in `technicians`) | Workforce and notifications |
| The week's two shifts | ZenSched only | Live schedule; never copied |
| GPS punches, actual times | ZenSched only | Verified record; queried via `shift_status` / `timesheet_export` |
| Two treatment records with photos | ZenSched (originals); summary + photo URLs in `jobs` | Read once (metered), then chemical log / invoices from SQLite |
| Two `jobs` rows referencing shift and submission IDs | SQLite | Billing + chemical-log extract |
| Two invoices, one paid | SQLite | Billing |
