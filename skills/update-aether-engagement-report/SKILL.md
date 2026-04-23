---
name: update-engagement-report
description: >
  Update the Aether global engagement HTML report with latest Kusto data from all 10 production clusters.
  Updates ALL sections: KPIs, hourly chart, DAU bars, table, bottom charts, and timestamp.
  Use when: "update engagement report", "update the report", "refresh engagement numbers",
  "update with latest data", "update numbers again".
---

# Update Engagement Report

Systematically update `aether-engagement-report.html` with fresh data from all 10 Kusto clusters.

## Step 1: Query all clusters

Run these queries. Use `gRPC stream ended cleanly` signal. Use `format_datetime(TIMESTAMP, "yyyy-MM-dd")` for UTC day alignment.

### 1a. US daily + hourly (fdislandscscprduswus)

```kql
// US DAU — last 3 days
KubernetesContainers | where TIMESTAMP > ago(3d)
| where container_name == "aetherruntime"
| where message has "gRPC stream ended cleanly"
| extend conv_id = extract(@"conversation_id=([a-f0-9\-:]+)", 1, message)
| where isnotempty(conv_id)
| extend user_id = tostring(split(conv_id, ":")[1])
| where user_id matches regex @"^[a-f0-9]{8}-"
| extend utc_date = format_datetime(TIMESTAMP, "yyyy-MM-dd")
| summarize dau = dcount(user_id) by utc_date
| order by utc_date asc
```

```kql
// US 2h hourly — last 2 days (for hourly chart extension)
KubernetesContainers | where TIMESTAMP > ago(2d)
| where container_name == "aetherruntime"
| where message has "gRPC stream ended cleanly"
| extend conv_id = extract(@"conversation_id=([a-f0-9\-:]+)", 1, message)
| where isnotempty(conv_id)
| extend user_id = tostring(split(conv_id, ":")[1])
| where user_id matches regex @"^[a-f0-9]{8}-"
| summarize users = dcount(user_id) by bin(TIMESTAMP, 2h)
| order by TIMESTAMP asc
```

### 1b. EU hourly (fdislandscscprdeuneu) — same 2h query

### 1c. JP hourly (fdislandscscprdjpejp) — same 2h query

### 1d. All other clusters — DAU only (today's UTC date)

Query each with: `where TIMESTAMP > datetime(YYYY-MM-DD)` using today's UTC date.

Clusters: UK (prduksuk), FR (prdfrcfr), DE (prddewcde), CH (prdchnch), AU (prdaueau), CA (prdcacca), IN (prdincin)

## Step 2: Compute totals

- **Europe** = EU + UK + FR + DE + CH
- **APAC+CA** = AU + CA + IN + JP
- **Global** = US + Europe + APAC+CA

## Step 3: Update ALL sections of the HTML

### 3a. Date labels — critical convention

All table rows, DAU bar labels (`Mon<br>Apr 20` etc.) and the geo-share heading use the **UTC date** from the Kusto query and the **day-of-week that matches that UTC date**. Never shift the date by ±1 — a row for `utc_date = 2026-04-20` is labeled `Mon Apr 20` because Apr 20 is a Monday. Double-check with `date -d "2026-04-20" +%A` (or equivalent) before writing a new label. If day-of-week and date disagree, the label is wrong.

### 3b. Timestamp
Get local time by running `date` in bash, then update the `#lastUpdated` span text.
Use format: `Updated: Mon Apr 18, 2026 5:07 PM PDT` (user's local timezone, not UTC).

### 3c. KPI cards (4 cards)
- Global peak or today's number
- Today's partial number
- Europe number
- Notable regional number (JP or other record)

### 3d. Hourly chart arrays
**This is the most commonly missed step.** Extend `usHourly`, `euHourly`, and `jpHourly` arrays with new 2h bins since the last entry. Compare the last timestamp label in the chart with the latest data — add only NEW bins. Pad EU/JP arrays to match US length.

Also extend the `days` label array if a new day name is needed.

**Hover tooltip**: The chart's tooltip `title` callback should show `"<day> <HH>:00 PDT (2h bin)"` so the hour is unambiguous (x-axis labels alone read like dates). Keep this callback when editing chart options.

### 3e. DAU bars
Update the last bar's data (today's partial) or add a new bar if a new day started. Update `maxVal` if global exceeds it.

### 3f. Table
Update the last row (today's partial) or add new finalized + partial rows.

### 3g. Bottom 3 charts (Sessions/User, Peak 2h, Tenants)
Add new day entries if needed. Update today's partial values.

### 3h. Geo distribution bars
Update if peak day changed.

## Step 4: Do NOT open the report

The HTML has `<meta http-equiv="refresh" content="300">` — any open browser tab picks up new data automatically every 5 min. Opening a new tab each run leaves the user with a dozen stale tabs by morning.

Only open the report on the first run of a fresh session, or when the user explicitly asks. Otherwise skip this step — the file is written, the tab will refresh itself.

## Checklist

Before reporting done, verify:
- [ ] Timestamp updated
- [ ] KPI cards reflect latest numbers
- [ ] Hourly chart arrays extended with new bins
- [ ] DAU bars include today
- [ ] Table row for today updated
- [ ] Report opened in browser
