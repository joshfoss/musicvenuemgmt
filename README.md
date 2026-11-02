# Music venue management for musicians

Claude tools that help bands and solo acts find live music venues, check they really host live music, research how they book, and rate how well each fits the act. Everyone works from one shared, live venue master, so research one musician does helps everyone.

## Install in Claude Cowork (recommended)

1. In Claude, open **Customize > Plugins**.
2. Choose **Add marketplace** and enter `joshfoss/musicvenuemgmt`.
3. Install **music-venue-tools**.
4. Open the plugin and check the **Venue Master** connector is enabled.

To get updates later, click **Update** on the marketplace.

In Claude Code: `/plugin install music-venue-tools --marketplace joshfoss/musicvenuemgmt`

## Use in Claude chat (skills only)

Plugins work in Cowork and Claude Code. For regular Claude chat:

1. Download the four ZIP files from the `skill-zips` folder.
2. In Claude, go to **Customize > Skills**, click **+**, then **Upload a skill**, once per ZIP.
3. Add the connector: **Settings > Connectors > Add custom connector**, name `Venue Master`, URL `https://venue-master.jfoss.workers.dev/mcp`.

## What you can ask

- "Would my band be a good fit for Gruene Hall?"
- "Rate the Hays County venues for my band."
- "Does Backporch Social have live music?"
- "Research the Williamson County venues."
- "That bar closed last month." (reports it for review)
- "We booked Firehouse Lounge for $400." (updates your own list)

See `plugins/music-venue-tools/README.md` for what's shared and what stays private.
