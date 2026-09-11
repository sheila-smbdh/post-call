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

Anthony is the control: his call is independently known to have run
12:59:31 → 14:11:22. The stage moved at **14:10:21 — 61 seconds before the
call ended**. Harry moved it while still on the call. (That timing was a
one-off verification of the gate, not a column the tracker carries.)

**Record the actor and the lag on every row.** Will Morgan's move was made by
Helen Guo at 11:45pm ET, ~7h after the call — not the closer, and not
call-coupled. See trap 5.

### Scope of the gate

Column 7 answers **did the call occur**, nothing more. The original
"qualified call (>15 min)" rule is retired: call length is not tracked, so a
short call the closer moved straight to `LOST` passes the same as a long one.

This is deliberate. Post-call analysis measures what the closer did *after*
the call — follow-up lag, channel mix, persistence, recovery — and every one
of those is measurable as soon as the call is known to have happened. Length
was never an input to them.

---

## Columns (25)

Trimmed from an initial 39 on 2026-09-10, then to 25 on 2026-09-11. Cuts are listed at the end of this
section with reasons, so nothing is silently dropped.

### A. Identity (4)
| # | Column | Source |
|---|---|---|
| 1 | Lead Name | contact `contact_name` |
| 2 | Close Lead URL | built from `lead_id` |
| 3 | Current Stage | opportunity `status_label` |
| 4 | Setter / Booked By | earlier intro-call meeting `user_id` |

### B. The first discovery call (3)
| # | Column | Source |
|---|---|---|
| 5 | Call Date + Time (ET) | meeting `starts_at` → America/New_York |
| 6 | Scheduled Duration (min) | meeting `duration` / 60 |
| 7 | **Call Happened?** | CRM-state gate — see below |

### C. Outcome (3)
| # | Column | Note |
|---|---|---|
| 8 | Stage After Call | |
| 9 | Lag: Call End → Stage Change | |
| 10 | Stage Moved By | may not be the closer |

### D. Post-call follow-up (4)
| # | Column | Note |
|---|---|---|
| 11 | First Post-Call Touch Channel | SMS / email / call |
| 12 | First Post-Call Touch (ET) | |
| 13 | **Lag: Call End → First Touch** | the headline metric |
| 14 | Post-Call Material Sent + What | e.g. "resources + agreement" |

### E. Channel volume (3)
| # | Column | Format |
|---|---|---|
| 15 | SMS Out / In | e.g. "5 / 4" |
| 16 | Email Out / In | e.g. "2 / 0" |
| 17 | Total Exchanges | closer n / lead n |

### F. Responsiveness (3)
| # | Column | Note |
|---|---|---|
| 18 | Closer Median Response Time | |
| 19 | Closer Slowest Response | median + slowest bounds the behavior |
| 20 | Send Hours (ET) | hour-of-day distribution |

### G. Chase / recovery (5)
| # | Column | Note |
|---|---|---|
| 21 | Days Call → Closer Re-initiation | |
| 22 | Unanswered Closer Re-touches | persistence |
| 23 | Follow-up Call Booked + Date (ET) | |
| 24 | Exchanges to Book Follow-up | |
| 25 | Who Went Silent Last | the drop-off point |

### Cut, with reasons

| Cut | Why |
|---|---|
| Closer | Redundant — one tab per closer. Re-add if tabs are ever merged. |
| Extract Date | Belongs in tab metadata, not repeated on every row. |
| Day of Week | Derivable in-sheet: `=TEXT(date,"ddd")`. |
| **Actual Duration (min)** | Dropped 2026-09-11. Call length is not needed for post-call analysis — the CRM-state gate establishes that the call happened, which is all the analysis requires. Removed the last REST API dependency. |
| **Verified Participants** | Dropped 2026-09-11. Sourced from Zoom `participants`, which exists only in the REST response. Unobtainable once REST is out, and attendance is already settled by the gate. |
| Transcript Word Count | Was a proxy for duration; duration is no longer tracked at all. |
| Duration Source | Was an audit trail for a column that no longer exists. |
| Zoom Start / Zoom End | Was evidence behind Actual Duration, which is gone. |
| Stage Before Call | Nearly always "Discovery Call Scheduled". |
| Post-Call Email (ET) | Covered by First Touch + Material columns. |
| Closer Fastest Response | Median and slowest already bound the distribution. |
| Lead First Reply (ET) | Implied by Total Exchanges and Who Went Silent Last. |
| Follow-up Zoom calls booked | Merged into "Follow-up Call Booked + Date". |
| Phone dials | Near-zero volume in the sample; re-add if dials become a real channel. |

## Verified CRM traps

These are real, observed in the 5-lead sample. The skill must handle each.

1. **`duration` is the booked length, not the call's.** The meeting object's
   `duration` returns the Calendly booking (2700s = 45m), which is why col 6
   is labelled *Scheduled* Duration. Never present it as how long the call
   ran. Actual length is no longer tracked — see the cuts table.

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

   Confirmed by direct negative test (2026-09-11): five leads Harry marked
   `No Show/Cancel` fired the **same** flag at their scheduled time —

   | Lead | Outcome | Fired |
   |---|---|---|
   | lead_hips… | No Show/Cancel | Sep 10 12:00:03 |
   | Noreen Merchant | No Show/Cancel | Sep 10 18:00:07 |
   | james ogburn | No Show/Cancel | Sep 8 17:00:42 |
   | Dayton D | No Show/Cancel | Sep 5 15:00:04 |
   | Nicole | No Show/Cancel | Sep 5 14:00:07 |

   That is 10 for 10 — five calls that happened, five that did not, flag
   identical for both. It carries **zero** information about attendance; it
   records only that a call was booked.

   Never use lead status as the did-it-happen gate — every booked call passes,
   no-shows included. Use the **opportunity** status change instead.

8. **Calendar notifications pollute inbound email counts.** Calendly and
   Google Calendar mail (`New Event: …`, `Accepted: …`,
   `Tentatively Accepted: …`) lands as `activity.email` with
   `direction: inbound`. In the 5-lead pilot, 17 of 19 inbound emails were
   these; only 2 were genuine lead replies. Filter them out by subject
   prefix before counting, or Email In is inflated ~9x.

9. **"Lag: Call End → X" is not computable.** Call end came from Zoom and
   duration is no longer tracked, so both lag columns measure from the
   **scheduled** end (`starts_at + duration`). On a call that overran this
   reads high: Anthony's first touch shows +25m from scheduled end but
   actually landed 50s *before* the call really finished. Label these
   columns "Sched End →", never "Call End →".

10. **Duplicate lead records exist.** "Will Morgan" returns two leads; only
    one (`lead_BdkUT2…`, status "Call in Progress") is the pilot lead, the
    other is an untouched "Potential". Disambiguate by status and by the
    presence of a meeting with the closer — never take the first hit.

11. **Closers do not write call notes in Close.** All 5 pilot leads have zero
   `activity.note` records after their discovery call. Anthony's only note is
   from 2026-08-22 by the *setter* (Jordan Kempster), about the intro call.
   The call record lives in Granola. Do not expect notes to corroborate the
   gate.
