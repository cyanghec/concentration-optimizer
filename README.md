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
| $\alpha_c \in [0, 1]$ | Quota percentage for cohort $c$ (governance lever; $\sum_c \alpha_c = 1$) |

### Derived Quantities

**Effective capacity** (seats available to concentration students):

$$\overline{\text{Cap}}_s = \max\!\Big(0,\; \text{Cap}_s - \text{Prev}_s\Big)$$

**Popularity-adjusted capacity**:

$$\widetilde{\text{Cap}}_s = \Big\lfloor \overline{\text{Cap}}_s \cdot \text{Pop}_k \Big\rfloor \quad \text{where } s = (k, p)$$

**Cohort quota** (maximum seats cohort $c$ can claim in slot $s$):

$$\bar{Q}_{s,c} = \begin{cases} 0 & \text{if } Q_{s,c} = 0 \text{ (cohort blocked from slot)} \\ \min\!\Big(Q_{s,c},\; \lfloor \widetilde{\text{Cap}}_s \cdot \alpha_c \rfloor\Big) & \text{if quota governance is active} \\ \min\!\Big(Q_{s,c},\; \widetilde{\text{Cap}}_s\Big) & \text{otherwise (no quota limit)} \end{cases}$$

Intuition: if a slot has 50 effective seats and $\alpha = (0.50, 0.25, 0.25)$, then S26 can claim up to 25, J27\_16mo up to 12, and J27\_12mo up to 12 — regardless of enrollment order. Blocked cohorts (quota = 0) remain blocked.

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

### Sequential Enrollment Extension

The pure LP above assumes simultaneous optimization. In practice, enrollment is **sequential** — S26 enrolls first, then J27\_16mo, then J27\_12mo. Each later cohort sees only the residual capacity.

This is modeled as a **sequence of LPs**, one per cohort in enrollment order $\sigma = (c_1, c_2, c_3)$:

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
| **Cohort Quotas** (S26: 50%, J16: 25%, J12: 25%) | Each cohort gets a guaranteed share of each slot's capacity | J12 jumps from 0 to **15/29 (52%)** — order-independent |
| **Cohort Caps** (S26: 90, J12: 25) | Limits concentration enrollment | Reduces total demand from 184 to 160 |
| **Qualify via 3 Intensives** | Qualification bar drops from ~3.7 to 3 courses | Total served rises from 125 to 142 |

### Combined Solutions under Perfect Storm

| Scenario | Total Served | S26 | J12 | J16 |
|----------|-------------|-----|-----|-----|
| No governance | 123/184 (67%) | 78 | 0 | 45 |
| + Quotas (50/25/25) | 87/184 (47%) | 39 | **15** | 33 |
| + Quotas + 3 Intensives | — | — | — | — |
| + All governance | — | — | — | — |

*Note: Exact numbers depend on quota percentages, which are tunable in the tool.*

### The Structural Argument

The root cause is **format asymmetry**: J27 12mo has zero elective periods. Their entire concentration must come from intensives. If the concentration is redefined so that **qualification requires 3 intensive courses** (with electives as optional extras), then:

- Every cohort can qualify through intensive slots alone
- 7 of 9 courses can be offered as intensive
- The 2 elective-only courses (TESP, DDDM) become enrichment, not barriers
- Cohort quotas ensure J12 actually gets intensive capacity

This reframing — intensives as core, electives as extras — resolves the structural inequity without reducing course breadth.
