# Changelog

What changed in the **music-venue-tools** plugin and its skills, newest first. To get the latest version, click **Update** on the `joshfoss/musicvenuemgmt` marketplace in Claude (Customize > Plugins), or download the ZIPs in `skill-zips` again if you use Claude chat.

The skills work with the shared Venue Master service; where a version needs a newer service, it says so. The service is updated centrally, so there's nothing for you to install.

## 0.1.8 (2026-11-04)

Needs Venue Master service 1.5 (live).

- **New venues arrive filled in.** When live-music-check confirms and adds a single new venue, it now researches it before answering: website, Facebook, Instagram, email, phone and booking details, starting from the venue's own homepage. The answer tells you what was found and what's still missing.
- **Clearer answers.** live-music-check only says a source was "not found" when it actually checked it; a source it skipped is "not checked". It also searches Bandsintown venue pages on their own and opens them in Claude's browser, since search tools often miss them.
- **venue-research records links.** Website, Facebook, Instagram, email and phone are now research fields, read from the venue's homepage (including icon-only social links).
- **Research passes in one call.** venue-research submits a batch of findings with the new `update_venues` tool (up to 50 venues per call) instead of one call per venue. Each venue is still checked on its own.
- **Cheaper list syncs.** band-venue-fit's sync (Step 8) saves the master's data version in your file's Sync sheet and sends it back next time. A week with no changes now costs the shared service about two row reads instead of reading every venue in the state. Your first sync after updating does one full check to record the version.
- venue-dataset documents data versions, `update_venues` and paging with `after`.

## 0.1.7 (2026-11-03)

- **live-music-check finds more evidence:** Bandsintown venue pages (upcoming shows, past shows and fan reviews), snippets from the venue's own Instagram and Facebook, Yelp and Google "Live music" attributes and reviews mentioning live music, and a Google search in Claude's browser for single venues.
- Single venues get a bigger research budget (about 8 web calls) and are never reported as N while a promising lead is unchecked.
- Evidence older than 18 months is reported to you as "historic only" rather than treated as confirmation.
- When research can't confirm live music for your own act's request, your act's own gig history is used as a fallback, and you're asked whether you've played a confirmed venue.

## 0.1.6 (2026-11-03)

- band-venue-fit: when the weekly sync calls the service directly (no connector in the session), it sends a User-Agent header, which the service requires.

## 0.1.5 (2026-11-03)

- band-venue-fit: optional weekly sync of a saved venue list as a scheduled task. Quiet weeks reply with one line.

## 0.1.4 (2026-11-03)

Needs Venue Master service 1.4.

- **Keep your list current.** band-venue-fit Step 8 syncs your saved list with the master in one call (`check_venues`): archived venues removed, closed ones blocked, changed ones re-rated, new venues in your area added for rating.
- **No duplicates.** Before adding a venue, the skills look it up again with everything research found (name, address, website, phone). A match is confirmed with you first.
- **Archive.** Venues closed or not booking for six months, and removed duplicates, move to an archive; lookups recognize them so they're never added again. A monthly recheck confirms closures first.
- live-music-check: a venue's own website describing ongoing live music counts as current evidence; confirmed venues are added without asking.
- The Venue Master connector is bundled with the plugin (no separate setup in Cowork or Claude Code).

## 0.1.1 (2026-11-02)

- Better venue name matching: initials ("B.D." vs "BD"), and the distinctive words of a name ignoring words like pub and bar, the city's name and "at <place>". Skills also run a name search before saying a venue isn't in the master.

## 0.1.0 (2026-11-02)

- First release: venue-dataset, live-music-check, venue-research and band-venue-fit, the Venue Master connector, and the venue list template.
