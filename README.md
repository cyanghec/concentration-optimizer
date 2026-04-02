# Tech & AI Concentration — Seat Allocation Optimizer

An interactive tool for modeling capacity constraints, seat allocation, and breaking points across MBA cohorts in the Tech & AI concentration at HEC Paris.

## The Problem

The Tech & AI concentration serves three cohorts with different enrollment timelines and course access:

| Cohort | Timeline | Accessible Periods |
|--------|----------|-------------------|
| **S26 16mo** | Starts Sep 2026, graduates Dec 2027 | Dec 26, Jan-Mar 27, Feb 27, Apr-Jun 27, Apr 27, Oct 27, Dec 27 |
| **J27 12mo** | Starts Jan 2027, graduates Dec 2027 | Apr 27, Oct 27, Dec 27 |
| **J27 16mo** | Starts Jan 2027, graduates Apr 2028 | Apr 27, Oct 27, Dec 27, Jan-Mar 28, Feb 28, Apr 28 |

**S26 enrolls first**, then J27 16mo and J27 12mo enroll **simultaneously**. The solver models this as S26 first (greedy), then J16/J12 sequentially as an approximation of their simultaneous enrollment.

**The structural tension**: J27 12mo has only 3 intensive periods (Apr 27, Oct 27, Dec 27) and 4 accessible courses. They compete for seats with larger cohorts.

## How the Math Works

### Sets & Indices

| Symbol | Definition |
|--------|-----------|
| $\mathcal{C}$ | Set of cohorts: {S26, J27\_16mo, J27\_12mo} |
| $\mathcal{K}$ | Set of courses: {UCAI, PIDT, AIFU, MAIR, TESP, DDDM, AIBDM, SCT, GAIM} |
| $\mathcal{P}$ | Set of periods: {Dec26, Jan-Mar27, Feb27, Apr-Jun27, Apr27, Oct27, Dec27, Jan-Mar28, Feb28, Apr28} |
| $\mathcal{S}$ | Set of slots: $\mathcal{S} \subseteq \mathcal{K} \times \mathcal{P}$, each slot $s = (k, p)$ is a course $k$ offered in period $p$ |
| $\mathcal{S}_c \subseteq \mathcal{S}$ | Slots accessible to cohort $c$ (determined by timeline and format) |
| $\mathcal{K}_c \subseteq \mathcal{K}$ | Courses accessible to cohort $c$: $\mathcal{K}_c = \{k : \exists\, p \text{ s.t. } (k,p) \in \mathcal{S}_c\}$ |

### Parameters

| Symbol | Definition |
|--------|-----------|
| $\text{Cap}_s$ | Raw seat capacity of slot $s$ |
| $\text{Prev}_s$ | Seats already consumed by non-concentration students (prior MBA cohorts) |
| $\text{Pop}_k \in [0, 1]$ | Popularity factor for course $k$ (scales effective capacity) |
| $Q_{s,c} \in \mathbb{Z}_{\geq 0} \cup \{\infty\}$ | Quota: max seats cohort $c$ may use in slot $s$. $Q_{s,c} = 0$ if $s \notin \mathcal{S}_c$ |
| $N_c$ | Size of cohort $c$ (students in concentration) |
| $L_c$ | Average course load for cohort $c$ (courses needed to qualify) |
| $R \geq 0$ | Seats reserved per course for J27\_12mo (governance lever, default 0) |
| $\alpha_c \in [0, 1]$ | Quota percentage for cohort $c$ (governance lever; $\sum_c \alpha_c = 1$) |

### Derived Quantities

**Effective capacity** (seats available to concentration students):

$$\overline{\text{Cap}}_s = \max\!\Big(0,\; \text{Cap}_s - \text{Prev}_s\Big)$$

**Popularity-adjusted capacity**:

$$\widetilde{\text{Cap}}_s = \Big\lfloor \overline{\text{Cap}}_s \cdot \text{Pop}_k \Big\rfloor \quad \text{where } s = (k, p)$$

**Cohort quota** (maximum seats cohort $c$ can claim in slot $s$):

$$\bar{Q}_{s,c} = \begin{cases} 0 & \text{if } Q_{s,c} = 0 \text{ (cohort blocked from slot)} \\ \min\!\Big(Q_{s,c},\; \lfloor \widetilde{\text{Cap}}_s \cdot \alpha_c \rfloor\Big) & \text{if quota governance is active} \\ \min\!\Big(Q_{s,c},\; \widetilde{\text{Cap}}_s\Big) & \text{otherwise (no quota limit)} \end{cases}$$

Intuition (quotas): if a slot has 50 effective seats and $\alpha = (0.50, 0.25, 0.25)$, then S26 can claim up to 25, J27\_16mo up to 12, and J27\_12mo up to 12 — regardless of enrollment order. Blocked cohorts (quota = 0) remain blocked.

**Seat reservation** (alternative to quotas): Instead of splitting capacity, subtract $R$ seats from non-J12 cohorts in J12-accessible slots:

$$\bar{Q}_{s,c}^{\text{res}} = \begin{cases} \min\!\Big(Q_{s,c},\; \widetilde{\text{Cap}}_s\Big) & \text{if } c = \text{J27\_12mo} \\ \min\!\Big(Q_{s,c},\; \widetilde{\text{Cap}}_s - R\Big) & \text{if } s \in \mathcal{S}_{\text{J27\_12mo}} \\ \min\!\Big(Q_{s,c},\; \widetilde{\text{Cap}}_s\Big) & \text{otherwise} \end{cases}$$

Intuition (reservation): if a slot has 50 effective seats and $R = 12$, then S26 and J27\_16mo can each use up to 38, while J27\_12mo sees the full 50. Combined, non-J12 cohorts can never take more than 38 — guaranteeing at least 12 remain for J12.

**Reservation vs Quotas**: Reservation is more efficient because it only constrains cohorts that compete with J12 and doesn't waste capacity on cohorts with abundant exclusive slots. Under Perfect Storm: reservation achieves 140/184 (76%) vs quotas at 110/184 (60%).

### Decision Variables

| Variable | Domain | Meaning |
|----------|--------|---------|
| $x_{s,c} \in \mathbb{Z}_{\geq 0}$ | for each $s \in \mathcal{S}_c,\; c \in \mathcal{C}$ | Seats allocated to cohort $c$ in slot $s$ |
| $n_c \in \mathbb{Z}_{\geq 0}$ | for each $c \in \mathcal{C}$ | Number of students served in cohort $c$ |

### Objective

**Maximize total students served:**

$$\max \sum_{c \in \mathcal{C}} n_c$$

### Constraints

**1. Capacity**: Total allocation across all cohorts in a slot cannot exceed adjusted capacity:

$$\sum_{c \in \mathcal{C}} x_{s,c} \;\leq\; \widetilde{\text{Cap}}_s \qquad \forall\; s \in \mathcal{S}$$

**2. Quota**: Each cohort respects its per-slot quota:

$$x_{s,c} \;\leq\; \bar{Q}_{s,c} \qquad \forall\; s \in \mathcal{S},\; c \in \mathcal{C}$$

**3. Cohort size**: Cannot serve more students than are in the cohort:

$$n_c \;\leq\; N_c \qquad \forall\; c \in \mathcal{C}$$

**4. Course load**: Each cohort's total seat-enrollments must meet the average load requirement:

$$\sum_{s \in \mathcal{S}_c} x_{s,c} \;\geq\; \lceil L_c \cdot n_c \rceil \qquad \forall\; c \in \mathcal{C}$$

**5. Per-course cap**: No more students in a course than are served (a student takes a course at most once):

$$\sum_{\substack{s = (k,p) \in \mathcal{S}_c}} x_{s,c} \;\leq\; n_c \qquad \forall\; c \in \mathcal{C},\; k \in \mathcal{K}_c$$

Note: Cohort quotas are handled within $\bar{Q}_{s,c}$ — no separate constraint needed. The quota percentages $\alpha_c$ are tunable in the tool (default: S26 50%, J16 25%, J12 25%).

### Greedy Sequential Solver

S26 enrolls first, then J27\_16mo and J27\_12mo enroll simultaneously. The solver approximates this as a **greedy sequential heuristic** — processing cohorts in order $\sigma = (c_1, c_2, c_3)$ = (S26, J27\_16mo, J27\_12mo), where each cohort maximizes its own allocation given remaining capacity. The J16/J12 sequential ordering is a simplification of their simultaneous enrollment.

This is modeled as a **sequence of sub-problems**, one per cohort:

**Stage $t$** (for cohort $c_t$): Solve

$$\max \; n_{c_t}$$

subject to constraints (1)–(5) for cohort $c_t$ only, with capacity reduced by prior allocations:

$$\widetilde{\text{Cap}}_s^{(t)} = \widetilde{\text{Cap}}_s - \sum_{\tau < t} x_{s, c_\tau}^* \qquad \forall\; s \in \mathcal{S}$$

where $x_{s, c_\tau}^*$ are the fixed allocations from stages $\tau < t$.

**Implementation note**: The tool solves each stage via binary search over $n_{c_t}$ with water-fill allocation (greedy LP relaxation), rather than a general LP solver.

### Fairness Mode (Equalize)

Instead of maximizing total served, find the highest **completion percentage** $P^*$ achievable by all cohorts simultaneously:

$$P^* = \max \; P$$

$$\text{subject to:} \quad n_c \;\geq\; \Big\lfloor \tfrac{P}{100} \cdot N_c \Big\rfloor \qquad \forall\; c \in \mathcal{C}$$

plus all constraints (1)–(5).

Equivalently: each cohort is **capped at its fair-share floor** $\lfloor \frac{P}{100} \cdot N_c \rfloor$, preventing early cohorts from consuming capacity that later cohorts need.

The tool finds $P^*$ via binary search: for each candidate $P$, it runs the sequential solver with each cohort capped at its floor and checks feasibility.

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
| Supply Chain Transformation (SCT) | Intensive | 50 | Not in default schedule; available as fix |
| Generative AI for Management (GAIM) | Intensive | 40 | |

**Course accessibility by cohort** (default schedule, without SCT):
- **S26**: 8 of 8 courses (full access including Oct 27 in year 2)
- **J27 16mo**: 7 of 8 (misses AIBDM — only in Apr-Jun elective)
- **J27 12mo**: 4 of 8 (only intensives in Apr 27, Oct 27, Dec 27 — no elective access)

With SCT added: 9 courses total. S26: 9/9, J16: 8/9, J12: 4/9.

## Previous Consumption (Carry-over from S25/J26)

Seats already consumed by non-concentration students in year 1 slots (per admin data, Apr 2026):

| Course | Period | Format | Prev Seats | Notes |
|--------|--------|--------|-----------|-------|
| UCAI | Oct 26 | Intensive | 24 | |
| PIDT | Dec 26 | Intensive | 30 | |
| AIFU | Dec 26 | Intensive | 15 | |
| MAIR | Jan-Mar 27 | Elective | 38 | 50 total last year; assuming 12 reserved for S26 |
| TESP | Jan-Mar 27 | Elective | 15 | No confirmed data; estimate |
| DDDM | Jan-Mar 27 | Elective | 35 | |
| MAIR | Feb 27 | Intensive | 12 | Corrected: 12 from prev intensive, not 50 |

Year 2 slots (Oct 27, Dec 27, etc.) have no previous consumption — they are fresh offerings.

## Stress Scenarios

| Scenario | S26 | J12 | J16 | Total | Key Risk |
|----------|-----|-----|-----|-------|----------|
| **Current setup** | 70 | 12 | 22 | 104 | J12 gets 0/12 without governance |
| **J26 Flood** | 70 | 12 | 22 | 104 | Previous cohort consumes year-1 slots |
| **All J12 join** | 70 | 29 | 22 | 121 | 29 students competing for 4 courses |
| **High uptake** | 110 | 25 | 45 | 180 | Demand exceeds capacity |
| **Everyone joins** | 150 | 29 | 54 | 233 | Maximum possible demand |

Stress factors stack (except "Everyone joins" which supersedes "All J12 join" and "High uptake").

## Solutions

### Schedule Fixes

Move courses to periods that improve J27 12mo access:

| Fix | Change | Effect |
|-----|--------|--------|
| MAIR to Oct intensive | Jan-Mar elective → Oct intensive (both years) | Opens slot in J12's period |
| AIBDM to Oct intensive | Apr-Jun elective → Oct intensive (both years) | J12 gains AIBDM access |
| UCAI to Jan-Mar elective | Oct intensive → Jan-Mar elective (both years) | S26 gets elective option |
| SCT add Apr intensive | Add SCT to Apr intensive (both years) | Extra course in shared period |

### Governance Mechanisms

| Mechanism | Effect | Impact under Perfect Storm |
|-----------|--------|---------------------------|
| **Reserve Seats** (R=12 per course) | Subtracts R from S26/J16 capacity in J12-accessible slots | J12 jumps from 0 to served; see sensitivity analysis |
| **Cohort Quotas** (S26: 50%, J16: 25%, J12: 25%) | Each cohort gets a guaranteed share of each shared slot | J12 gains access — but total drops (wastes unused quota) |
| **Cohort Caps** (S26: 90, J12: 25) | Limits concentration enrollment | Reduces total demand |
| **Qualify via 2 Courses** | Qualification bar drops from ~3.7 to 2 courses | Under current setup: 100% accommodation (104/104) |

**Reservation sensitivity**: The tool includes a sensitivity analysis sweeping R from 0 to 25, showing the per-seat tradeoff between J12 access and S26 impact. Reservation is not zero-sum in students — each reserved seat costs S26 ~1 student but gives J12 ~1-2 students, because J12 has fewer courses and uses capacity more efficiently.

### Combined Solutions

**Reservation vs Quotas**: Reservation is strictly more efficient — it only constrains cohorts competing with J12 in shared slots, without wasting capacity on J16 who has abundant exclusive late slots and never uses shared slot quota.

### Qualify via 2 Courses — Scenario Summary

Lowering the qualification bar from ~3.7 to 2 courses is the single most impactful governance lever. Below shows the minimum configuration needed to serve everyone under each scenario.

| Scenario | Qualify via 2 | + Reserve | + Schedule Fixes | Result |
|----------|:---:|:---:|:---:|--------|
| **Current setup** (104 students) | ✓ | — | — | **104/104 (100%)** |
| **High uptake** (180 students) | ✓ | — | — | 166/180 (J12: 11/25) |
| **High uptake** | ✓ | R=12 | — | **180/180 (100%)** |
| **Everyone joins** (233 students) | ✓ | — | — | 204/233 (J12: 0/29) |
| **Everyone joins** | ✓ | R=12 | — | 228/233 (J12: 24/29) |
| **Everyone joins** | ✓ | R=15 | — | **233/233 (100%)** |
| **Everyone joins** | ✓ | R=12 | All fixes | **233/233 (100%)** |
| **Everyone + J26 Flood** | ✓ | — | — | 173/233 (J12: 0/29) |
| **Everyone + J26 Flood** | ✓ | R=12 | All fixes | 210/233 (J12: 29/29, S26: 127/150) |

**Key takeaways:**
- **Current setup**: "Qualify via 2" alone achieves 100%. No other levers needed.
- **High uptake**: "Qualify via 2" + Reserve 12 seats = 100%.
- **Everyone joins**: "Qualify via 2" + Reserve 15 seats = 100%, or Reserve 12 + all schedule fixes = 100%.
- **Everyone + J26 Flood** (worst case): Even with all levers, 100% is not achievable. S26 drops to 127/150. J12 can be fully served with reserve + fixes.

### The Structural Argument

The root cause is **format asymmetry**: J27 12mo has zero elective periods and only 3 accessible intensive periods. Their entire concentration must come from intensives. The default qualification bar (~3.7 courses avg) is nearly impossible for J12 with only 4 accessible courses.

Two key governance levers:
1. **Qualify via 2 Courses** — lowers the bar so J12's 4 accessible courses are sufficient. Achieves 100% under current setup.
2. **Reserve Seats** — guarantees J12 capacity in shared slots. Use the sensitivity analysis to find the optimal R that balances J12 access vs S26 impact.

### J16 is Not the Problem

Analysis shows J27 16mo naturally fills from **exclusive elective and late intensive slots** (Jan-Mar 28, Feb 28, Apr 28) and does not compete with J12 in shared intensive periods. Under all stress scenarios tested, J16 uses zero shared intensive slots — their 4 courses come entirely from exclusive periods.

**The real bottleneck is S26**, who consumes intensive capacity in Apr 27, Oct 27, and Dec 27 before J12 can enroll. Governance levers (reservation, caps) should target S26's intensive consumption, not J16's behavior.

Note: SCT (Supply Chain Transformation) is not in the default course schedule but can be added via the "SCT: + add Apr intensive" toggle.
