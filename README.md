# Tech & AI Concentration — Seat Allocation Optimizer

An interactive tool for modeling capacity constraints, seat allocation, and breaking points across MBA cohorts in the Tech & AI concentration at HEC Paris.

## The Problem

The Tech & AI concentration serves three cohorts with different enrollment timelines and course access:

| Cohort | Timeline | Accessible Periods |
|--------|----------|-------------------|
| **S26 16mo** | Starts Sep 2026, graduates Dec 2027 | Dec 26, Jan-Mar 27, Feb 27, Apr-Jun 27, Apr 27, Oct 27, Dec 27 |
| **J27 12mo** | Starts Jan 2027, graduates Dec 2027 | Feb 27, Apr 27, Oct 27, Dec 27 |
| **J27 16mo** | Starts Jan 2027, graduates Apr 2028 | Apr 27, Oct 27, Dec 27, Jan-Mar 28, Feb 28, Apr 28 |

Enrollment is **sequential**: S26 enrolls first, then J27 16mo, then J27 12mo. Later cohorts get whatever capacity remains.

**The structural tension**: J27 12mo has only 4 intensive periods (no elective access), limiting them to 6 of 9 courses. They enroll last and compete for seats with larger cohorts.

## How the Math Works

### Core Concepts

**Effective Capacity** of a slot = raw seat capacity minus seats already consumed by previous MBA cohorts (non-concentration students):

```
effCap(slot) = max(0, slot.cap - slot.prev)
```

**Course Popularity** scales effective capacity to model uneven demand:

```
effCapPref(slot) = floor(effCap(slot) * coursePop[slot.course] / 100)
```

**Quota** limits how many seats a cohort can use in a slot. `null` means unlimited access; `0` means blocked:

```
getQuota(slot, cohort) =
  if quota is null: effCap(slot)
  else: min(quota, effCap(slot))
```

### The Solver (Greedy with Binary Search)

For each cohort in enrollment order (S26 -> J27 16mo -> J27 12mo):

1. **Build course map**: For each course, collect all accessible slots (quota > 0) and their remaining capacity after previous cohorts consumed seats.

2. **Compute per-course capacity**: Sum remaining capacity across all slots for that course, minus any reserved seats (for non-J12 cohorts).

3. **Binary search for max students**: Find the largest number `N` where the total available seat-enrollments across all courses can serve `N` students at their average course load:

```
canServeLoad(N, courses, caps, avgLoad):
  totalCap = sum of min(cap[course], N) for each course
  return totalCap >= ceil(avgLoad * N)
```

The `min(cap, N)` reflects that no more than N students can use any single course.

4. **Water-fill allocation**: Distribute `ceil(avgLoad * N)` seat-enrollments across courses, filling the lowest-allocated course first (like pouring water into connected vessels). This ensures even distribution.

5. **Consume capacity**: Deduct allocated seats from remaining slot capacity before the next cohort enrolls.

### Fairness Mode (Equalize)

Instead of maximizing total students served, find the highest completion percentage `P%` achievable by **all** cohorts simultaneously:

1. Binary search over `P` from 100% down to 0%.
2. For each `P`, set each cohort's floor = `round(cohortSize * P / 100)`.
3. Run the solver with each cohort **capped at its floor** (not at maximum). This prevents early cohorts from consuming capacity that later cohorts need.
4. If all floors are feasible, `P` is achievable.

This reveals the equity cost: total served drops, but no cohort is left behind.

## The 9 Courses

| Course | Format | Cap | Notes |
|--------|--------|-----|-------|
| User-centric AI Product Design (UCAI) | Intensive/Elective | 50 | |
| Product Innovation through Design Thinking (PIDT) | Intensive/Elective | 36 | 30 as elective |
| AI Fundamentals (AIFU) | Intensive | 50 | |
| Managing AI Responsibly (MAIR) | Intensive/Elective | 50 | |
| Tech Sprints (TESP) | Elective only | 50 | |
| Data Driven Decision Making (DDDM) | Elective only | 50 | Cannot be intensive |
| AI for Business Decision Making (AIBDM) | Elective/Intensive | 35 | |
| Supply Chain Transformation (SCT) | Intensive | 50 | |
| Generative AI for Management (GAIM) | Intensive | 40 | |

**Course accessibility by cohort:**
- **S26**: 9 of 9 courses (full access)
- **J27 16mo**: 8 of 9 (misses AIBDM — only in Apr-Jun elective)
- **J27 12mo**: 6 of 9 (misses TESP, DDDM, AIBDM — no elective periods)

## Breaking Points

### Under Current Setup (70 S26, 12 J12, 22 J16 = 104 total)

All 104 students can complete. The system works — but with little margin.

### Stress Factor: J26 Flood

Previous cohort (J26 16mo) floods shared first-year courses, consuming 34-45 seats per slot in Dec 26, 40 seats in Jan-Mar 27 and Feb 27. This dramatically reduces effective capacity for concentration students.

### Stress Factor: All J27 12mo Join

All 29 J27 12mo students take the concentration (vs. default 12). They only have 4 intensive periods with ~6 courses accessible. At 29 students needing ~3.7 courses each = ~107 seat-enrollments from just ~6 slots with shared capacity.

### Stress Factor: High Uptake

Concentration becomes popular: S26 rises from 70 to 110, J27 12mo from 12 to 25, J27 16mo from 22 to 45. Total demand jumps from 104 to 180 students.

### Perfect Storm (All 3 Combined)

184 students compete for limited seats. Results:

| Cohort | Served | Rate |
|--------|--------|------|
| S26 | ~78/110 | 71% |
| J27 12mo | **0/29** | **0%** |
| J27 16mo | 45/45 | 100% |
| **Total** | **~123/184** | **67%** |

**J27 12mo is completely shut out.** S26 and J27 16mo consume all shared intensive capacity before J27 12mo can enroll.

## Solutions

### Schedule Fixes

Move courses to periods that improve J27 12mo access:

| Fix | Change | Effect |
|-----|--------|--------|
| MAIR to Oct intensive | Jan-Mar elective -> Oct intensive (both years) | Opens slot in J12's period |
| AIBDM to Oct intensive | Apr-Jun elective -> Oct intensive (both years) | J12 gains AIBDM: 6 -> **7 unique courses** |
| UCAI to Jan-Mar elective | Oct intensive -> Jan-Mar elective (both years) | S26 gets elective option |
| AIFU add Feb intensive | Add Feb offering alongside Dec (both years) | More AIFU capacity for J12 |

**Key insight**: The AIBDM fix alone gives J27 12mo access to 7 of 7 intensive-capable courses. The only 2 they can never reach (TESP, DDDM) are elective-only.

### Governance Mechanisms

| Mechanism | Effect | Impact under Perfect Storm |
|-----------|--------|---------------------------|
| **Mandatory Course** | 1 required course, avg load drops by 1 | Reduces total seat demand by ~29 enrollments |
| **Reserve 12 seats/course** | Holds seats in J12-accessible slots | J12 jumps from 0 to **24/29 (83%)** |
| **Cohort Caps** (S26: 90, J12: 25) | Limits concentration enrollment | Reduces total demand from 184 to 160 |
| **Qualify via 3 Intensives** | Qualification bar drops from ~3.7 to 3 courses | Total served rises from 125 to 142 |

### Combined Solutions under Perfect Storm

| Scenario | Total Served | S26 | J12 | J16 |
|----------|-------------|-----|-----|-----|
| No governance | 123/184 (67%) | 78 | 0 | 45 |
| + Qualify via 3 Intensives | 142/184 (77%) | 97 | 0 | 45 |
| + 3 Int. + Reserve 12 | 142/184 (77%) | 73 | **24** | 45 |
| + All governance | 142/160 (89%) | 73 | **24** | 45 |

### The Structural Argument

The root cause is **format asymmetry**: J27 12mo has zero elective periods. Their entire concentration must come from intensives. If the concentration is redefined so that **qualification requires 3 intensive courses** (with electives as optional extras), then:

- Every cohort can qualify through intensive slots alone
- 7 of 9 courses can be offered as intensive
- The 2 elective-only courses (TESP, DDDM) become enrichment, not barriers
- Seat reservation ensures J12 actually gets intensive capacity

This reframing — intensives as core, electives as extras — resolves the structural inequity without reducing course breadth.
