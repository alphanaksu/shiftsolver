# ShiftSolver — Specification (v0.1)

Status: design only, no code yet. Every later module (`requirements.py`, `model.py`, `validate.py`,
`baseline.py`, `app.py`) is built from this document. If code and spec disagree, fix one of them
on purpose, never silently.

---

## 1. Purpose and scope

**Goal.** Given hourly demand and staff availability, produce the cheapest one-week roster that
meets coverage and labor rules, in two modes:

- **Café mode**: demand is customers per hour.
- **Call-center mode**: demand is calls per hour, converted to agents with Erlang C.

**In scope (v0.1)**
- One week (7 days × 24 hours), 1-hour time slots.
- Shifts chosen from a fixed list of templates (overnight shifts allowed).
- Soft coverage and soft supervisor rule (penalties), hard labor rules.
- A single interchangeable role, plus a "supervisor" flag.

**Out of scope (v0.1)**
- Multiple weeks, multiple skills or roles with separate demand, shift preferences, fairness
  terms, overtime premiums, unpaid breaks, holidays, real personal data.

---

## 2. Notation

| Symbol | Meaning |
|---|---|
| `E` | set of employees |
| `Sup ⊆ E` | employees flagged as supervisors |
| `D = {0..6}` | days, 0 = Monday |
| `T = {0..167}` | hour slots of the week, `t = 24·d + h` (h = 0..23) |
| `S` | set of shift templates |
| `start_s`, `len_s` | start hour (0..23) and length in hours of template `s` |
| `I = D × S` | shift instances; instance `i = (d, s)` starts at `a_i = 24·d + start_s` and ends at `b_i = a_i + len_s` |
| `cover(i)` | slots covered by `i`: `{ (a_i + k) mod 168 : k = 0..len_s−1 }` |
| `need_t` | staff required in slot `t` (section 4) |
| `w_e` | hourly wage of employee `e`, in integer cents |
| `R` | minimum rest between shifts, hours (default 11) |
| `OFF` | minimum days off per week (default 2) |
| `P_cov`, `P_sup` | penalties in cents (default 100 000 = 1000 € each) |

**Time wraps.** Slot indices are taken mod 168, so a Sunday 22:00–06:00 shift covers Monday
00:00–06:00 of the same week. This assumes the week repeats (a standard cyclic roster).

---

## 3. Inputs

Each dataset is a folder `data/<name>/` with five files. All data is synthetic.

### 3.1 `settings.json`

| Key | Mode | Meaning | Default |
|---|---|---|---|
| `mode` | both | `"cafe"` or `"callcenter"` | required |
| `open_start`, `open_end` | cafe | opening hours, e.g. 7 and 19 (closed hours need 0) | 0, 24 |
| `customers_per_staff_hour` | cafe | productivity, `p` | 20 |
| `min_staff_open` | cafe | minimum staff while open, `m` | 2 |
| `aht_sec` | callcenter | average handle time | 300 |
| `sl_target` | callcenter | service level target | 0.80 |
| `sl_seconds` | callcenter | answer-within threshold `T_sl` | 20 |
| `max_occupancy` | callcenter | cap on agent busy fraction | 0.85 |
| `shrinkage` | callcenter | fraction of paid time not on the phones | 0.30 |
| `min_rest_hours` | both | `R` | 11 |
| `min_days_off` | both | `OFF` | 2 |
| `penalty_short_cents` | both | `P_cov` | 100000 |
| `penalty_sup_cents` | both | `P_sup` | 100000 |
| `time_limit_sec` | both | solver time limit | 30 |

(`json` is in Python's standard library, so this adds no dependency.)

### 3.2 `demand.csv`

| Column | Type | Rule |
|---|---|---|
| `day` | int 0..6 | 0 = Monday |
| `hour` | int 0..23 | |
| `volume` | number ≥ 0 | café: customers in that hour; call center: calls in that hour |

Exactly 168 rows (one per slot). Hours outside café opening hours must have `volume = 0`.

### 3.3 `staff.csv`

| Column | Type | Rule |
|---|---|---|
| `employee_id` | str | unique, e.g. `S01` (synthetic ids, no real names) |
| `wage_cents` | int > 0 | cents per paid hour |
| `min_hours` | int ≥ 0 | weekly minimum |
| `max_hours` | int ≥ `min_hours` | weekly maximum |
| `is_supervisor` | 0/1 | 1 = counts toward the supervisor rule |

### 3.4 `availability.csv`

| Column | Type | Rule |
|---|---|---|
| `employee_id` | str | must exist in `staff.csv` |
| `day` | int 0..6 | |
| `earliest_start` | int 0..23 | |
| `latest_end` | int 1..48 | values above 24 mean spill into the next day (30 = 06:00 next day) |

A day with **no row means unavailable** that day.

### 3.5 `shifts.csv`

| Column | Type | Rule |
|---|---|---|
| `shift_id` | str | unique, e.g. `M0715` |
| `start_hour` | int 0..23 | |
| `length_hours` | int 1..12 | paid hours (breaks are not modeled) |

### 3.6 Input validation (before solving)

Fail with a readable message if: demand has ≠ 168 rows or duplicates; any id in availability is
unknown; `min_hours > max_hours`; a template runs longer than 12 h; café demand is positive
while closed. **Warn** (do not fail) if total `max_hours` of all staff is below total `need_t`
(guaranteed understaffing) or if no supervisor exists.

---

## 4. Staff requirement per slot (`requirements.py`)

Output: integer vector `need_t`, `t ∈ T`.

### 4.1 Café mode

In words: when the café is open, we need enough people for the customers, and never fewer than
the minimum crew. When closed, we need nobody.

```
need_t = 0                                   if slot t is outside [open_start, open_end)
need_t = max( m , ceil( volume_t / p ) )     otherwise
```

### 4.2 Call-center mode (Erlang C)

In words: calls arrive randomly and agents are shared; we look for the smallest number of agents
`N` such that enough calls are answered within `T_sl` seconds and agents are not overloaded, then
add extra people because agents are not on the phone all the time (breaks, training: shrinkage).

For a slot with `λ` calls per hour and handle time `AHT` seconds:

```
A        = λ · AHT / 3600                         # offered load in Erlangs
P_wait   = ErlangC(N, A)
         =  [ A^N / N! · N/(N−A) ]  /  [ Σ_{k=0}^{N−1} A^k / k!  +  A^N / N! · N/(N−A) ]     (needs N > A)
SL(N)    = 1 − P_wait · exp( −(N − A) · T_sl / AHT )
occ(N)   = A / N

N*       = smallest integer N > A  with  SL(N) ≥ sl_target  AND  occ(N) ≤ max_occupancy
need_t   = ceil( N* / (1 − shrinkage) )           (and need_t = 0 if λ = 0)
```

**Worked example (also the first unit test).** λ = 120 calls/h, AHT = 300 s, `T_sl` = 20 s:

- A = 120 · 300 / 3600 = **10 Erlangs**
- N = 11: SL = 0.362 · N = 12: SL = 0.607 · N = 13: SL = 0.766 · **N = 14: P_wait = 0.1741,
  SL = 1 − 0.1741 · e^(−4·20/300) = 1 − 0.1741 · 0.7659 = 0.867 ≥ 0.80**, occupancy 10/14 = 0.714 ≤ 0.85 ✔
- N\* = 14, so `need = ceil(14 / 0.70) = 20`.

Other checkpoints (from the same formulas): λ = 10 → 5, λ = 30 → 8, λ = 60 → 12, λ = 156 → 25.

---

## 5. Decision variables

| Variable | Domain | Meaning |
|---|---|---|
| `x[e,i]` | {0,1} | employee `e` works shift instance `i = (d,s)` |
| `short[t]` | integer ≥ 0 | missing staff in slot `t` |
| `supshort[t]` | {0,1} | slot `t` has `need_t > 0` but no supervisor on duty |

Derived: `worked[e,d] = Σ_s x[e,(d,s)]` (0 or 1 by C4).

---

## 6. Constraints

Each constraint has: the rule in words, the math, the validator check that re-verifies it
independently of the solver (CLAUDE.md rule: every constraint gets a validator check and a test).

### C1 — Coverage (soft)
**Words.** In every hour, people on shift plus the shortfall must reach the requirement. Being
short is allowed but penalised in the objective; being over-staffed is allowed and only costs wages.
```
Σ_{e∈E} Σ_{i : t ∈ cover(i)} x[e,i]  +  short[t]  ≥  need_t        ∀ t ∈ T
```
**Validator V1.** Recount staff per slot from the roster; report `short_t = max(0, need_t − staffed_t)`.
**Test.** Tiny instance with one worker and need 2 in one hour → `short = 1`.

### C2 — Supervisor present (soft)
**Words.** Every hour that needs staff must have at least one supervisor on shift; otherwise pay a penalty.
```
Σ_{e∈Sup} Σ_{i : t ∈ cover(i)} x[e,i]  +  supshort[t]  ≥  1        ∀ t with need_t > 0
```
Supervisors also count toward the headcount in C1.
**Validator V2.** Per slot with `need_t > 0`, count supervisors on shift.
**Test.** Only non-supervisors available → `supshort = 1` for every open hour.

### C3 — Availability (hard)
**Words.** A person can work a shift only if the whole shift fits inside that day's availability window.
```
x[e,(d,s)] = 0   unless  earliest_start[e,d] ≤ start_s   and   start_s + len_s ≤ latest_end[e,d]
```
(No row for `(e,d)` = never.) The window applies to the day the shift **starts**.
**Validator V3.** Each rostered shift must lie inside its window.
**Test.** Employee unavailable Monday → no Monday shift for them.

### C4 — At most one shift per day (hard)
```
Σ_{s∈S} x[e,(d,s)]  ≤  1        ∀ e ∈ E, d ∈ D
```
**Validator V4.** No employee has two shifts starting the same day.
**Test.** One worker, two overlapping templates, need 2 → solver may not use both.

### C5 — Minimum rest between shifts (hard)
**Words.** After a shift ends, the same person cannot start another until `R` hours have passed.
Computed on the wrapped week, so Sunday night → Monday morning is covered. Two instances `i ≠ j`
**conflict** if one starts before the other has finished plus the rest time:
```
conflict(i,j)  ⇔  (a_j − a_i) mod 168  <  len_i + R     OR     (a_i − a_j) mod 168  <  len_j + R

x[e,i] + x[e,j]  ≤  1        ∀ e ∈ E, ∀ conflicting pairs (i,j)
```
This also forbids overlapping shifts. Example with R = 11: a 14–22 shift on Monday conflicts with a 06–14
shift on Tuesday (gap 8 h) but not with a 14–22 shift on Tuesday (gap 16 h).
**Validator V5.** For each employee, sort shifts by start; compute every end→next-start gap (including
last shift → first shift of next week, +168); each must be ≥ R.
**Test.** Evening shift followed by morning shift is rejected; evening followed by evening is allowed.

### C6 — Weekly hours (hard)
```
min_hours_e  ≤  Σ_{i=(d,s)} len_s · x[e,i]  ≤  max_hours_e        ∀ e ∈ E
```
Hard on purpose: contracts are not optional. If availability makes `min_hours` impossible the model
is infeasible and the app must say which rule blocks it (see section 11, open question 1).
**Validator V6.** Sum paid hours per employee and compare with the bounds.
**Test.** `max_hours = 8` → at most one 8-hour shift.

### C7 — Minimum days off (hard)
```
Σ_{d∈D} worked[e,d]  ≤  7 − OFF        ∀ e ∈ E
```
**Validator V7.** Count working days (by shift start day) per employee.
**Test.** With OFF = 2, nobody works 6 days even if demand exists.

### Integrity checks (validator V0)
Each assignment refers to a known employee and template; no duplicate `(e,d,s)` rows.

---

## 7. Objective

Minimise total cost in integer cents (CP-SAT needs integers):

```
min   Σ_{e∈E} Σ_{i=(d,s)} w_e · len_s · x[e,i]          (wages)
    + P_cov · Σ_{t∈T} short[t]                          (understaffing penalty)
    + P_sup · Σ_{t∈T} supshort[t]                       (missing-supervisor penalty)
```

Because the default penalties (1000 €) are far above any hourly wage (≈ 15–25 €), the solver first
removes shortfalls and only then saves wages. The weights are settings, so the owner can explore the
trade-off ("what does one hour of missing coverage cost me?").

---

## 8. Outputs

1. **Roster table**: `employee_id, day, shift_id, start, end, paid_hours`.
2. **Hourly coverage table**: `t, need, scheduled, short, supervisors_on_duty`.
3. **KPIs**: total cost, wage cost, penalty cost, total short staff-hours, % of required hours fully
   covered, hours per employee, solver status (`OPTIMAL / FEASIBLE / INFEASIBLE`), solve time.
4. **Validator report**: pass/fail per check V0–V7, with violating rows listed.
5. **Baseline comparison**: greedy cost vs solver cost and the % saving.
6. **Plotly heatmap**: need vs scheduled per day × hour; shortfalls highlighted.
7. **Excel export** (openpyxl): roster, coverage, KPIs on separate sheets.

**Solver contract.** `model.py` takes the parsed inputs and `need_t`, solves with CP-SAT
(time limit from settings, fixed random seed for reproducibility) and returns the roster plus status.
It never returns a roster that `validate.py` rejects; the tests assert this on every dataset.

---

## 9. Baseline heuristic (`baseline.py`)

A greedy algorithm for comparison, deliberately simple:

1. Walk through slots in time order; find the first slot with `short > 0`.
2. Among (employee, template) options covering that slot that pass C3–C7, pick the cheapest per
   newly covered needed hour; assign it.
3. Repeat until no feasible assignment reduces the shortfall.

The baseline may be worse or leave more shortfall; it is checked by the same validator.

---

## 10. Synthetic sample datasets (to be generated later, fixed seed, no real data)

Hand estimates below are for sanity only; exact numbers come from the code and tests.

### 10.1 `cafe_small`
- Mode `cafe`, open 07–19 daily, `p = 20`, `m = 2`, rest 11, OFF 2.
- Templates: `07–15` (8h), `11–19` (8h), `07–12` (5h), `14–19` (5h).
- Demand (customers/h). Weekdays: 07–09 → 30, 09–12 → 35, 12–14 → 60, 14–17 → 30, 17–19 → 25.
  Weekends: 07–10 → 30, 10–12 → 55, 12–15 → 80, 15–19 → 40. Closed hours 0.
  → need ≈ 26 staff-hours per weekday, 32 per weekend day, ≈ 194 per week.
- Staff (8): `S01–S03` supervisors (wage 2000, 24–40 h), `S04–S05` full-time (1500, 24–40 h),
  `S06–S08` part-time (1300–1400, 8–20 h). Maximum capacity ≈ 260 h. 3 supervisors are needed because
  the open hours require ≈ 14 supervisor shifts a week and each person can work at most 5.
- Availability: mostly full days 07–19; `S06` only after 14:00 on weekdays (student).
- **Expectation:** full coverage, zero penalty. Tests solver optimum ≤ baseline.

### 10.2 `callcenter`
- Mode `callcenter`, open 24/7, AHT 300 s, SL 80/20, occupancy ≤ 85 %, shrinkage 30 %.
- Templates: `06–14`, `14–22`, `22–06`, `09–17`, `10–18` (8h each), `10–14`, `17–21` (4h each).
- Demand (calls/h). Weekdays: 00–06 → 10, 06–09 → 30, 09–17 → 60, 17–22 → 30, 22–24 → 10.
  Weekends: 00–08 → 10, 08–20 → 20, 20–24 → 10. **Monday ×1.3 for 06–22** (the spike).
  → need ≈ 200 staff-hours per weekday, ≈ 130 per weekend day, ≈ 1300 per week.
- Staff (40): `C01–C06` team leads (supervisors, wage 2400, 32–40 h), `A01–A24` full-time agents
  (1700, 32–40 h), `A25–A34` part-time agents (1600, 16–24 h). Maximum capacity 1440 h.
  Six leads are needed because 24/7 supervision is ≈ 21 eight-hour shifts and each lead works ≤ 5.
- Availability: mostly full (window 0–48); part-timers unavailable weekdays before 16:00.
- **Expectation:** tight but feasible; night shifts exercise the wrap-around rest rule.

### 10.3 `cafe_stress` (understaffed café, tests the soft path)
- `cafe_small` with demand × 1.25 (the minimum crew `m = 2` stays) and only 6 staff: remove `S03`
  (a supervisor) and `S08`; `S02` is unavailable on Saturday and Sunday.
  → need ≈ 237 staff-hours vs capacity ≈ 200; only 2 supervisors (≤ 10 shifts) for 14 needed.
- **Expectation:** `short > 0`, `supshort > 0`, positive penalty cost, solver still returns a roster that
  passes the validator, with shortfalls concentrated in the lunch peaks.

---

## 11. Assumptions and open questions

**Assumptions (chosen by me, easy to change)**
- Overnight shifts allowed; the week is cyclic (Sunday night wraps to Monday).
- Shift length equals paid hours; breaks are not modeled.
- A "work day" is the day a shift starts; availability applies to the start day only, so a night
  shift's spill-over past midnight is not blocked by next-day unavailability.
- Supervisors count toward headcount.
- Money in integer cents; default penalty 1000 € per missing staff-hour and per missing-supervisor hour.
- Max-consecutive-days rule left out: with ≥ 2 days off in a 7-day week it is implied.

**Open questions for you**
1. Should `min_hours` be hard (current) or soft? Hard can make a whole dataset infeasible when
   availability is tight.
2. Should there be an upper bound on staff per hour (a café with 8 people at a quiet hour is unrealistic)?
3. Is a missing supervisor as bad as a missing agent (same 1000 € penalty), or less?
4. Should `cafe_small` and `callcenter` use the same rest rule (11 h), or should cafés allow shorter rest?
