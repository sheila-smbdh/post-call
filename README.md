# Closer Deep Dive

A skill that pulls a closer's post-discovery-call behavior out of Close CRM,
writes it to a Google Sheet tracker, and produces an analysis report.

Fourth skill, separate from `extract-pre-call-behaviors`,
`analyze-pre-call-behaviors`, and `new-hire-tracker-sync`.

---

## Status — 2026-09-11

Design and feasibility work is **complete and verified against live data**.
Implementation is **fully unblocked** — ready to build.

Call duration is not tracked. The eligibility gate reads closer-driven CRM
state after the scheduled time, which answers the only question the analysis
needs: did the call happen? Everything the tracker measures is post-call
closer behavior, none of which depends on how long the call ran.

Every column is served by the Close MCP connector. There is no REST API
dependency. See "Did the call happen?" in `docs/tracker-schema.md`.

### Settled

| Decision | Answer |
|---|---|
| Pilot scope | Harry Whyte, "Next 7 Days" stage — 5 leads |
| Tracker location | New sheet "Closer Deep Dive Tracker", one tab per closer |
| Deliverables | Narrative write-up + stats summary + published HTML report |
| Eligibility gate | Opportunity status change, not duration (2026-09-11) |
| Call duration | Not tracked — dropped 2026-09-11 |
| Schema | 25 columns — see `docs/tracker-schema.md` |

## The five pilot leads

Harry Whyte, Sales pipeline (`pipe_6BtcTllrXJwFF7nSvcRrLu`),
"Next 7 Days" (`stat_ve3fYEwPJNbmssRUK7gVESah8r0kQYSrmWid1GMqC8a`).

| Lead | First call (ET) | Stage moved by |
|---|---|---|
| Will Morgan | Aug 31, 5:00pm | Helen Guo |
| Ming Su | Sep 3, 1:00pm | Harry |
| Anthony Mariani | Sep 4, 9:00am | Harry |
| Ari Brownstein | Sep 9, 3:00pm | Harry |
| Kevin Gindi | Sep 10, 4:00pm | Harry |

Harry's user id: `user_f3vkQZe4xJvRsPV9L6cU1UOHk7nwxStq94aMqJYLqzm`

---

## Next steps

1. Build extraction: opportunities → first-call meeting → activity trail
   split by `direction`. Apply the CRM-state gate for eligibility.
2. Create "Closer Deep Dive Tracker" with a "Harry Whyte" tab, 25 columns.
3. Run the five pilot leads.
4. Produce the narrative write-up, stats summary, and HTML report.

Read `docs/tracker-schema.md` first — it carries the eligibility gate, the
column definitions, and eight CRM traps that were confirmed against live
data. Those traps are the expensive part of this work; do not rediscover
them.

---

## Security

No credential is ever committed. The skill reads Close through the MCP
connector and holds no API key of its own. `.gitignore` covers `.env`, key
files, and token files.
