# Tech & AI Concentration — Seat Allocation Optimizer

An interactive tool for modeling capacity constraints, seat allocation, and breaking points across MBA cohorts in the Tech & AI concentration at HEC Paris.

---

## 1. The Problem

The Tech & AI concentration serves three cohorts with different enrollment timelines and course access:

| Cohort | Timeline | Accessible Periods | # Periods |
|--------|----------|-------------------|:---------:|
| **S26 16mo** | Sep 2026 → Dec 2027 | Dec 26, Jan-Mar 27, Feb 27, Apr-Jun 27, Apr 27, Oct 27, Dec 27 | 7 |
| **J27 12mo** | Jan 2027 → Dec 2027 | Apr 27, Oct 27, Dec 27 | 3 |
| **J27 16mo** | Jan 2027 → Apr 2028 | Apr 27, Oct 27, Dec 27, Jan-Mar 28, Feb 28, Apr 28 | 6 |

> [!IMPORTANT]
> **S26 enrolls first**, then J16 and J12 enroll **simultaneously**.
> J27 12mo has only 3 intensive periods and 4 accessible courses — they compete for seats with larger cohorts.

### 1.1 The 9 Courses

| Course | Abbr | Format | Cap |
|--------|------|--------|:---:|
| User-centric AI Product Design | UCAI | Intensive / Elective | 50 |
| Product Innovation through Design Thinking | PIDT | Intensive / Elective | 36 |
| AI Fundamentals | AIFU | Intensive | 50 |
| Managing AI Responsibly | MAIR | Intensive / Elective | 50 |
| Tech Sprints | TESP | Elective only | 50 |
| Data Driven Decision Making | DDDM | Elective only | 50 |
| AI for Business Decision Making | AIBDM | Elective / Intensive | 35 |
| Supply Chain Transformation | SCT | Intensive | 50 |
| Generative AI for Management | GAIM | Intensive | 40 |

> [!NOTE]
> SCT is not in the default schedule — available via the "SCT: + add Apr intensive" toggle.

**Course accessibility by cohort** (default, without SCT):

| Cohort | Courses | Access |
|--------|:-------:|--------|
| **S26** | 8 / 8 | Full access (including Oct 27 in year 2) |
| **J27 16mo** | 7 / 8 | Misses AIBDM (only in Apr-Jun elective) |
| **J27 12mo** | 4 / 8 | Only intensives in Apr 27, Oct 27, Dec 27 |

With SCT added → S26: 9/9, J16: 8/9, J12: 5/9.

### 1.2 Previous Consumption (Carry-over from S25/J26)

Seats already consumed by non-concentration students in year 1 slots (per admin data, Apr 2026):

| Course | Period | Format | Prev Seats | Notes |
|--------|--------|--------|:---------:|-------|
| UCAI | Oct 26 | Intensive | 24 | |
| PIDT | Dec 26 | Intensive | 30 | |
| AIFU | Dec 26 | Intensive | 15 | |
| MAIR | Jan-Mar 27 | Elective | 38 | 50 total last year; assuming 12 reserved for S26 |
| TESP | Jan-Mar 27 | Elective | 15 | No confirmed data; estimate |
| DDDM | Jan-Mar 27 | Elective | 35 | |
| MAIR | Feb 27 | Intensive | 12 | Corrected: 12 from prev intensive, not 50 |

Year 2 slots (Oct 27, Dec 27, etc.) have no previous consumption — they are fresh offerings.

---

## 2. Stress Scenarios

| Scenario | S26 | J12 | J16 | Total | Key Risk |
|----------|:---:|:---:|:---:|:-----:|----------|
| **Current setup** | 70 | 12 | 22 | 104 | J12 gets 3/12 without governance |
| **J26 Flood** | 70 | 12 | 22 | 104 | Previous cohort consumes year-1 slots |
| **All J12 join** | 70 | 29 | 22 | 121 | 29 students competing for 4 courses |
| **High uptake** | 110 | 25 | 45 | 180 | Demand exceeds capacity |
| **Everyone joins** | 150 | 29 | 54 | 233 | Maximum possible demand |

> [!NOTE]
> Stress factors stack, except "Everyone joins" which supersedes "All J12 join" and "High uptake".

---

## 3. Solutions

### 3.1 Schedule Fixes

Move courses to periods that improve J27 12mo access:

| Fix | Change | Effect |
|-----|--------|--------|
| **MAIR → Oct** | Jan-Mar elective → Oct intensive (both years) | Opens slot in J12's period |
| **AIBDM → Oct** | Apr-Jun elective → Oct intensive (both years) | J12 gains AIBDM access |
| **UCAI → Jan-Mar** | Oct intensive → Jan-Mar elective (both years) | S26 gets elective option |
| **SCT + Apr** | Add SCT to Apr intensive (both years) | Extra course in shared period |

> [!WARNING]
> **UCAI → Jan-Mar hurts J12** — it removes UCAI from Oct 27, which J12 needs. Avoid using it alone.

### 3.2 Governance Mechanisms

| Mechanism | Effect | Notes |
|-----------|--------|-------|
| **Reserve Seats** (default R=12) | Subtracts R from S26/J16 capacity in J12-accessible slots | See sensitivity analysis in tool |
| **Cohort Quotas** (50/25/25%) | Each cohort gets a guaranteed share of each shared slot | Less efficient than reservation |
| **Cohort Caps** (S26: 90, J12: 25) | Limits concentration enrollment | Reduces total demand |
| **Qualify via 2 Courses** | Qualification bar drops from 3 to 2 courses | Most impactful single lever |

> [!TIP]
> **Reservation vs Quotas**: Reservation is strictly more efficient — it only constrains cohorts competing with J12 in shared slots, without wasting capacity on J16 who fills from exclusive late slots.

---

## 4. Scenario Summary

> [!IMPORTANT]
> Lowering the qualification bar from 3 to 2 courses ("Qualify via 2") is the **single most impactful governance lever**.

| Scenario | Qualify via 2 | + Reserve | + Schedule Fix | Result |
|----------|:---:|:---:|:---:|--------|
| **Current setup** (104) | — | — | — | 95/104 (J12: 3/12) |
| **Current setup** | ✓ | — | — | **104/104 (100%)** |
| **High uptake** (180) | ✓ | — | — | **180/180 (100%)** |
| **Everyone joins** (233) | ✓ | — | — | 210/233 (J12: 6/29) |
| **Everyone joins** | ✓ | R=6 | — | 222/233 (J12: 18/29, S26: 150) |
| **Everyone joins** | ✓ | R=12 | — | 230/233 (J12: 26/29, S26: 150) |
| **Everyone joins** | ✓ | R=15 | — | **233/233 (100%)** |
| **Everyone joins** | ✓ | R=12 | Any one of MAIR/AIBDM/SCT | **233/233 (100%)** |
| **Everyone + Flood** | ✓ | — | — | 173/233 (J12: 0, S26: 119) |
| **Everyone + Flood** | ✓ | R=6 | — | 173/233 (J12: 12, S26: 107) |
| **Everyone + Flood** | ✓ | R=12 | — | 173/233 (J12: 24, S26: 95) |
| **Everyone + Flood** | ✓ | R=12 | SCT | 198/233 (J12: 29, S26: 115) |
| **Everyone + Flood** | ✓ | R=12 | All fixes | 210/233 (J12: 29, S26: 127) |

**Key takeaways:**

- **Current setup** → "Qualify via 2" alone = 100%. No other levers needed.
- **High uptake** → "Qualify via 2" alone = 100%.
- **Everyone joins** → R never costs S26 anything (stays at 150). R=15 or R=12 + one fix = 100%.
- **Everyone + Flood** → Real tradeoff: each +2R gives J12 ~5 students but costs S26 ~5. Best balance: R=12 + SCT (J12: 29/29, S26: 115/150).

---

## 5. Key Insights

### The Structural Argument

The root cause is **format asymmetry**: J27 12mo has zero elective periods and only 3 accessible intensive periods. Their entire concentration must come from intensives. The default qualification bar (3 courses minimum) leaves J12 with almost no margin — they have only 4 accessible courses and must fill 3.

Two key governance levers:
1. **Qualify via 2 Courses** — lowers the bar so J12's 4 accessible courses are sufficient. Achieves 100% under current setup.
2. **Reserve Seats** — guarantees J12 capacity in shared slots. Use the sensitivity analysis to find the optimal R.

### J16 is Not the Problem

> [!NOTE]
> J27 16mo naturally fills from **exclusive late slots** (Jan-Mar 28, Feb 28, Apr 28) and uses zero shared intensive capacity. The real bottleneck is **S26**, who consumes intensive capacity in Apr 27, Oct 27, and Dec 27 before J12 can enroll.

Governance levers (reservation, caps) should target S26's intensive consumption, not J16's behavior.

---

<details>
<summary><h2>6. How the Math Works (click to expand)</h2></summary>

### Sets & Indices

| Symbol | Definition |
|--------|-----------|
| **C** | Set of cohorts: {S26, J27 16mo, J27 12mo} |
| **K** | Set of courses: {UCAI, PIDT, AIFU, MAIR, TESP, DDDM, AIBDM, SCT, GAIM} |
| **P** | Set of periods: {Dec26, Jan-Mar27, Feb27, Apr-Jun27, Apr27, Oct27, Dec27, Jan-Mar28, Feb28, Apr28} |
| **S** | Set of slots: S ⊆ K × P, each slot s = (k, p) is a course k offered in period p |
| **S(c)** ⊆ S | Slots accessible to cohort c (determined by timeline and format) |
| **K(c)** ⊆ K | Courses accessible to cohort c: K(c) = {k : ∃ p such that (k,p) ∈ S(c)} |

### Parameters

| Symbol | Definition |
|--------|-----------|
| **Cap(s)** | Raw seat capacity of slot s |
| **Prev(s)** | Seats already consumed by non-concentration students (prior MBA cohorts) |
| **Pop(k)** ∈ [0, 1] | Popularity factor for course k (scales effective capacity) |
| **Q(s,c)** ≥ 0 or ∞ | Quota: max seats cohort c may use in slot s. Q(s,c) = 0 if s ∉ S(c) |
| **N(c)** | Size of cohort c (students in concentration) |
| **L(c)** | Average course load for cohort c (courses needed to qualify) |
| **R** ≥ 0 | Seats reserved per course for J27 12mo (governance lever, default 0) |
| **α(c)** ∈ [0, 1] | Quota percentage for cohort c (governance lever; Σ α(c) = 1) |

### Derived Quantities

**Effective capacity** (seats available to concentration students):

> **EffCap(s) = max(0, Cap(s) − Prev(s))**

**Popularity-adjusted capacity**:

> **AdjCap(s) = floor(EffCap(s) · Pop(k))** where s = (k, p)

**Cohort quota** (maximum seats cohort c can claim in slot s):

| Condition | Quota limit |
|-----------|-------------|
| Cohort blocked from slot | 0 |
| Quota governance active | min(quota, floor(effective capacity × cohort share)) |
| Otherwise | min(quota, effective capacity) |

**Seat reservation** (alternative to quotas):

| Condition | Reservation-adjusted limit |
|-----------|---------------------------|
| J27 12mo | min(quota, effective capacity) — full access |
| Other cohort, J12-accessible slot | min(quota, effective capacity − R) — reduced |
| Non-J12 slot | min(quota, effective capacity) — unaffected |

### Decision Variables

| Variable | Domain | Meaning |
|----------|--------|---------|
| **x(s,c)** ≥ 0, integer | for each slot s accessible to cohort c | Seats allocated to cohort c in slot s |
| **n(c)** ≥ 0, integer | for each cohort c | Number of students served in cohort c |

### Objective

**Maximize total students served:**

> **Maximize Σ n(c)** over all cohorts c

### Constraints

**1. Capacity**: Total allocation across all cohorts in a slot cannot exceed adjusted capacity:

> For each slot s: **Σ x(s,c)** over all cohorts c **≤ AdjCap(s)**

**2. Quota**: Each cohort respects its per-slot quota:

> For each slot s, cohort c: **x(s,c) ≤ Q(s,c)**

**3. Cohort size**: Cannot serve more students than are in the cohort:

> For each cohort c: **n(c) ≤ N(c)**

**4. Course load**: Each cohort's total seat-enrollments must meet the average load requirement:

> For each cohort c: **Σ x(s,c)** over slots s accessible to c **≥ ceil(L(c) · n(c))**

**5. Per-course cap**: No more students in a course than are served (a student takes a course at most once):

> For each cohort c and course k: **Σ x(s,c)** over slots s = (k, p) accessible to c **≤ n(c)**

### Two-Phase Greedy Solver

**Phase 1 — S26 (greedy):** Solve for S26 alone, maximizing n(S26) subject to constraints (1)–(5), consuming capacity from each slot.

**Phase 2 — J16 + J12 (simultaneous):** With remaining capacity after S26, try both orderings (J16→J12 and J12→J16). Each ordering solves cohorts sequentially with reduced capacity:

> **Cap'(s, t) = Cap(s) − Σ x(s, cτ)*** for all prior cohorts τ < t

Keep whichever ordering produces the higher total n(J16) + n(J12).

**Implementation note**: Each cohort is solved via binary search over n(c) with water-fill allocation (greedy), rather than a general LP solver.

### Fairness Mode (Equalize)

Instead of maximizing total served, find the highest **completion percentage P*** achievable by all cohorts simultaneously:

> **Maximize P** subject to: n(c) ≥ floor(P / 100 · N(c)) for all cohorts c

plus all constraints (1)–(5).

Equivalently: each cohort is **capped at its fair-share floor** floor(P / 100 · N(c)), preventing early cohorts from consuming capacity that later cohorts need.

The tool finds P\* via binary search: for each candidate P, it runs the two-phase solver with each cohort capped at its floor and checks feasibility.

</details>
