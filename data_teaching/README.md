# Course Scheduling — Teaching Dataset

A small, self-contained instance designed for a first MILP course-scheduling practice.

## Problem statement

Assign each course exactly **one timeslot** and **one room** so that all hard constraints
are satisfied and the total scheduling cost is minimised.

---

## Files

| File | Rows | Key columns |
|------|------|-------------|
| `courses.csv` | 10 | `course_id, name, lecturer_id, n_students, capacity_needed, preferred_period` |
| `rooms.csv` | 5 | `room_id, name, capacity, cost_per_slot` |
| `timeslots.csv` | 15 | `timeslot_id, day, period, start_time, cost` (3 days × 5 periods) |
| `lecturers.csv` | 8 | `lecturer_id, name` |
| `course_conflicts.csv` | 8 | `course_a, course_b, shared_students` |
| `course_period_options.csv` | 6 | `course_id, period` (allowed periods for restricted courses only) |

---

## Sets & parameters

| Symbol | Meaning | Source |
|--------|---------|--------|
| **C** | set of courses | `courses.csv` |
| **R** | set of rooms | `rooms.csv` |
| **T** | set of timeslots | `timeslots.csv` |
| `cap[r]` | capacity of room *r* | `rooms.capacity` |
| `size[c]` | students needing a seat | `courses.capacity_needed` |
| `lec[c]` | lecturer of course *c* | `courses.lecturer_id` |
| `rc[r]` | room cost per slot | `rooms.cost_per_slot` |
| `tc[t]` | timeslot cost | `timeslots.cost` |
| `E` | set of conflicting course pairs (student overlap) | `course_conflicts.csv` |
| `A[c]` | allowed timeslot IDs for course *c* (unrestricted if absent) | `course_period_options.csv` |

---

## Decision variable

```
x[c, t, r] ∈ {0, 1}     for all c ∈ C, t ∈ T, r ∈ R

x[c,t,r] = 1  iff course c is scheduled in timeslot t and room r
```

With |C|=10, |T|=15, |R|=5 this gives **750 binary variables** — small enough to
inspect by hand and solve in seconds with any MILP solver.

---

## Hard constraints

### C1 — Each course is assigned exactly once
```
∑_{t,r} x[c,t,r] = 1     ∀ c ∈ C
```

### C2 — No two courses share the same (timeslot, room)
```
∑_c x[c,t,r] ≤ 1     ∀ t ∈ T, r ∈ R
```

### C3 — Room capacity must not be exceeded
```
x[c,t,r] = 0     if cap[r] < size[c]
```
*(Equivalently: add this as a constraint or pre-filter feasible (c,r) pairs.)*

### C4 — Student conflicts: two courses sharing students cannot overlap
```
∑_r x[c1,t,r] + ∑_r x[c2,t,r] ≤ 1     ∀ (c1,c2) ∈ E, ∀ t ∈ T
```

### C5 — Lecturer conflict: a lecturer teaches at most one course per timeslot
```
∑_r x[c1,t,r] + ∑_r x[c2,t,r] ≤ 1     ∀ t ∈ T,
    ∀ c1 ≠ c2 such that lec[c1] = lec[c2]
```

*Lecturers with two courses in this dataset:*
- **Prof. Nguyen (id=1)**: Math Analysis (1) + Statistics (9)
- **Prof. Le (id=3)**: Programming 101 (3) + Databases (6)

### C6 — Period restrictions (optional hard constraint)
Some courses may only use certain periods.
`course_period_options.csv` lists the **allowed periods** for restricted courses:

| Course | Allowed periods | Rationale |
|--------|----------------|-----------|
| Databases (6) | P3, P4, P5 | Computer lab only available from 09:30 |
| ML Basics (10) | P1, P2, P3 | Morning lecture slot only |

Courses **not** listed in that file can use **any** period.

```
∑_r x[c,t,r] = 0     ∀ c with restrictions, ∀ t whose period ∉ A[c]
```

---

## Objective function

Minimise the total scheduling cost, where each assignment pays a **room cost** and
a **timeslot cost**:

```
Minimise  ∑_{c,t,r}  (rc[r] + tc[t]) × x[c,t,r]
```

### Cost structure

**Room costs** (`rc[r]`) reflect size — larger rooms cost more to open:

| Room | Capacity | Cost/slot |
|------|----------|-----------|
| Hall A | 45 | 5 |
| Hall B | 40 | 4 |
| Room C | 30 | 3 |
| Room D | 25 | 2 |
| Room E | 20 | 1 |

**Timeslot costs** (`tc[t]`) reflect time-of-day preference:

| Period | Start | Cost | Interpretation |
|--------|-------|------|----------------|
| P1 | 07:30 | 2 | Too early |
| P2 | 08:30 | 1 | Ideal |
| P3 | 09:30 | 1 | Ideal |
| P4 | 10:30 | 2 | Late morning |
| P5 | 13:30 | 3 | Afternoon (heat / fatigue) |

The cost structure creates a **natural tension**: a large course (many students) *must*
use an expensive room (C3), while the optimizer tries to avoid wasting extra capacity
and avoids high-cost time slots wherever constraints allow.

---

## Extension exercise — soft preference penalty

`courses.preferred_period` records each course's ideal period.  
Add a soft penalty to the objective:

```
Minimise  ∑_{c,t,r} (rc[r] + tc[t]) × x[c,t,r]
        + λ × ∑_{c,t,r} penalty[c,t] × x[c,t,r]

where  penalty[c,t] = 1  if period(t) ≠ preferred_period[c]
                    = 0  otherwise
```

Experiment with different values of λ to see how much the solver deviates from the
hard-constraint optimal to respect preferences.

---

## Conflict graph (quick reference)

```
Student conflicts (C4):
  1 — 2   (15 students)
  1 — 9   (10 students)
  2 — 5   (12 students)
  3 — 4   (18 students)
  3 — 6   (14 students)   ← also lecturer conflict (Prof. Le)
  4 — 10  ( 8 students)
  5 — 10  (20 students)
  7 — 8   (16 students)

Lecturer conflicts (C5, not already in C4):
  1 — 9   (Prof. Nguyen)
```

---

## Feasibility notes

- 10 courses, 15 timeslots, 5 rooms → 75 possible (timeslot, room) slots, far more than needed.
- The binding constraints are capacity (courses 1, 3, 10 all need Hall A or B) and
  the lecturer / student conflict graph.
- The dataset is **guaranteed feasible** with all six constraints active.
