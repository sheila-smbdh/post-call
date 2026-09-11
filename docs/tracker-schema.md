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
2. Stage is **not** `Discovery Call Scheduled`, **not** `No Show/Cancel`,
   and **not** `Reschedule Request` — those leads have not had a call yet.
3. A meeting activity exists with the closer as `user_id`, in the past.
4. The call is confirmed to have really happened — see
   "Did the call happen?" below.

Scope is the **first discovery call only**. Everything measured starts at
that call. Earlier setter/intro-call activity is context, not measurement.

---

## Did the call happen?

The eligibility gate rests on **closer-driven CRM state after the scheduled
time**, not on call duration. Verified against all 5 pilot leads 2026-09-11.

**Signal: the opportunity moves out of `Discovery Call Scheduled`.**

| Moved to | Reading |
|---|---|
| `Next 7 Days`, `In Month Closing`, `Long Term w/ Intention`, `Long Term No Timeline`, `Long Term Farm`, `Split Pay/Deposit`, `Offer Not Presented`, `WON`, `LOST` | Call **happened** |
| `No Show/Cancel`, `Reschedule Request` | Call **did not happen** |
| Still `Discovery Call Scheduled` | Not yet actioned — exclude, do not assume |

Both branches are live: Harry alone has 50 opportunities sitting in
`No Show/Cancel` / `Reschedule Request`, so the gate genuinely discriminates.

### Why this is trustworthy

The move is human, and its timing tracks the real call:

| Lead | Call start (UTC) | Moved | Lag | Actor |
|---|---|---|---|---|
| Will Morgan | Aug 31 21:00 | Sep 1 03:45:15 | +6h45m | **Helen Guo** |
| Ming Su | Sep 3 17:00 | Sep 3 19:41:51 | +2h42m | Harry |
| Anthony Mariani | Sep 4 13:00 | Sep 4 14:10:21 | +1h10m | Harry |
| Ari Brownstein | Sep 9 19:00 | Sep 9 21:52:09 | +2h52m | Harry |
| Kevin Gindi | Sep 10 20:00 | Sep 10 20:54:39 | +54m | Harry |

Anthony is the control: his call ran 12:59:31 → 14:11:22 (71m51s, established
via Zoom integration data). The stage moved at **14:10:21 — 61 seconds before
the call ended**. Harry moved it while still on the call.

**Record the actor and the lag on every row.** Will Morgan's move was made by
Helen Guo at 11:45pm ET, ~7h after the call — not the closer, and not
call-coupled. See trap 5.

### What this changes

Column 8 no longer means "qualified call (>15 min)". It means **the call
occurred**. A short call that the closer moved straight to `LOST` now passes
the gate. If a quality threshold is still wanted, it needs Actual Duration,
which needs the REST API.

---

## Columns (27)

Trimmed from an initial 39 on 2026-09-10. Cuts are listed at the end of this
section with reasons, so nothing is silently dropped.

### A. Identity (4)
| # | Column | Source |
|---|---|---|
| 1 | Lead Name | contact `contact_name` |
| 2 | Close Lead URL | built from `lead_id` |
| 3 | Current Stage | opportunity `status_label` |
| 4 | Setter / Booked By | earlier intro-call meeting `user_id` |

### B. The first discovery call (5)
| # | Column | Source |
|---|---|---|
| 5 | Call Date + Time (ET) | meeting `starts_at` → America/New_York |
| 6 | Scheduled Duration (min) | meeting `duration` / 60 |
| 7 | Actual Duration (min) | zoom `end_time - start_time`, exact. REST API only — blocked |
| 8 | **Call Happened?** | CRM-state gate — see below. Replaces the >15 min rule |
| 9 | Verified Participants | zoom `participants`, de-duplicated by name |

### C. Outcome (3)
| # | Column | Note |
|---|---|---|
| 10 | Stage After Call | |
| 11 | Lag: Call End → Stage Change | |
| 12 | Stage Moved By | may not be the closer |

### D. Post-call follow-up (4)
| # | Column | Note |
|---|---|---|
| 13 | First Post-Call Touch Channel | SMS / email / call |
| 14 | First Post-Call Touch (ET) | |
| 15 | **Lag: Call End → First Touch** | the headline metric |
| 16 | Post-Call Material Sent + What | e.g. "resources + agreement" |

### E. Channel volume (3)
| # | Column | Format |
|---|---|---|
| 17 | SMS Out / In | e.g. "5 / 4" |
| 18 | Email Out / In | e.g. "2 / 0" |
| 19 | Total Exchanges | closer n / lead n |

### F. Responsiveness (3)
| # | Column | Note |
|---|---|---|
| 20 | Closer Median Response Time | |
| 21 | Closer Slowest Response | median + slowest bounds the behavior |
| 22 | Send Hours (ET) | hour-of-day distribution |

### G. Chase / recovery (5)
| # | Column | Note |
|---|---|---|
| 23 | Days Call → Closer Re-initiation | |
| 24 | Unanswered Closer Re-touches | persistence |
| 25 | Follow-up Call Booked + Date (ET) | |
| 26 | Exchanges to Book Follow-up | |
| 27 | Who Went Silent Last | the drop-off point |

### Cut, with reasons

| Cut | Why |
|---|---|
| Closer | Redundant — one tab per closer. Re-add if tabs are ever merged. |
| Extract Date | Belongs in tab metadata, not repeated on every row. |
| Day of Week | Derivable in-sheet: `=TEXT(date,"ddd")`. |
| Transcript Word Count | Was a proxy for duration. Real durations make it redundant. |
| Duration Source | Was an audit trail for a mixed-provenance column. Now always Zoom. |
| Zoom Start / Zoom End | Evidence behind Actual Duration; kept in the raw JSON, not the sheet. |
| Stage Before Call | Nearly always "Discovery Call Scheduled". |
| Post-Call Email (ET) | Covered by First Touch + Material columns. |
| Closer Fastest Response | Median and slowest already bound the distribution. |
| Lead First Reply (ET) | Implied by Total Exchanges and Who Went Silent Last. |
| Follow-up Zoom calls booked | Merged into "Follow-up Call Booked + Date". |
| Phone dials | Near-zero volume in the sample; re-add if dials become a real channel. |

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

7. **Lead status `Call in Progress` is automation, not evidence.** It is set
   by a rule running under Jay DeCristofaro's account and fires at the
   *scheduled* start time whether or not anyone joins. Across all 5 pilot
   leads it landed 13–57s after the booked time:

   | Lead | Scheduled | Fired | Lag |
   |---|---|---|---|
   | Will Morgan | Aug 31 21:00 | 21:00:13 | +13s |
   | Ming Su | Sep 3 17:00 | 17:00:25 | +25s |
   | Anthony Mariani | Sep 4 13:00 | 13:00:19 | +19s |
   | Ari Brownstein | Sep 9 19:00 | 19:00:16 | +16s |
   | Kevin Gindi | Sep 10 20:00 | 20:00:57 | +57s |

   Never use lead status as the did-it-happen gate — every booked call passes,
   no-shows included. Use the **opportunity** status change instead.

8. **Closers do not write call notes in Close.** All 5 pilot leads have zero
   `activity.note` records after their discovery call. Anthony's only note is
   from 2026-08-22 by the *setter* (Jordan Kempster), about the intro call.
   The call record lives in Granola. Do not expect notes to corroborate the
   gate.

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
