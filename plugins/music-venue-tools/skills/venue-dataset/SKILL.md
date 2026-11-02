---
name: "venue-dataset"
description: "Look up, add and update venues in the shared venue master (the TX venue dataset), the live list of verified live music venue leads, through the Venue Master connector. Used by live-music-check, venue-research and band-venue-fit."
---

# Venue dataset

The one way to read and write the shared **venue master** (also called the TX venue dataset): a list of **verified live music venue leads** shared by every musician who uses these skills. It's live: an update one user makes reaches everyone straight away. Other skills call this skill instead of handling the data themselves:

- `live-music-check`: looks venues up first; adds venues it confirms have live music.
- `venue-research`: reads venue facts and submits what it learns.
- `band-venue-fit`: reads venue facts and joins them, by Venue ID, to the user's own act analysis file; never writes to the master.

A user can also ask directly ("Is Gruene Hall in the venue dataset?", "What do we know about Saxon Pub?").

## Where the data lives

The master is a hosted database reached through the **Venue Master** connector: `https://venue-master.jfoss.workers.dev/mcp`. Nobody edits it directly. Every change is a submission that the service checks, applies or holds for review, and logs.

1. **Connector tools available** (lookup_venues, find_venues, get_venue, add_venue, update_venue, report_status, get_submission, recent_changes, master_stats): use them for everything below.
2. **Not available:** if the skills came from the music-venue-tools plugin, the connector is part of the plugin: ask the user to check it is enabled on the plugin's Connectors page (Customize > Plugins > music-venue-tools). Otherwise tell the user, in one short message, how to add it: in Claude, Settings > Connectors > Add custom connector, name "Venue Master", URL `https://venue-master.jfoss.workers.dev/mcp`. Until they do, read the latest copy from `https://venue-master.jfoss.workers.dev/export?state=TX` (CSV; add `&format=json` for JSON). Load it with a short script and bring back only the rows needed. Writes aren't possible without the connector: keep confirmed findings in the user's results file, marked "to submit", and submit them once the connector is added.

Never keep or update a separate copy of the master. Downloaded exports are snapshots for reading, not files to maintain.

## What belongs in the master

- **Only venues with confirmed live music** (Live Music = Y, with a source). Venues that couldn't be confirmed are never added; their results go back to the user in their own results file.
- **Only venue facts, recorded as observations:** Venue ID, name, address, city, state, county, business type, links, booking contacts published for booking, live music evidence (Music Source, Source Type, Evidence Date, Checked Date, Confidence, Recurring, Open Mic Only, Check Notes), and research fields (Status, Booking Notes, Genres, Listening Environment, Act Size, Local vs Touring, Admission, Data Basis, Evidence, Last Researched). Write what the venue is ("Genre-specific venue (Country / honky-tonk)"), never a verdict for any act ("genre mismatch", "not a fit").
- **Never anything about an act or user:** no fit ratings, rationale, drive times, bookings, rates, act names, home addresses or preferences. The service rejects unknown fields and holds act-specific wording for review. Those details live in each user's act analysis file, which points to venues by Venue ID.

## Key fields

| Field | Values |
|---|---|
| Venue ID | Assigned by the service (TX-0001 and up, a separate sequence per state); never changed or reused |
| Status | Active, Not actively booking, Closed, Possibly closed. Put details and dates (for example "closed since Aug 2024 per its site") in Check Notes |
| Booking Notes | How the venue books, when notable: Genre-specific venue (with the genre), Books touring acts, Not a gig venue, Not a music venue (comedy). Blank = open booking |

Venues with Booking Notes "Not a gig venue" or "Not a music venue" always have Status "Not actively booking". Field names in the tools are lowercase with underscores (business_name, local_vs_touring); the tool descriptions list each field's allowed values.

## Look up venues

- **Named venues:** `lookup_venues` with name, city, state and the address when known, up to 100 per call (split longer lists). The service normalizes names (case, punctuation, a leading "The", LLC/Inc, "&" vs "and") and returns, per venue:
  - **found:** use its facts.
  - **check_location:** same name, different address, or several locations in that city. Pick the right Venue ID by address; if it's still unclear, say so and treat the venue as not found for research.
  - **possible:** a close name in the same city. Confirm with the address or website before treating it as found.
  - **not_found:** pass it to the next step (usually `live-music-check`).
- **An area:** `find_venues` with state and county (or city), paging with next_offset. Use it to pull a region for research or rating.
- **One venue in full:** `get_venue` with its Venue ID (all fields plus recent changes).

Each result carries freshness flags: `evidence_stale` (no dated evidence, or older than 18 months) and `research_needed` (not researched in about six months). Stale records still count as found; the caller decides whether to recheck. For lists, report counts (found, check location, possible, not found) rather than listing every row in chat.

## Add a venue

Only when `live-music-check` (or a confirmed dispute) has established live music, and only after a lookup came back not_found. Call `add_venue` with:

- **Required:** business_name, city, state, music_source, source_type, evidence_date (within 18 months, or a dated musician report), confidence (High or Medium).
- **County:** work it out from the street address (users rarely know it, and some cities span two counties). The service fills it only for cities already in the master, and rejects the add otherwise.
- Anything else known: address, links, booking contacts, and research fields if they were gathered.

The service assigns the Venue ID; use the returned ID in the user's results.

## Update a venue

- **Facts and evidence:** `update_venue` with the Venue ID and only the fields you have evidence for, plus a short check_notes saying what was checked. Notes are appended with the date, never replaced.
- **Openings and closures:** `report_status` with the new Status and the evidence (what, where, when) in check_notes.

The service enforces the merge rules: filled fields can't be blanked; older live music evidence never replaces newer (the old source is noted); Listening Environment is recomputed when Genres change without it; Last Researched and Checked Date are set automatically.

## Outcomes

Every write returns a submission ID and one of:

- **applied:** live for everyone. Report the Venue ID and what changed.
- **held:** waiting for the maintainer's review. Closures, Not actively booking, changes to facts confirmed by hand, possible duplicates and act-specific wording are held. Tell the user it was submitted for review; nothing changed yet. `get_submission` shows what happened later.
- **rejected:** fix what the reasons say and resubmit if the evidence supports it (for example add county, a newer evidence date or allowed values). Don't resubmit unchanged, and don't reword facts to get past a rule.
- **no_change:** the master already had it.

If a write fails with a limit message, stop writing for the day and keep the remaining findings in the user's results, marked "to submit".

Tell the user what was added, updated, held and rejected (counts for lists), with the Venue IDs.