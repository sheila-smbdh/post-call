# Closer Deep Dive

A skill that pulls a closer's post-discovery-call behavior out of Close CRM,
writes it to a Google Sheet tracker, and produces an analysis report.

Fourth skill, separate from `extract-pre-call-behaviors`,
`analyze-pre-call-behaviors`, and `new-hire-tracker-sync`.

---

## Status — 2026-09-10

Design and feasibility work is **complete and verified against live data**.
Implementation is **blocked on one environment change** (below).

### Settled

| Decision | Answer |
|---|---|
| Pilot scope | Harry Whyte, "Next 7 Days" stage — 5 leads |
| Tracker location | New sheet "Closer Deep Dive Tracker", one tab per closer |
| Deliverables | Narrative write-up + stats summary + published HTML report |
| Duration source | Close REST API, nested Zoom integration data |
| Schema | 27 columns — see `docs/tracker-schema.md` |

### Blocked on

**`api.close.com` must be added to the environment's network allowlist**, and
the Close API key stored as the `CLOSE_API_KEY` environment variable.

The egress proxy currently rejects `api.close.com:443` with a 403 at the
CONNECT tunnel (organization policy). A control request to another host
returns 200, so this is host-specific policy — not a network fault and not a
credential problem. Environment configuration is documented at
https://code.claude.com/docs/en/claude-code-on-the-web

Both changes take effect in a **new session**.

---

## Why the REST API is required

The Close MCP connector exposes 12 fields on a meeting; the REST API exposes
48. Actual call duration is only in the REST response, and only nested inside
`integrations[].integration_data`.

Every field that *looks* like it should hold actual duration does not:

| Field | Reference call | Reality |
|---|---|---|
| `duration` (connector + REST) | 2700 | Scheduled Calendly length |
| `ends_at` (REST) | 13:45 | `starts_at + duration`, not actual end |
| `actual_duration` (REST) | *empty* | Exists but unpopulated |
| `integrations[].integration_data` | 12:59:31 → 14:11:22 | **The real thing** |

Reference call (Anthony Mariani, 2026-09-04): actual **71m51s** against a 45m
booking. The UI label "1h 11m" is truncated; the sibling `duration: 72` is
rounded. Only `end_time - start_time` is exact.

---

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

## Next steps once unblocked

1. Verify `api.close.com` is reachable and `CLOSE_API_KEY` is set.
2. Build extraction: opportunities → first-call meeting → Zoom timing →
   activity trail split by `direction`.
3. Create "Closer Deep Dive Tracker" with a "Harry Whyte" tab, 27 columns.
4. Run the five pilot leads.
5. Produce the narrative write-up, stats summary, and HTML report.

Read `docs/tracker-schema.md` first — it carries the eligibility gate, the
column definitions, and six CRM traps that were confirmed against live data.
Those traps are the expensive part of this work; do not rediscover them.

---

## Security

No credential is ever committed. The skill reads `CLOSE_API_KEY` from the
environment. `.gitignore` covers `.env`, key files, and token files.
