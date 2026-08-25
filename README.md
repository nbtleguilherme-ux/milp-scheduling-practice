# Session 8 — MILP Course Scheduling

Practice session on **Mixed-Integer Linear Programming (MILP)** applied to university course scheduling.  
You will model and solve a scheduling problem from scratch using [Pyomo](http://www.pyomo.org/) and the HiGHS solver.

---

## Objectives

By the end of this session you will be able to:

- Translate a real scheduling problem into decision variables, constraints, and an objective function
- Implement a MILP model in Pyomo (`ConcreteModel`, `Var`, `Constraint`, `Objective`)
- Use a rule-based constraint and a `ConstraintList` for irregular sets
- Solve the model with the HiGHS solver and extract the solution
- Extend a hard-constraint model with a soft preference penalty and analyse the trade-off

---

## Repository structure

```
milp_scheduling_practice.ipynb   ← your working notebook (complete the TODOs)
milp_scheduling_solution.ipynb   ← full reference solution (solution branch only)
eda_teaching_dataset.ipynb       ← exploratory analysis of the dataset
data_teaching/                   ← dataset (see below)
```

> The **`main`** branch contains the practice notebook only.  
> The **`solution`** branch adds the complete reference solution — shared at the end of the session.

---

## Dataset — `data_teaching/`

A small, self-contained scheduling instance: **10 courses**, **5 rooms**, **15 timeslots** (3 days × 5 periods).

| File | Rows | Description |
|------|------|-------------|
| `courses.csv` | 10 | Course id, name, lecturer, number of students, capacity needed, preferred period |
| `rooms.csv` | 5 | Room id, name, capacity, cost per slot |
| `timeslots.csv` | 15 | Timeslot id, day (Mon/Tue/Wed), period (P1–P5), start time, cost |
| `lecturers.csv` | 8 | Lecturer id, name |
| `course_conflicts.csv` | 8 | Pairs of courses that share students (cannot overlap) |
| `course_period_options.csv` | 6 | Allowed periods for restricted courses only |

### Cost structure

**Room costs** (larger room = higher cost):

| Room | Capacity | Cost/slot |
|------|----------|-----------|
| Hall A | 45 | 5 |
| Hall B | 40 | 4 |
| Room C | 30 | 3 |
| Room D | 25 | 2 |
| Room E | 20 | 1 |

**Timeslot costs** (reflects time-of-day preference):

| Period | Start | Cost |
|--------|-------|------|
| P1 | 07:30 | 2 — too early |
| P2 | 08:30 | 1 — ideal |
| P3 | 09:30 | 1 — ideal |
| P4 | 10:30 | 2 — late morning |
| P5 | 13:30 | 3 — afternoon |

---

## Problem summary

Assign each course to exactly one (timeslot, room) pair to **minimise total cost**, subject to:

| Constraint | Rule |
|------------|------|
| C1 — Assignment | Every course is scheduled exactly once |
| C2 — Room conflict | At most one course per (timeslot, room) |
| C3 — Capacity | Room capacity ≥ class size |
| C4 — Student clash | Courses sharing students cannot overlap |
| C5 — Lecturer clash | A lecturer cannot teach two courses at the same time |
| C6 — Period restriction *(optional)* | Some courses may only use certain periods |

**Extension (Part 5):** add a soft penalty for courses scheduled outside their preferred period and tune the penalty weight λ.

---

## Installation

Python 3.9+ is recommended. Install all dependencies with:

```bash
pip install pyomo highspy pandas numpy matplotlib seaborn
```

| Package | Purpose |
|---------|---------|
| `pyomo` | MILP modelling framework |
| `highspy` | HiGHS solver (called via `appsi_highs` inside Pyomo) |
| `pandas` | Data loading and result tables |
| `numpy` | Numerical utilities |
| `matplotlib` | Schedule grid and utilisation charts |
| `seaborn` | Plot styling |

### Verify your setup

Run the first two cells of `milp_scheduling_practice.ipynb`. You should see:

```
Pyomo ready.
Data loaded.
  courses      (10, 6)
  rooms        (5, 4)
  timeslots    (15, 5)
  ...
```

---

## How to work through the notebook

1. Read each `## Part N` heading and the mathematical formulation in the markdown cell above it.
2. Complete every cell marked **`# TODO`** — the hint and Pyomo pattern are included as comments.
3. Run Part 4 to solve and check your result (optimal cost should be **30**).
4. Complete Part 5 (soft constraints) and experiment with different values of λ.
5. Complete Part 6 Step 1 to generate the visualisations.
