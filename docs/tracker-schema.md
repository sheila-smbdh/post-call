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

1. **`duration` is scheduled, not actual.** Close's meeting object exposes
   exactly one duration field and it returns the booked Calendly length
   (2700s = 45m) for every call, including the one that actually ran 1h11m.
   There is no `ends_at`, no recordings field.

2. **`user_id` on inbound messages is the closer, not the sender.** Filtering
   by `user_id` mislabels every lead reply as closer-authored. Always split
   with the `direction` filter (`inbound` / `outbound`).

3. **Ghost duplicate meetings.** Anthony had a second Sep 4 meeting
   ("Anthony Mariani and Ezra Isla", 10:00am ET) that the setter told him to
   cancel; it was never marked cancelled. Naive counting sees two discovery
   calls. De-duplicate by closer `user_id` + proximity.

4. **Calendly email subject times are not ET.** Anthony's follow-up subject
   reads "11:00 Mon, 14 Sep 2026" while the meeting is 16:00 UTC = 12:00pm ET
   and the closer's own SMS says "Monday 12 pm EST". Never parse times from
   email subjects — use meeting `starts_at` converted to America/New_York.

5. **Stage changes are not always the closer's.** Will Morgan's move was made
   by Helen Guo. Record the actor.

6. **Granola coverage is partial.** 3 of 5 sample calls had transcripts
   (Anthony, Kevin Gindi, Ari Brownstein); Ming Su and Will Morgan had none.
   Join is by lead name + ET datetime — there is no shared ID.

---

## Open question — actual call duration

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
