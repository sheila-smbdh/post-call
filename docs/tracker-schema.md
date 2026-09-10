# Closer Deep Dive — Tracker Column Schema

One row per **eligible lead**. One tab per closer in the
**Closer Deep Dive Tracker** Google Sheet.

Grounded against 5 real leads in Harry Whyte's "Next 7 Days" stage
(Anthony Mariani, Kevin Gindi, Ari Brownstein, Ming Su, Will Morgan),
verified via the Close MCP connector on 2026-09-10.

---

## Eligibility gate

A lead enters the tracker only if **all** hold:

1. Opportunity is in the Sales pipeline (`pipe_6BtcTllrXJwFF7nSvcRrLu`)
   and owned by the target closer.
2. Stage is **not** `Discovery Call Scheduled` and **not** `No Show/Cancel`
   — those leads have not had a call yet.
3. A meeting activity exists with the closer as `user_id`, in the past.
4. The call is confirmed to have really happened (see Duration, below).

Scope is the **first discovery call only**. Everything measured starts at
that call. Earlier setter/intro-call activity is context, not measurement.

---

## Columns

### A. Identity (6)
| # | Column | Source |
|---|---|---|
| 1 | Closer | opportunity `user_name` |
| 2 | Lead Name | contact `contact_name` |
| 3 | Close Lead URL | built from `lead_id` |
| 4 | Current Stage | opportunity `status_label` |
| 5 | Setter / Booked By | earlier intro-call meeting `user_id` |
| 6 | Extract Date | run timestamp |

### B. The first discovery call (7)
| # | Column | Source | Note |
|---|---|---|---|
| 7 | Call Date (ET) | meeting `starts_at` → America/New_York | |
| 8 | Call Time (ET) | same | |
| 9 | Day of Week | derived | |
| 10 | Scheduled Duration (min) | meeting `duration` / 60 | always 45 in sample |
| 11 | **Actual Duration (min)** | **PENDING API-key test** | see Open Question |
| 12 | Duration Source | `zoom-api` / `granola-proxy` / `manual` / `unavailable` | audit trail |
| 13 | Transcript Word Count | Granola transcript | 12,843 for Anthony |

### C. Outcome (5)
| # | Column | Note |
|---|---|---|
| 14 | Stage Before Call | from `opportunity_status_change` |
| 15 | Stage After Call | |
| 16 | Stage Change (ET) | |
| 17 | Lag: Call End → Stage Change | Harry logs same-day in 4/5 |
| 18 | **Stage Moved By** | Will Morgan's was moved by Helen Guo, not Harry — do not credit the closer blindly |

### D. Post-call follow-up — the core (6)
| # | Column | Note |
|---|---|---|
| 19 | First Post-Call Touch Channel | SMS / email / call |
| 20 | First Post-Call Touch (ET) | |
| 21 | **Lag: Call End → First Touch** | Anthony = **7h27m**, not "immediate" |
| 22 | Post-Call Material Sent? | Y/N + what |
| 23 | Post-Call Email (ET) | |
| 24 | Material Described | e.g. "resources + agreement" |

### E. Channel volume (5)
| # | Column |
|---|---|
| 25 | SMS Out / SMS In |
| 26 | Email Out / Email In |
| 27 | Follow-up Zoom calls booked |
| 28 | Phone dials (closer → lead) |
| 29 | Total Exchanges (Closer n / Lead n) |

### F. Responsiveness (5)
| # | Column | Note |
|---|---|---|
| 30 | Lead First Reply (ET) | |
| 31 | Closer Median Response Time | Anthony thread: ~3m and ~1h17m |
| 32 | Closer Fastest Response | 2m53s |
| 33 | Closer Slowest Response | |
| 34 | Send Hours (ET) | hour-of-day distribution; Harry confirmed at 5:30pm |

### G. Chase / recovery (5)
| # | Column | Note |
|---|---|---|
| 35 | Days Call → Closer Re-initiation | Anthony = 4.8 days |
| 36 | Unanswered Closer Re-touches | measures persistence |
| 37 | Follow-up Call Booked? | Ari = Sep 28, 11:00am ET |
| 38 | Exchanges to Book Follow-up | Anthony = 6 |
| 39 | Who Went Silent Last | closer / lead — the drop-off point |

**39 columns.**

---

## Verified CRM traps

These are real, observed in the 5-lead sample. The skill must handle each.

1. **`duration` is scheduled, not actual — and so is `ends_at`.** Via the MCP
   connector, the meeting object exposes one duration field returning the
   booked Calendly length (2700s = 45m). Via the REST API, `ends_at` is
   likewise just `starts_at + duration` (13:45 for a call that ran to 14:11),
   and the top-level `actual_duration` field exists but is **empty**.

   Actual duration lives in `integrations[].integration_data` — see
   "Actual call duration", below.

2. **`user_id` on inbound messages is the closer, not the sender.** Filtering
   by `user_id` mislabels every lead reply as closer-authored. Always split
   with the `direction` filter (`inbound` / `outbound`).

3. **Ghost duplicate meetings from departed reps.** Anthony had a second
   Sep 4 meeting ("Anthony Mariani and Ezra Isla", 10:00am ET) that the setter
   told him to cancel; it was never marked cancelled. Ezra has left the
   company, and his calls were being cancelled and reassigned — Harry took
   this one over. Naive counting sees two discovery calls.

   **Primary rule:** the first-call meeting must have
   `user_id == target closer`. The Ezra record fails this outright, so no
   proximity heuristic is needed. Apply this before any de-duplication.

   **Secondary signal:** a meeting whose `user_id` is absent from the active
   `org_users` roster belongs to a departed rep (Ezra's
   `user_IZsgcW3rxd1herqkr9P2OIkDvzekDImciId3Ni6nh4e` is not in the roster).
   Use this only to *annotate* the row — "duplicate booking with a departed
   rep ignored" — never as an exclusion rule on its own: a closer who has
   since left the company would otherwise return zero rows, making historical
   analysis impossible.

4. **Calendly email subject times are not ET.** Anthony's follow-up subject
   reads "11:00 Mon, 14 Sep 2026" while the meeting is 16:00 UTC = 12:00pm ET
   and the closer's own SMS says "Monday 12 pm EST". Never parse times from
   email subjects — use meeting `starts_at` converted to America/New_York.
   Confirmed with Sheila 2026-09-10: the subject line was the source of the
   discrepancy in her manual read, and is not a reliable source.

5. **Stage changes are not always the closer's.** Will Morgan's move was made
   by Helen Guo. Record the actor.

6. **Granola coverage is partial.** 3 of 5 sample calls had transcripts
   (Anthony, Kevin Gindi, Ari Brownstein); Ming Su and Will Morgan had none.
   Join is by lead name + ET datetime — there is no shared ID.

---

## Actual call duration — RESOLVED 2026-09-10

Actual duration **is** retrievable, but only through the Close REST API, and
only from a nested field the MCP connector does not project.

`GET /api/v1/activity/meeting/{id}/` → `integrations[]` → the entry with
`integration_name == "zoom"` → `integration_data`:

```json
{
  "start_time": "2026-09-04T12:59:31+00:00",
  "end_time":   "2026-09-04T14:11:22+00:00",
  "duration":   72,
  "participants": [
    {"zoom_id": "rZO_-CF-SdSATRK3TuacaA", "name": "Harry Whyte"},
    {"zoom_id": "", "name": "anthony"},
    {"zoom_id": "", "name": "anthony"}
  ],
  "processing_status": "processing"
}
```

### Rules

- **Compute duration as `end_time - start_time`**, not from the `duration`
  field. For the reference call that is 71m51s, while `duration` reports 72 —
  it is rounded. The Close UI shows "1h 11m" (truncated). All three describe
  the same call; only the computed value is exact.
- **Do not gate on `processing_status`.** It still read `"processing"` six days
  after the call. Treat the timing data as usable as soon as it is present.
- **De-duplicate `participants` by name.** The reference call lists "anthony"
  twice — a rejoin or second device — which would otherwise inflate attendee
  counts.
- **`participants` is an independent attendance signal.** It shows who really
  joined the Zoom, regardless of stage labels, so it can catch a booked call
  where the lead never appeared.
- Fall back to the Granola/hybrid path when the zoom integration entry is
  absent, rather than dropping the lead.

### Fields that exist but are empty (do not design around them)

Confirmed empty on the reference meeting: `actual_duration`, `user_note`,
`user_note_html`, `outcome_id`, `outcome_reason`,
`outcome_autofill_reasoning`, `summary`, `notetaker_id`, `attached_call_ids`,
`attendees`. Useful only if the organization later starts populating them.

---

## Superseded — earlier open question on duration

Sheila's original gate is "the call must have run >15 minutes to count as a
real discovery call." Actual duration is visible in the Close **UI** as Zoom
recording assets (Gallery View / Shared Screen / Active Speaker / Audio Only,
each labelled e.g. "1h 11m") but is **not exposed by the Close MCP
connector**: the meeting schema has no recordings field, and both
`fetch_meeting_transcript` and `fetch_call` return Not Found.

Decision: test whether the Close **REST API** (`GET /activity/meeting/{id}`)
exposes recording objects that the OAuth connector hides.

- If yes → column 11 is populated automatically, `Duration Source = zoom-api`.
- If no → fall back to the hybrid: Granola transcript presence + word count
  gates eligibility, and the leads Granola missed are flagged for manual fill.

The API key is read from the `CLOSE_API_KEY` environment variable. It is
never written to source, config, or any commit.

### Egress constraint (tested 2026-09-10)

`api.close.com` is **blocked by the organization egress policy** in the Claude
Code remote environment. The proxy rejects the CONNECT tunnel with 403
(`connect_rejected`, host `api.close.com:443`), while a control request to
`api.github.com` returns 200 — so this is host-specific policy, not a network
fault, and not a bad key.

Consequences for the skill's design:

- The skill **cannot depend on the Close REST API** when run from a Claude
  Code web/remote session. Only the OAuth MCP connector is reachable, and
  that connector does not expose recording objects.
- Using the REST API would require adding `api.close.com` to the environment's
  allowed hosts in the network policy. Until that happens, the REST route is
  unavailable regardless of whether a valid key exists.
- Therefore the **hybrid duration approach is the default design**, with the
  REST path kept as an optional enhancement guarded behind a reachability
  check that degrades gracefully rather than erroring.
