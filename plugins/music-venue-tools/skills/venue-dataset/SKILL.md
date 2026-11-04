---
name: "venue-dataset"
description: "Look up, add and update venues in the shared venue master (the TX venue dataset), the live list of verified live music venue leads, through the Venue Master connector. Used by live-music-check, venue-research and band-venue-fit."
---

# Venue dataset

The one way to read and write the shared **venue master** (also called the TX venue dataset): a list of **verified live music venue leads** shared by every musician who uses these skills. It's live: an update one user makes reaches everyone straight away. Other skills call this skill instead of handling the data themselves:

- `live-music-check`: looks venues up first; adds venues it confirms have live music, then has `venue-research` fill them in.
- `venue-research`: reads venue facts and submits what it learns.
- `band-venue-fit`: reads venue facts and joins them, by Venue ID, to the user's own act analysis file; never writes to the master.

A user can also ask directly ("Is Gruene Hall in the venue dataset?", "What do we know about Saxon Pub?").

**When another skill called this one, finishing here returns control to that skill's next step.** This skill handles reads and writes only; it never ends the calling skill's job.

## Where the data lives

The master is a hosted database reached through the **Venue Master** connector: `https://venue-master.jfoss.workers.dev/mcp`. Nobody edits it directly. Every change is a submission that the service checks, applies or holds for review, and logs.

1. **Connector tools available** (lookup_venues, find_venues, get_venue, get_versions, add_venue, update_venue, update_venues, report_status, report_match_issue, archive_rechecks_due, submit_archive_recheck, get_submission, recent_changes, check_venues, master_stats): use them for everything below. `check_venues` brings a user's own list (ratings or notes keyed by Venue ID) up to date in one call; band-venue-fit Step 8 explains how to use it.
2. **Not available:** if the skills came from the music-venue-tools plugin, the connector is part of the plugin: ask the user to check it is enabled on the plugin's Connectors page (Customize > Plugins > music-venue-tools). Otherwise tell the user, in one short message, how to add it: in Claude, Settings > Connectors > Add custom connector, name "Venue Master", URL `https://venue-master.jfoss.workers.dev/mcp`. Until they do, read the latest copy from `https://venue-master.jfoss.workers.dev/export?state=TX` (CSV; add `&format=json` for JSON). Load it with a short script and bring back only the rows needed. Writes aren't possible without the connector: keep confirmed findings in the user's results file, marked "to submit", and submit them once the connector is added.

Never keep or update a separate copy of the master. Downloaded exports are snapshots for reading, not files to maintain.

**Reads cost the shared service money, so read only what you need.** The master is billed by rows read. Look venues up by name or Venue ID rather than pulling a whole state; page with `after`; and when re-reading data you've read before, send the version you saw (below) so an unchanged master answers in about two rows.

## What belongs in the master

- **Only venues with confirmed live music** (Live Music = Y, with a source). Venues that couldn't be confirmed are never added; their results go back to the user in their own results file.
- **Only venue facts, recorded as observations:** Venue ID, name, address, city, state, county, business type, links (website, Facebook, Instagram), public email and phone, booking contacts published for booking, live music evidence (Music Source, Source Type, Evidence Date, Checked Date, Confidence, Recurring, Open Mic Only, Check Notes), and research fields (Status, Booking Notes, Genres, Listening Environment, Act Size, Local vs Touring, Admission, Data Basis, Evidence, Last Researched). Write what the venue is ("Genre-specific venue (Country / honky-tonk)"), never a verdict for any act ("genre mismatch", "not a fit").
- **Never anything about an act or user:** no fit ratings, rationale, drive times, bookings, rates, act names, home addresses or preferences. The service rejects unknown fields and holds act-specific wording for review. Those details live in each user's act analysis file, which points to venues by Venue ID.

## Key fields

| Field | Values |
|---|---|
| Venue ID | Assigned by the service (TX-0001 and up, a separate sequence per state); never changed or reused |
| Status | Active, Not actively booking, Closed, Possibly closed. Put details and dates (for example "closed since Aug 2024 per its site") in Check Notes |
| Booking Notes | How the venue books, when notable: Genre-specific venue (with the genre), Books touring acts, Not a gig venue, Not a music venue (comedy). Blank = open booking |
| Inactive Since | Set by the service: when the venue entered Closed, Not actively booking or Possibly closed |
| Expected Reopen | For a closed venue that says it will reopen: YYYY-MM or YYYY-MM-DD |

Venues with Booking Notes "Not a gig venue" or "Not a music venue" always have Status "Not actively booking".

**The archive.** Venues that have been Closed or Not actively booking for 180 days, and are confirmed still closed by a monthly recheck, move to an archive, as do removed duplicates. Archived venues aren't leads: they don't appear in lists or exports, but lookups still recognize them so they're never added again. Field names in the tools are lowercase with underscores (business_name, local_vs_touring); the tool descriptions list each field's allowed values.

## Data versions

The master keeps a version number that goes up by one with every change users can see (a whole `update_venues` call counts as one change), plus a number per state: the version of the last change that touched that state. `find_venues` and `check_venues` return them as `version` and `state_version` / `state_versions`; `get_versions` returns just the numbers.

- Whenever a user or skill keeps venue data to use again (a saved list, a dashboard, an act analysis file), keep the version it came from next to it.
- To re-read, pass that number as `since_version`: if nothing in the state changed, the answer is `unchanged: true`; otherwise you get only the venues changed since, plus `removed` (Venue IDs archived or withdrawn since, with the reason).
- Never treat a version as a date or edit it; just store it and send it back.

## Look up venues

- **Named venues:** `lookup_venues` with name, city, state, and the address, website and phone when known, up to 100 per call (split longer lists). The service normalizes names (case, punctuation, initials such as "B.D." vs "BD", a leading "The", LLC/Inc, "&" vs "and") and also compares the distinctive part of each name (ignoring words like pub, bar, brewing, the city's name and "at <place>"), then returns, per venue:
  - **found:** use its facts.
  - **check_location:** same name, different address, or several locations in that city. Pick the right Venue ID by address; if it's still unclear, say so and treat the venue as not found for research.
  - **possible:** a close name in the same city, a similar name listed under another city in the state (neighboring city, or the venue moved), or a record sharing the street address, phone or website. Each candidate says what it matched on. Shared details alone don't prove it's the same venue (hotels, food halls and shopping centers share them), so confirm with the user as described in "Before adding" before treating it as found.
  - **archived:** it matches a venue archived as closed or not booking, or a removed duplicate. Don't add it and don't offer it as a lead. For a duplicate, use the record named in the note. For a closed venue, tell the user it closed (with the date archived); if they or your research have current evidence it reopened with live music, file `report_status` with status Active on the archived Venue ID, and the maintainer restores it.
  - **not_found:** pass it to the next step (usually `live-music-check`).
- **An area:** `find_venues` with state and county (or city). Page by passing `next_after` back as `after` until it's null (`offset` still works but reads every skipped row). The total is given on the first page only. Keep `state_version` from the first page for the next read of that area (see Data versions).
- **One venue in full:** `get_venue` with its Venue ID (all fields plus recent changes).
- **Before saying a venue isn't in the master, or adding it:** also run `find_venues` with the state and `name_contains` set to the most distinctive word of the name (for example `riley` for "BD Riley's Irish Pub"; skip words like pub, bar, the, brewing and the city's name). Venues are often stored under a fuller or older name ("B.D. RILEY'S PUB AT MUELLER") or a neighboring city. If the search turns it up, confirm with the address or website and use that Venue ID. Only report "not in the master" when both the lookup and this search come up empty.

Each result carries freshness flags: `evidence_stale` (no dated evidence, or older than 18 months) and `research_needed` (not researched in about six months). Stale records still count as found; the caller decides whether to recheck. For lists, report counts (found, check location, possible, archived, not found) rather than listing every row in chat.

## Before adding: no duplicates

Duplicate records damage the master for everyone, so never add a venue while there's any chance it's already there.

1. **Look again with what research found.** Run `lookup_venues` with the official name, street address, website and phone gathered during research, plus the name search above. Research often turns up details that match a record the user's wording didn't.
2. **Nothing matches:** go ahead and add.
3. **A record matches now that didn't before:** don't add. Tell the user, in one short message, which record you found and ask whether it's the venue they mean, for example: "The master already has this as TX-0002, B.D. RILEY'S PUB AT MUELLER, 1905 Aldrich St. Is that the venue you mean?" For a list, ask about all such venues in one message.
   - **Yes:** use the existing record (update it with any new facts instead of adding). Then call `report_match_issue` with kind `missed_match`: what the user searched, what the first lookup returned, the Venue ID, what it matched on, user_confirmed `yes`, and a one- or two-sentence note on why the first search probably missed it (for example: stored name uses initials and extra words; listed under a neighboring city; user gave a nickname).
   - **No:** treat it as a different venue and add it (the service may hold it for the maintainer's review). Then call `report_match_issue` with kind `false_match`, user_confirmed `no`, and a note on why the records looked alike.
4. **A possible match from the first lookup** that the user confirms needs no report: the search worked. If they say it's a different venue, file a `false_match`.
5. Keep the report out of the user's way: tell them only "Thanks, I've flagged that so the search improves." The diagnosis goes to the maintainer.

## Add a venue

Only when `live-music-check` (or a confirmed dispute) has established live music, and only after "Before adding" found no existing record. Call `add_venue` with:

- **Required:** business_name, city, state, music_source, source_type, evidence_date (within 18 months, or a dated musician report), confidence (High or Medium).
- **County:** work it out from the street address (users rarely know it, and some cities span two counties). The service fills it only for cities already in the master, and rejects the add otherwise.
- Anything else known: address, website, Facebook, Instagram, email, phone, booking contacts, and research fields if they were gathered.

The service assigns the Venue ID; use the returned ID in the user's results.

**An add is not a finished record.** After an applied add of a single venue, return to the caller: `live-music-check` continues to its Step 7A, which runs `venue-research` on the new ID before answering. If a user asked to add a venue directly (no calling skill), run `venue-research` on it yourself before reporting back.

## Update a venue

- **One venue:** `update_venue` with the Venue ID and only the fields you have evidence for, plus a short check_notes saying what was checked. Notes are appended with the date, never replaced.
- **Several venues at once** (a research pass, or any time you have findings for more than one venue): `update_venues` with up to 50 entries per call, each { venue_id, fields } with the same fields and rules as `update_venue`. Split larger passes into calls of 50. Each venue is checked on its own and gets its own result (applied, held, rejected or no_change) and submission ID; everything applied in the call lands together as one change, under one `batch_id`. Each Venue ID may appear once per call (merge its fields). Prefer it to many `update_venue` calls: one request instead of one per venue.
- **Openings and closures:** `report_status` with the new Status and the evidence (what, where, when) in check_notes. If a closed venue says when it will reopen, include `expected_reopen` (YYYY-MM or YYYY-MM-DD); archiving then waits until that date passes.

The service enforces the merge rules: filled fields can't be blanked; older live music evidence never replaces newer (the old source is noted); Listening Environment is recomputed when Genres change without it; Last Researched and Checked Date are set automatically.

## Archive rechecks (monthly)

Run when asked (normally by a monthly scheduled task) to keep the archive accurate.

1. Call `archive_rechecks_due`. It lists venues Closed or Not actively booking for 180 days or more that need a recheck, with their notes and any wording that hints at a reopening.
2. For each, research its current state at this location: its website (gone, closure notice, or current events), its listing on maps and event platforms, recent local news, and any reopening plans, especially where `reopen_hints` appear. Use up to 3 web calls per venue.
3. Call `submit_archive_recheck` with one finding and the evidence (what you checked, what it showed, the source):
   - **still_closed:** closed, gone or not booking music, with no sign of reopening. The service archives it at its next daily run.
   - **reopen_planned:** it says it will reopen; give `expected_reopen`. Archiving waits until then and a new recheck.
   - **reopened:** open and hosting live music again. Also file `report_status` with status Active and the evidence.
   - **inconclusive:** couldn't tell either way. Nothing is archived; it comes up again next month.
4. Be conservative: archive only on clear evidence. When in doubt, use inconclusive.
5. Report only changes: venues found reopened or with a reopening date. Venues still closed or inconclusive need no message; the maintainer's phone alerts cover archiving.

## Outcomes

Every write returns a submission ID (`update_venues` returns one per venue, plus a `batch_id` and counts) and one of:

- **applied:** live for everyone. Report the Venue ID and what changed.
- **held:** waiting for the maintainer's review. Closures, Not actively booking, changes to facts confirmed by hand, possible duplicates, matches to archived records, reopening reports on archived venues and act-specific wording are held. Tell the user it was submitted for review; nothing changed yet. `get_submission` shows what happened later.
- **rejected:** fix what the reasons say and resubmit if the evidence supports it (for example add county, a newer evidence date or allowed values). Don't resubmit unchanged, and don't reword facts to get past a rule.
- **rejected or held with `existing_records`:** the service recognized the venue as one already in the master or the archive (it has logged this for the maintainer, so no report is needed). Treat it as step 3 of "Before adding": confirm the existing record with the user and use it; never try to add it again under a different name. If the user already said it's a different venue, tell them it couldn't be added automatically and the maintainer will look at it (the `false_match` report you filed covers it).
- **no_change:** the master already had it.

If a write fails with a limit message, stop writing for the day and keep the remaining findings in the user's results, marked "to submit". (`update_venues` counts each venue toward the daily limit.)

Tell the user what was added, updated, held and rejected (counts for lists), with the Venue IDs.