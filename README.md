# Railway Reservation System

A small Python project that manages **ticket booking, cancellation and seat availability** for a train, and fixes a real-world problem: **double booking (overbooking)** when many users book the last seats at the same time.

Built as a case study for Chandigarh University (Student: Kavish, UID 25BCD10107, BCA Data Science, Section 25BCD 2B, Guide: Mr. Jitendra Kumar).

## The problem

A simple booking routine reads the free-seat count, checks it is greater than zero, then inserts a ticket. When several users run the check at the same instant, they all pass it and all book. In the simulation (`naive.py`), **10 users got confirmed tickets for only 3 seats**. Cancelled seats were also not given to waiting passengers.

## The solution

| Fix | How it works |
|-----|--------------|
| Atomic booking | The check-and-allocate step runs inside a lock and a database transaction, so a seat can never be sold twice. |
| Derived availability | Available seats are counted from confirmed tickets, not stored in a separate counter that could drift. |
| Waitlist | When a class is full, new passengers get status `WL` (limit 5 per class, then rejected). |
| Auto promotion | Cancelling a confirmed ticket gives the freed seat to the oldest waitlisted passenger (first in, first out). |
| Refund slabs | Refund depends on how close the travel date is. |

## Features

- Seat availability for a train, date and class (total, available, waitlist, fare)
- Booking with automatic seat allocation (`CNF`) or waitlisting (`WL`) and a unique PNR
- Cancellation (`CAN`) with refund calculation
- Automatic waitlist promotion
- Reservation chart listing all active passengers

## Ticket status

| Status | Meaning |
|--------|---------|
| `CNF` | Confirmed, seat number assigned |
| `WL` | Waitlisted, no seat yet |
| `CAN` | Cancelled (final) |

## Refund policy

| Cancellation time | Refund |
|-------------------|--------|
| More than 2 days before travel | 75% of fare |
| 1 to 2 days before travel | 50% of fare |
| Less than 1 day before travel | 25% of fare |
| Waitlisted ticket | 90% of fare |

The demo uses a fixed reference date of **1 November 2026** for "today" (see `cancel()` in `rrs.py`). Change it to `date.today()` for real use.

## Requirements

- Python 3.8 or later
- Standard library only (`sqlite3`, `threading`, `datetime`) for the core program
- `Pillow` (`pip install pillow`) only if you want to regenerate the screenshots with `shot.py`

## Project structure

```
railway-reservation-system/
├── rrs.py          Core system: the RRS class (booking, cancellation, availability, chart)
├── demo.py         Runs the main scenario and saves outputs to s1.txt ... s6.txt
├── naive.py        Reproduces the problem (legacy logic overbooks 3 seats to 10 users)
├── edge.py         Refund slabs and edge cases (writes s6.txt)
├── shot.py         Turns the .txt outputs into terminal-style PNG screenshots
├── screenshots/    Output screenshots used in the report
└── README.md
```

## How to run

```bash
# 1. Reproduce the problem (prints to the terminal)
python3 naive.py

# 2. Run the main demo (writes s1.txt to s6.txt)
python3 demo.py

# 3. Run refund slabs and edge cases (prints, and overwrites s6.txt)
python3 edge.py

# 4. Optional: make PNG screenshots from the .txt files (needs Pillow)
python3 shot.py
```

`demo.py` also overwrites `s6.txt`. Run `edge.py` after it if you want the refund-slab version of that screenshot. Output files are written to the folder you run the scripts from.

## Using the class in your own code

```python
from rrs import RRS

r = RRS()
r.add_train("12001", "Shatabdi Express", "New Delhi", "Chandigarh",
            "2026-11-10", {"CC": (5, 780), "EC": (3, 1450)})   # class: (seats, fare)

print(r.availability("12001", "2026-11-10", "CC"))
pnr = r.book("12001", "2026-11-10", "CC", "Aman", 28)          # returns PNR, or None if full
print(r.get(pnr))                                              # (pnr, name, status, seat, fare)
refund, promoted = r.cancel(pnr)                               # refund amount, promoted WL ticket or None
print(r.chart("12001", "2026-11-10", "CC"))                    # active passengers
```

`book()` returns `None` when both the class and the waitlist are full. `cancel()` returns `None` for an unknown or already cancelled PNR.

## Database tables

| Table | Key columns | Purpose |
|-------|-------------|---------|
| `train` | no (PK), name, src, dst | Train details |
| `inventory` | train_no, travel_date, cls (PK), total, fare | Seats and fare per train, date, class |
| `ticket` | pnr (PK), train_no, travel_date, cls, name, age, status, seat, fare, booked_seq | One row per booking |

The database is in memory (SQLite), so data is lost when the program ends.

## Test results

| Test | Result |
|------|--------|
| Availability search | 5 of 5 seats free |
| 7 bookings on a 5-seat class | 5 `CNF` (seats 1 to 5), 2 `WL` |
| Cancel a confirmed ticket | 75% refund (Rs. 585), first `WL` promoted into the freed seat |
| Refund slabs | 75%, 50%, 25% and 90% (waitlist) all correct |
| Cancel an already cancelled or unknown PNR | Rejected |
| 20 concurrent users, 3 seats | 3 `CNF`, 5 `WL`, 12 rejected, 0 duplicate seats |
| Legacy logic, 10 users, 3 seats | 10 confirmed (defect reproduced) |

## Limitations and future work

- One lock covers all bookings. That is correct on one machine but would be a bottleneck at large scale. Use per train, date and class locks, or row locks such as `SELECT ... FOR UPDATE` in PostgreSQL.
- Single journey segment only, no segment-wise availability.
- No RAC status, berth preference or quotas (Tatkal, Ladies, Senior Citizen).
- No payment, login, notifications or user interface.
