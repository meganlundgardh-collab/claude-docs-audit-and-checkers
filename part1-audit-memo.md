# Part 1 — Audit Memo: Skills, Plugins, Connectors

**Methodology:** Findings come from a full manual review of the Skills/Plugins/Connectors slice — the same ~50-page scrape later used to build the Part 3 checkers — cross-referenced against llms.txt and sitemap.xml to catch pages one index missed and the other didn't, then validated against the live pages to rule out crawler artifacts. Findings are prioritized by user-facing consequence first: P0 covers active risk or a broken path a reader can hit today; P1 covers the structural causes behind repeated drift, which don't break anything by themselves but keep generating P0s. Out of scope: platform.claude.com/docs and code.claude.com/docs — adjacent surfaces with their own ownership, not part of this slice.

---

## What's actually wrong (prioritized)

### P0 — Active Risks & Broken Paths

- **Missing pre-install risk warnings for Plugins:** Plugin installs carry no risk warning, despite plugins being riskier than connectors (executing local scripts and automated hooks). Plugin risk is addressed on a page written for submitters (plugins/submit.md), not installers. *Fix:* Apply the existing `<Warning>` pattern currently used in the M365 office-agents/connectors-and-skills.md docs.
- **Misleading admin deployment links:** cowork/guide/plugins.md links to /docs/cowork/3p/extensions three separate times, all with the same anchor text ("MCP, plugins, skills, and hooks"). That page is scoped to Claude Desktop on third-party gateways (e.g., Bedrock), sending Cowork admins to the wrong deployment model's documentation entirely. An admin following any of the three either hits a mismatch or starts applying the wrong deployment model's config guidance to their org.

### P1 — Root causes (structural drift)

- **No canonical, site-owned definitions:** Skills and Plugins lack canonical definition pages on claude.com/docs (they point to Claude Code docs instead). As a result, surface pages (Cowork, Gov) re-derive definitions from scratch.
- **Component drift:** Because definitions are re-derived, "what a plugin contains" is answered differently across four different pages. (e.g., government/desktop/plugins.md claims plugins include "hooks," a capability not listed in the canonical overview).
- **Index fragmentation:** The site's machine-generated indexes disagree. sitemap.xml is missing four real, live pages that llms.txt correctly lists, highlighting a risk for LLM-driven discovery.
- **Structural imbalance:** Skills and Plugins share 4 indexed pages combined; Connectors have 37. Connectors is bloated with sub-features, while Skills/Plugins lack baseline authoring guides.

---

## What should be deleted or merged

- **Merge re-derived preambles:** Trim the repetitive "what is a plugin/skill" intros on surface pages (Gov, Cowork) down to 1-2 sentences, and link to a canonical definition.
- **Merge plugins/submit.md overlap:** Fold the "What makes a good plugin" section into plugins/overview.md. It substantially overlaps overview.md's "How plugins compose capabilities" table — same concept, explained twice with two framings. Keep submit.md strictly focused on submission mechanics.
- **Restructure & Redirect connectors/getting-started.md:** This page mixes durable content (connector-specific tips) with a UI click-path and troubleshooting accordion that goes stale when the UI changes.
  - Merge the durable "tips" into connectors/overview.md.
  - Delete the click-paths, following the estate's own precedent (relying on Support articles instead, matching the M365 docs pattern).
  - 301 redirect the URL to overview.md and fix connectors/overview.md's "Next steps" card, which currently points to the page being deleted. Update llms.txt to drop the merged entry.

---

## Proposed information architecture

The current shape is flat — a page list per primitive plus a number of surface sections each independently deciding how much to re-explain. To solve the imbalance and prevent semantic drift, reorganize all three primitives (Skills, Plugins, Connectors) into a strict three-tier pattern.

**Tier 1: Concepts (Canonical).** One page per primitive defining what it is, how it relates to the other two, and its availability. Surface pages must link here, not re-derive. Includes a unified trust/verification model for all three primitives.

**Tier 2: Build / Author.** Standardized `/<primitive>/build/` paths. (This fixes the Connectors bloat by nesting MCP Tunnels and Apps as sub-features, not top-level siblings).

**Tier 3: Use in [Surface].** Pages for Cowork, Gov, M365, etc. Restricted *only* to surface-specific installation, UI, and admin limits.

**Note:** Claude Tag requires a separate, task-based pattern due to its unique taxonomy.

---

## What to measure / instrument

**Leading Indicators (Automated via Part 3 Checker):**

- **Internal link validity:** % of internal links that resolve to a confirmed-live page. Target: 0 broken/orphaned links, checked daily in CI.
- **Definition consistency:** Track the number of conflicting restatements of a term that should have one definition. Target: trending from 4 toward 1 canonical source + N linking pages, 0 restating independently.

**Lagging Indicators (On-Page Reader Feedback Widget):**

- **Findability & Clarity:** Track the existing "Yes / No" widget categories ("easy to find what I needed" vs "make it easier to find" and "guide worked as expected").
  - A light tagging pass on the "Something else" free text, mapped against taxonomy, turns it into an early-warning signal.
- **Instrumentation constraint:** If the current widget only provides site-wide aggregates, my first engineering ask would be enabling URL-level querying to measure before/after category shifts on the specific pages we restructure.

**Other Lagging Signals:**

- **Support-ticket volume/tags** referencing this slice, before/after fixes ship.
- **Install-funnel completion:** for connectors and plugins: track page view → connect/install click → success
- **Page-level engagement:** time-on-page bounce/exit rate, before/after.
