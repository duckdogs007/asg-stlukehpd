# asg-stlukehpd

Power Automate flows and supporting artifacts for Advanced Security Group's
St. Luke / Henrico PD monitoring work.

## Overview

This repo's first flow, **HPD SRA 312 Real-Time Alerts**, watches the Henrico
Police Calls-for-Service RSS feed filtered to Security Reporting Area 312
(St. Luke neighborhood) and emails an alert within minutes of a new call
being published.

Upstream feed: `https://ppd.henrico.gov/rss/cad.aspx?sra=312`

Three paths to stand this up are documented below. **Option 1** is the
default — build it in the Power Automate UI. Options 2 and 3 are for
promotion / reference.

---

## Option 1 — UI build (recommended for v1)

**Steps:**

1. https://make.powerautomate.com → **Create** → **Scheduled cloud flow**
2. Name: `HPD SRA 312 Real-Time Alerts`
3. Starting: tomorrow at any time, repeat every **10 minutes**
4. **Create**, then delete the default Recurrence trigger — we'll replace it with the RSS trigger
5. Add trigger: search `RSS` → select **When a feed item is published**
   - Feed URL: `https://ppd.henrico.gov/rss/cad.aspx?sra=312`
   - Chosen property will be used for deduplication: `PublishDate`
   - Since: leave default (Now)
6. Add action: search `Outlook` → select **Send an email (V2)**
   - To: *your email*
   - Subject:
     ```
     [HPD SRA 312] @{triggerOutputs()?['body/title']} — @{formatDateTime(triggerOutputs()?['body/publishDate'], 'MM/dd/yyyy h:mm tt')}
     ```
   - Body (toggle to HTML mode via the `</>` button) — see [`flows/email-template.html`](flows/email-template.html)
   - Importance: `Normal`
7. **Save**

Test immediately — see [Testing](#testing) below.

---

## Option 2 — Solution import (for promotion to production later)

Once v1 is stable, package the flow into a Dataverse solution (`.zip`), check
it into source control, and import into other environments.

**Steps (after flow is built and tested):**

1. https://make.powerautomate.com → **Solutions** → **New solution**
2. Name: `ASG Security Automation`, publisher: whatever's standard at ASG
3. Into the solution: **Add existing → Cloud flow → Select `HPD SRA 312 Real-Time Alerts`**
4. **Export** → Managed or Unmanaged (Unmanaged if you'll edit in the target environment)
5. The exported `.zip` is your deployable artifact — drop it into [`solution/`](solution/)

To import elsewhere: **Solutions → Import solution → upload the .zip**.
Connections will need to be re-mapped per environment.

---

## Option 3 — ARM/Logic Apps-style import (not supported for Power Automate)

Power Automate cloud flows are not importable from raw workflow JSON the way
Azure Logic Apps are. The [`flows/definition.json`](flows/definition.json) in
this repo uses the same underlying schema, but Power Automate's UI doesn't
accept it as a direct upload. Use Option 1 or 2.

---

## The definition, for reference

### Trigger (RSS)

```
Connector: RSS (shared_rss)
Operation: When a feed item is published
Parameters:
  feedUrl: https://ppd.henrico.gov/rss/cad.aspx?sra=312
  feedProperty: PublishDate
```

### Action (Send email)

```
Connector: Office 365 Outlook (shared_office365)
Operation: Send an email (V2)
Parameters:
  To: <your email>
  Subject: [HPD SRA 312] @{triggerOutputs()?['body/title']} — @{formatDateTime(triggerOutputs()?['body/publishDate'], 'MM/dd/yyyy h:mm tt')}
  Body: (HTML — see flows/email-template.html)
  Importance: Normal
```

---

## Trigger output fields

| Field | Example |
|---|---|
| `id` | `https://henrico.gov/police` |
| `title` | `Firearm Violation` |
| `primaryLink` | `https://henrico.gov/police` |
| `links` | array of any additional links |
| `updatedOn` | `2026-04-19T01:40:10Z` |
| `publishDate` | `2026-04-19T01:40:10Z` |
| `summary` | `Firearm Violation<br /><strong>100 Block    Engleside Ct  </strong><br />4/19/2026 1:40:10 AM` |
| `copyright` | `Copyright Henrico County, Virginia` |
| `categories` | array (empty for this feed) |

**Note on `summary`:** the Henrico feed packs the address into HTML inside
`<strong>` tags. Rendered in an email body, this displays correctly. If you
later need to extract the address programmatically, strip HTML and parse in
Flow B — not here.

---

## Testing

1. **Flow will not fire immediately.** The RSS trigger runs on Microsoft's
   polling schedule (every few minutes for paid plans). First alert may take
   5–15 minutes.
2. **Manual test is limited.** `Test → Manually → Run flow` will check the
   feed once, but only fires if there's an item newer than the last-seen
   checkpoint.
3. **Force a test alert:** change the Feed URL temporarily to a high-volume
   feed (e.g., a BBC News RSS), confirm the email format, then revert.

### Success criteria for the first 24 hours

- [ ] At least one alert email arrives
- [ ] Subject line renders with call type and timestamp (no raw `@{...}` visible)
- [ ] Body renders HTML correctly in Outlook
- [ ] No duplicate alerts for the same call

### If you get zero alerts in 24 hours

- Confirm the feed has new items at `https://ppd.henrico.gov/rss/cad.aspx?sra=312`
- Check **My flows → HPD SRA 312 Real-Time Alerts → Run history**
- Open run details — compare `publishDate` against what's in the feed

---

## What you're NOT doing in v1

- ✘ No Teams post (v2)
- ✘ No severity filtering (all calls trigger)
- ✘ No address filtering (SRA 312 ≈ St. Luke per sample data)
- ✘ No cross-reference with Reliant / ASG reports (Flow B)
- ✘ No custom dedup (using Microsoft's built-in RSS dedup)

---

## Known limitations

1. **RSS dedup uses `PublishDate` hash.** Same timestamp = treated as same item.
2. **Polling cadence is not user-configurable** on Standard RSS connector (~3–5 min).
3. **No error alerting.** Add a monitoring flow in v3.
4. **Flow runs count against quota.** ~8,640 trigger-checks/month at 5-min polling.

---

## Repo layout

```
.
├── README.md
├── flows/
│   ├── definition.json
│   ├── email-template.html
│   └── HPD_SRA_312_Real_Time_Alerts/
└── solution/
```
