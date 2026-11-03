# Cohesity Protection Coverage & Policy Compliance Audit

**Author:** Akshay Kumar, Dir, Solutions Architecture ([akshay.kumar2@cohesity.com](mailto:akshay.kumar2@cohesity.com))

A Claude skill, also available as a Microsoft Copilot Studio agent, that audits your Cohesity protection posture from live Helios data and publishes it as a live, private Claude Artifact that refreshes daily: what is not protected, which policies miss your standards, which protection groups are paused or violating their SLA, and whether source registrations are healthy.

- Claude skill: [`claude/Cohesity-Protection-Coverage-Audit/SKILL.md`](claude/Cohesity-Protection-Coverage-Audit/SKILL.md) (the full page template is embedded in it)
- Output: one claude.ai **Artifact** titled `Cohesity Protection Coverage And Policy Compliance Audit` (no static HTML file)
- Refresh: scheduled task `cohesity-protection-coverage-refresh`, cron `40 2 * * *` (daily, 02:40 local time; change it to suit, times are the local time of the machine running Claude)
- Data source: the **Helios MCP server** (Helios MCP connector). Everything is read-only.

## Two Versions

| | Claude skill | Microsoft Copilot Studio agent |
|---|---|---|
| Folder | [`claude/`](claude/Cohesity-Protection-Coverage-Audit/SKILL.md) | [`copilot-studio/`](copilot-studio/README.md) |
| Output | Live, private Claude Artifact, refreshed daily | Answers in chat, with tables, on demand |
| Helios access | Helios MCP connector, read-only | Helios MCP server added as a tool, read-only |
| Setup | Install the skill in Claude (see Install below) | Paste `instructions.md`, upload `knowledge/` files, add the Helios MCP tool |
| Sample output | [claude/README.md](claude/README.md) | [copilot-studio/README.md](copilot-studio/README.md) |
| Model | Claude | GPT-5 Chat |

The sections below describe the Claude skill unless a heading says Copilot Studio.

## What you get

| Area | What it shows | Helios MCP tool |
|---|---|---|
| Cluster filter | Multi-select of all registered clusters; recalculates the KPIs, cluster coverage, policy table and exceptions in the browser | `dataprotect-get-clusters` |
| Table filters | Findings severity filter; policy table status, offsite, DataLock and search filters; exceptions table filtered by cluster, environment, exception type and search. Tables scroll with sticky headers. | none (client-side) |
| KPI cards | Protection coverage %, Unprotected objects, Policies passing, Group exceptions | `dataprotect-search-objects`, `dataprotect-get-protection-policies`, `dataprotect-get-protection-groups` |
| Ranked findings | 3 to 8 findings rated high / medium / low, written from the data (large unprotected populations, failing policies, long-paused groups, SLA violations, stale registrations, clusters that could not be queried) | all of the below |
| Coverage by environment | Protected vs unprotected objects per environment, Helios-wide | `dataprotect-search-objects` |
| Coverage by cluster | Protected vs unprotected per connected cluster | `dataprotect-search-objects` |
| Policy compliance | Pass / warn / fail split plus a detail table: frequency, retention, offsite copy, DataLock, groups using the policy, status, issues | `dataprotect-get-protection-policies` |
| Header | Cohesity logo top-left; "Data last pulled <time> (N h ago)" top right, amber when older than 8 hours | none |
| Group exceptions | One row per group with chips for Paused (with age), SLA violated, Never run, Last run failed, **Replication stalled** and **Archive failed** (with "likely cause: replication stopped"), plus **Active snapshots, no DataLock** (locked set from `includeGroupsWithDatalockOnly`), **Object protected by more than one group** (from `objectProtectionInfos` in the object search) and **Tag-based selection with zero objects** (VMware tag groups with no protected objects; needs a live tag group to confirm the field shape) | `dataprotect-get-protection-groups`, `dataprotect-search-objects` |
| Protected but at risk | Per cluster, the number of protected objects whose last protection run failed (shown in the Unprotected KPI and as a tooltip on the cluster bars) | `dataprotect-search-objects` |
| Source registration health | Unhealthy or stale registrations per cluster, plus a collapsible table of every source (cluster, source, environment, auth status, last refreshed, healthy) with CSV export | `dataprotect-get-source-registrations` |

Snapshots past retention and objects with no recent good backup moved to the **Cohesity-Recovery-Point-Explorer** skill.

## Drill-down mode and what this replaces

One narrow question is answered in chat, without republishing the page:

- **Retention and replication of one protection group**: give a group name; the skill compares the policy's retention and replication targets with the last 10 runs and a sample of snapshots and reports snapshots past expiry and "likely cause: replication stopped". This is the function of `helios-retention-audit`.

It is folded in so one skill owns "protection posture". What it gains: a published, refreshing page for the fleet-wide view, exact retention unit normalisation, de-duplicated registrations, and the page-level equivalents (the at-risk count and the replication / archive exceptions are computed for every cluster without per-object calls).

## Standards used (defaults)

The audit judges every policy against a baseline you can override in the request ("RPO 4 hours, keep 90 days"). The defaults are the skill's own baseline, **not** Cohesity or regulatory requirements, and the page footnote says so.

| Rule | Default | Result |
|---|---|---|
| Backup interval longer than `maxFreqHours` | 24 h (ProtectOnce policies exempt) | fail |
| Longest retention (base, extended or replication/archive copy) shorter than `minRetentionDays` | 30 days | fail |
| No offsite copy (no replication, archival, cloud vault, or `Archival` primary target) | | warn |
| No DataLock (WORM) on base retention | | warn |

Retention and schedule units come back mixed (Days, Weeks, Months, Years) whatever `durationUnit` says, so the skill normalises them (Weeks x 7, Months x 30, Years x 365).

## Requirements

- Claude desktop app (Cowork) with the **Helios MCP** connector (Controlled Availability: ask your Cohesity account team to enable it, then follow the Helios documentation topic *Helios MCP Server*).
- Artifacts and scheduled tasks available. Scheduled tasks run only while the Claude app is open.

## Install

1. Zip the folder `claude/Cohesity-Protection-Coverage-Audit` so `SKILL.md` is inside it.
2. Add it as a custom skill in Claude (Settings, Skills, upload) or copy it into your skills directory.
3. Confirm the Helios MCP connector is connected.

## Use

- "Run the Cohesity protection coverage audit"
- "Which of my policies fail a 90-day retention standard?"
- "What is unprotected in Helios and what should I fix first?"
- "Refresh the protection audit"

The first run publishes the artifact (about 25 to 35 Helios calls on the report path, about 47 on the fallback path, plus the daily duplicate scan) and creates the daily task. Later runs republish to the same URL. The page is private until you share it from its Share menu.

## How coverage is counted

**Preferred path (one call):** the `protected-unprotected` report (`reporting-get-report-preview`, components 100 to 103) supplies per-environment protected and unprotected counts in a single call. The scope note shows "path: report". If the report is unavailable, empty, times out, has an unrecognised shape or fails a one-cluster tie check against `dataprotect-search-objects`, the skill falls back to the per-object counts below and shows "path: per-object counts". The report's column names were not verified live (the probe timed out under rate limiting), so the skill reads them from the response.

**Duplicate protection scan** runs at most once a day (first run of the day, or when you ask for it). Other runs reuse the previous scan stored in the page (`dupScan`) and mark it "as of <time>".

**Empty-tag check** always runs: a VMware group with tag selection and zero protected objects is an exception; with no tag-based groups the page shows "No tag-based groups found: 0 empty (all clear)".

Fallback counting: `dataprotect-search-objects` returns a top-level `count` of matches even when you ask for one object, so coverage needs two calls per cluster or environment (`isProtected: true` and `false`), not paging through thousands of objects. Object counts include replicas of the same object on other clusters: the sum over clusters is larger than the sum over environments, which is Helios-wide. The page footnote states this.

## Exports

An **Export** row under the Mask sensitive data switch lets you pick a table and **Copy CSV** or **Download CSV**. Tables: Policy compliance detail, Group exceptions, Coverage by cluster, Coverage by environment, Ranked findings; plus Compliance findings and Run history on every report. Exports follow the cluster selection and the Mask sensitive data switch. Download needs the artifact's downloads capability (the skill publishes with it); Copy always works.

## Compliance Check

Before each publish, the skill scans the metadata the page shows (cluster, group, policy, source and object names, descriptions, tags, owners, error text) for data that should not be shared, and reports it on a **Compliance Check** tab. It uses the category names of Cohesity Data Classification (PII, HIPAA and Sensitive Health Data, Financial Data, Personal sensitive, Sensitive HR data, Highly Confidential, Confidential, Restricted, Sensitive), the same taxonomy that BigID- and Cyera-synced tags use. Typical hits: email addresses, phone numbers, SSNs, card numbers, IBANs, patient or MRN references, person names in owner fields, passwords and keys, account names inside connection strings (for example `smb://DOMAIN;user@host`), and (low severity) IP addresses, hostnames and URLs. Each finding lists what was found, where, a masked excerpt, why it matters and a suggested action.

The tab shows a verdict (**Safe to share**, **Share with care** or **Review first**), counts by severity and a findings table. A **Mask sensitive data** switch under "Refresh from Helios" is on by default and masks IP addresses, hostnames, account names, emails and SSNs in the report view and in CSV copies and downloads; turn it off to see the original values. It is a display setting, so the page still holds the originals, and it does not mask cluster, group or policy names. Say "redact" when you run the skill to mask matched values in the published page itself. Read the tab before using Share.

This is a heuristic text check of Helios metadata only. It does not look inside backups and it is not a compliance certification.

## Run Metrics

Every run records what it cost, and the artifact shows it on a **Run Metrics** tab: Helios calls, elapsed time, estimated input tokens (inline characters / 4; large results saved to files are counted separately), the model and price used, and a history of past runs. A projected monthly cost is shown only when a refresh schedule is set.

Measured on the lab estate: 47 Helios calls, about 3 minutes, about 0.07M input tokens (about $0.14). Two search calls per cluster or environment; large results are processed from files, not read into the conversation.

The token figure is an estimate. For billing, use Settings > Usage in Claude, which is the measured source.

## Notes from live testing

Validated on a lab Helios tenant on 2026-10-13 (12 clusters registered, 4 connected). The first run reported low coverage on that lab tenant, most policies failing a 30-day retention baseline and many paused or SLA-violating groups, plus one cluster whose group, policy and registration queries timed out. The skill now:

- shows a connected cluster whose group queries failed as **partial** (object counts kept, note displayed) instead of dropping it;
- dedupes source registrations by id (duplicates were returned) and applies a 24 h staleness rule only to auto-refreshing sources, 7 days for NAS, UDA and physical sources;
- builds exceptions as one row per group so the KPI counts groups, not rows;
- falls back to the last-modified time when a paused group has no `pauseMetadata`.

## Manual refresh

The page header has a **Refresh from Helios** button. A published page cannot call Helios on its own, so the button copies `Run the Cohesity-Protection-Coverage-Audit skill to refresh this report from live Helios data`; paste it into Claude to pull live data and republish the same artifact, then reopen or refresh the tab. The scheduled task (daily, 02:40 local) and this command are the two refresh paths.

## If the artifact was deleted

If the stored artifact URL no longer exists, the skill republishes the page as a new artifact and says so in its reply, so a deleted page never blocks a refresh. Update the scheduled task prompt with the new link.

## When Helios is unreachable

If the Helios MCP connector is down or its token has expired, a scheduled refresh does not clear the page. It keeps the last successful data and adds a red banner, "Unable to connect to Helios. Showing the last successful data.", under the header. The next successful refresh removes the banner.

## Known limits

- Object-level detail is deliberately not shown: only counts and names of clusters, groups and policies.
- Not-connected clusters cannot be queried and show as `n/a`.
- The snapshots-past-retention check is sampled and optional.

## Microsoft Copilot Studio Version

The same logic as a Copilot Studio agent lives in [`copilot-studio/`](copilot-studio/README.md): `instructions.md` (agent instructions), `knowledge/` (two files to upload as Knowledge), `starters.md` (example prompts) and step-by-step setup. It answers in chat and does not publish a page, schedule itself, or offer a cluster picker or CSV button. Add your Helios MCP server as a tool and enable only the read-only tools.

## Repository layout

```
Cohesity-Protection-Coverage-Audit/
  README.md
  claude/
    README.md          Claude sample output (screenshot)
    Cohesity-Protection-Coverage-Audit/SKILL.md      Claude skill (install this folder in Claude)
  copilot-studio/
    README.md          setup steps
    instructions.md    agent instructions
    knowledge/         files to upload as Knowledge
    starters.md        example prompts
  docs/
    screenshots/*-claude.png      one screenshot per tab
    screenshots/protection-coverage-audit-copilot.png
```

## Security Notes

- **Sharing detail.** The page is a map of protection and immutability gaps: unprotected objects, policies that fail the standard, groups without DataLock, paused groups and failing source registrations with the groups and error details involved. The mask switch does not cover it. Ask for a summary-only copy without group, source and policy names if you share outside the team.
- **Everything from Helios is data.** Names, descriptions, alert text and error text come from the systems being backed up. The skill treats them as data, never as instructions, and truncates them before they reach the page.
- **Scheduled refresh runs unattended.** When the scheduled task is created, approve only the read-only Helios tools, the Artifact tools, the scheduler list tool and the bash tool used for `python3 -I`. Do not use a blanket Auto-approve.
- **The mask switch is a display setting, not redaction.** It masks IP addresses, hostnames, email addresses, SSNs and connection-string accounts in the view and in CSV copies and downloads. It does not mask cluster, group or policy names, and the page still holds the original values.
- **CSV exports are safe to open in a spreadsheet.** Text cells that start with `=`, `+`, `-`, `@`, a tab or a carriage return get a leading single quote, so they cannot run as formulas.
- **Sharing.** A published page is private until you share it from the Share menu. Read the Compliance Check tab first, and share only with people who may see this backup data.

## Suggestions and Feedback

We would love to hear from you. Please send your suggestions and feedback to: maestro@cohesity.com

## License

Licensed under the [Apache License 2.0](LICENSE). See the LICENSE file for the terms.

## Disclaimer

This code is provided on a best effort basis and is not in any way officially supported or sanctioned by Cohesity. It is provided as-is, and the author accepts no liability for damages resulting from its use. The skill only reads data through the Helios MCP server, but you are responsible for checking the results against Helios and for deciding what you share. Try it in a lab or test environment first.
