# Math & Algorithms Deep-Dive: Resource Leveling Heuristics & RCPSP Walkthrough

Welcome to the definitive mathematical and algorithmic reference for **Resource Leveling & Resource-Constrained Project Scheduling (RCPSP)** in Primavera P6 and modern project scheduling engines.

When scheduling projects in Primavera P6, the **Critical Path Method (CPM)** assumes **infinite resources**. CPM calculates Early Start ($ES$), Early Finish ($EF$), Late Start ($LS$), and Late Finish ($LF$) assuming every crane, pile driver, commissioning engineer, and inverter module is instantly available whenever an activity wants to execute.

In reality, project resources are strictly finite. When concurrent activities demand more labor, equipment, or power than the available limit, a **resource conflict (over-allocation)** occurs. Resolving these over-allocations without or with minimal project delay is the job of **Resource Leveling**.

---

## 1. The Core Problem: Why Resource Leveling is NP-Hard

The fundamental computer science problem behind P6's resource leveler is the **Resource-Constrained Project Scheduling Problem (RCPSP)**.

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        CPM vs RCPSP COMPARISON                         │
├────────────────────────────────┬───────────────────────────────────────┤
│ Critical Path Method (CPM)     │ Resource-Constrained Project          │
│                                │ Scheduling Problem (RCPSP)            │
├────────────────────────────────┼───────────────────────────────────────┤
│ Complexity: O(V + E)           │ Complexity: NP-Hard (Strongly)        │
│ Solved via: Topological Sort   │ Solved via: Priority Rule Heuristics, │
│             & Dynamic Prog.    │             Branch & Bound, or ILP    │
│ Assumptions: Infinite Resource │ Assumptions: Finite Renewable Limits  │
│ Criticality: Zero Total Float  │ Criticality: Critical Chain &         │
│                                │             Resource Dependences      │
└────────────────────────────────┴───────────────────────────────────────┘
```

Because RCPSP is NP-hard, finding the absolute minimum project duration for 50,000 activities subject to multi-resource constraints would require centuries of compute time using brute force. Primavera P6 therefore utilizes **Priority-Rule Based Constructive Heuristics** (Serial and Parallel Schedule Generation Schemes) combined with user-defined priority rules.

---

## 2. Mathematical Formulation of RCPSP

Let a project be represented as an activity-on-node directed acyclic graph $G = (V, E)$, where:
- $V = \{0, 1, 2, \dots, n, n+1\}$ is the set of activities. Activity $0$ is the dummy project start milestone, and $n+1$ is the dummy project finish milestone.
- $E$ is the set of precedence relationships $(i, j) \in E$ meaning activity $i$ must complete before activity $j$ can start ($FS = 0$).
- Each activity $j \in V$ has a non-negative deterministic processing duration $d_j \ge 0$, with $d_0 = d_{n+1} = 0$.
- $R$ is the set of renewable resource types ($k \in R$).
- Each resource type $k \in R$ has a constant capacity limit $C_k$ per time period $t$.
- Each activity $j \in V$ requires $r_{jk}$ units of resource $k$ per time unit throughout its active duration $d_j$.

### Integer Linear Programming (ILP) Formulation

Let binary decision variable $x_{jt} \in \{0, 1\}$ be defined as:
$$x_{jt} = \begin{cases} 1 & \text{if activity } j \text{ finishes at time } t \\ 0 & \text{otherwise} \end{cases}$$

Let $T$ be the scheduling horizon ($T = \sum_{j=1}^{n} d_j$).

#### 1. Objective Function: Minimize Project Makespan (Finish Time of Milestone $n+1$)
$$\min \sum_{t=1}^{T} t \cdot x_{n+1, t}$$

#### 2. Unique Completion Constraint
Every activity must complete exactly once in the schedule horizon:
$$\sum_{t=1}^{T} x_{jt} = 1, \quad \forall j \in V$$

#### 3. Precedence Constraints
For every relationship $(i, j) \in E$, activity $j$ cannot start until activity $i$ has finished:
$$\sum_{t=1}^{T} t \cdot x_{jt} - d_j \ge \sum_{t=1}^{T} t \cdot x_{it}, \quad \forall (i, j) \in E$$

#### 4. Renewable Resource Capacity Constraints
At any time period $t \in \{1, \dots, T\}$, the total demand across all concurrently active tasks cannot exceed resource capacity $C_k$:
$$\sum_{j \in V} r_{jk} \left( \sum_{s=t}^{t + d_j - 1} x_{j, s} \right) \le C_k, \quad \forall k \in R, \; \forall t \in \{1, \dots, T\}$$

---

## 3. P6 Leveling Engine: Schedule Generation Schemes (SGS)

Modern project scheduling engines implement one of two constructive scheduling frameworks:

```text
               ┌────────────────────────────────────────────────────────┐
               │         SCHEDULE GENERATION SCHEMES (SGS)              │
               └──────────────────────────┬─────────────────────────────┘
                                          │
            ┌─────────────────────────────┴─────────────────────────────┐
            ▼                                                           ▼
┌───────────────────────┐                                   ┌───────────────────────┐
│      SERIAL SGS       │                                   │     PARALLEL SGS      │
├───────────────────────┤                                   ├───────────────────────┤
│ Activity-oriented.    │                                   │ Time-oriented.        │
│ Iterates n times:     │                                   │ Moves forward clock   │
│ Selects best task     │                                   │ t = 0, 1, 2, ...      │
│ from eligible list,   │                                   │ At time t, selects    │
│ places at earliest    │                                   │ all possible tasks    │
│ valid time window.    │                                   │ that fit capacity.    │
└───────────────────────┘                                   └───────────────────────┘
```

### The Parallel Schedule Generation Scheme (P-SGS)
P6's leveling engine predominantly behaves according to the **Parallel SGS** because it naturally handles time-based calendar shifts and step-by-step capacity validation:

1. Initialize time clock $t = 0$.
2. Form the **Eligible Set** $E_t$: All unscheduled activities whose predecessors have completed by time $t$, and whose earliest permissible start date $ES_j \le t$.
3. Sort $E_t$ according to the **Priority Heuristic Rule**.
4. For each activity $j \in E_t$ in ranked order:
   - Check if resource demand $r_{jk} \le \text{Remaining Capacity}_k(t, t + d_j)$ for all resources $k \in R$.
   - **If YES:** Schedule activity $j$ to start at time $t$. Deduct resource demand.
   - **If NO:** Activity $j$ cannot start at time $t$. It is delayed.
5. If eligible tasks remain that could not be scheduled, advance clock $t$ to the next activity completion event. Repeat until all activities are scheduled.

---

## 4. Priority Rules & Tie-Breaking Heuristics

When two or more activities demand the same bottleneck resource simultaneously, how does the engine decide who goes first?

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           P6 LEVELING PRIORITY RULES                             │
├──────────────────┬─────────────────────────────────────┬─────────────────────────┤
│ Rule Code        │ Mathematical Definition             │ Physical Intuition      │
├──────────────────┼─────────────────────────────────────┼─────────────────────────┤
│ Min Total Float  │ $\min TF_j$                         │ Protect critical path;  │
│ (P6 Default)     │                                     │ lowest float goes first │
├──────────────────┼─────────────────────────────────────┼─────────────────────────┤
│ Early Start      │ $\min ES_j$                         │ First-come, first-served│
│ (FIFO)           │                                     │ chronological priority  │
├──────────────────┼─────────────────────────────────────┼─────────────────────────┤
│ Late Start       │ $\min LS_j$                         │ Least slack until delay │
│                  │                                     │ becomes critical        │
├──────────────────┼─────────────────────────────────────┼─────────────────────────┤
│ Activity ID      │ Lexicographic sort on ActivityCode  │ Deterministic tie-break │
└──────────────────┴─────────────────────────────────────┴─────────────────────────┘
```

### The Standard P6 Priority Hierarchy
In Primavera P6, when you click **Tools > Level Resources**, the default sorting hierarchy is:
1. **Activity Priority** (User-assigned Top, High, Normal, Low, Lowest)
2. **Total Float** (Ascending: activities with 0 or negative float go before activities with positive float)
3. **Early Start Date** (Ascending: earlier CPM starts go first)
4. **Activity ID** (Alphabetical string comparison: ensures pure algorithmic determinism)

---

## 5. Concrete Numerical Walkthrough: 4 Activities, 1 Pile Driver

Let us trace the exact mathematics on our running case study: **Surya 100MW Solar IPP Project** (Rajasthan, India).

### Scenario Setup
We have **one specialized hydraulic pile-driver machine** ($C_{\text{pile}} = 1$). Four pile-driving foundation tasks across four solar array blocks are ready to start simultaneously on Day 0 after site grading.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                    INPUT ACTIVITIES & CPM CHARACTERISTICS                    │
├───────┬──────────────────────────────┬──────────┬──────────┬─────────┬───────┤
│ ActID │ Description                  │ Dur ($d$)│ Dem ($r$)│ $ES, EF$│ $TF$  │
├───────┼──────────────────────────────┼──────────┼──────────┼─────────┼───────┤
│ ACT-A │ Block 1 Piling (Central Sub) │ 4 Days   │ 1 unit   │ [0, 4]  │ 0 d   │
│ ACT-B │ Block 2 Piling (Inverter 1)  │ 3 Days   │ 1 unit   │ [0, 3]  │ 2 d   │
│ ACT-C │ Block 3 Piling (Inverter 2)  │ 2 Days   │ 1 unit   │ [0, 2]  │ 1 d   │
│ ACT-D │ Block 4 Piling (Tracker Test)│ 3 Days   │ 1 unit   │ [0, 3]  │ 5 d   │
└───────┴──────────────────────────────┴──────────┴──────────┴─────────┴───────┘
Total Available Pile Drivers: 1 unit.
Unconstrained Day 0 Total Demand: 4 units (Over-allocated by 3 units!).
```

### Unconstrained Timeline (Collision)
```text
Day:    0     1     2     3     4     5     6     7     8     9     10    11    12
ACT-A: [═══════════════════]
ACT-B: [═════════════]
ACT-C: [═══════════]
ACT-D: [═════════════]
Demand:  4     4     3     2     0     0     0     0     0     0     0     0     0  <-- OVERALLOCATED!
Limit:   1     1     1     1     1     1     1     1     1     1     1     1     1
```

### Trace of Parallel SGS with Min Total Float Rule:

#### Step 1: Clock $t = 0$
- Eligible set: $\{ \text{ACT-A}, \text{ACT-B}, \text{ACT-C}, \text{ACT-D} \}$
- Priority ranking by $\min TF$:
  1. ACT-A ($TF = 0$)
  2. ACT-C ($TF = 1$)
  3. ACT-B ($TF = 2$)
  4. ACT-D ($TF = 5$)
- Resource availability: $C = 1$.
- Evaluate ACT-A: Demands 1 unit. Fits!
  - Schedule ACT-A: $[0, 4]$.
  - Remaining capacity at $t \in [0, 4) = 0$.
- Evaluate ACT-C, ACT-B, ACT-D: No capacity left. Cannot start.
- Clock advances to next completion: $t = 4$.

#### Step 2: Clock $t = 4$
- ACT-A finishes at $t = 4$. Pile driver released ($C = 1$).
- Eligible set remaining: $\{ \text{ACT-C}, \text{ACT-B}, \text{ACT-D} \}$
- Priority ranking by $\min TF$:
  1. ACT-C ($TF = 1$)
  2. ACT-B ($TF = 2$)
  3. ACT-D ($TF = 5$)
- Evaluate ACT-C: Demands 1 unit. Fits!
  - Schedule ACT-C: $[4, 6]$.
  - Remaining capacity at $t \in [4, 6) = 0$.
- Evaluate ACT-B, ACT-D: Cannot start.
- Clock advances to next completion: $t = 6$.

#### Step 3: Clock $t = 6$
- ACT-C finishes at $t = 6$. Pile driver released ($C = 1$).
- Eligible set remaining: $\{ \text{ACT-B}, \text{ACT-D} \}$
- Priority ranking by $\min TF$:
  1. ACT-B ($TF = 2$)
  2. ACT-D ($TF = 5$)
- Evaluate ACT-B: Demands 1 unit. Fits!
  - Schedule ACT-B: $[6, 9]$.
  - Remaining capacity at $t \in [6, 9) = 0$.
- Evaluate ACT-D: Cannot start.
- Clock advances to next completion: $t = 9$.

#### Step 4: Clock $t = 9$
- ACT-B finishes at $t = 9$. Pile driver released ($C = 1$).
- Eligible set remaining: $\{ \text{ACT-D} \}$
- Evaluate ACT-D: Demands 1 unit. Fits!
  - Schedule ACT-D: $[9, 12]$.
- Clock advances to $t = 12$. All tasks scheduled!

### Post-Leveling Schedule & Float Impact
```text
Day:    0     1     2     3     4     5     6     7     8     9     10    11    12
ACT-A: [═══════════════════]
ACT-C:                      [═══════════]
ACT-B:                                  [═════════════]
ACT-D:                                                [═════════════]
Demand:  1     1     1     1     1     1     1     1     1     1     1     1     0  <-- PERFECTLY LEVEL!
Limit:   1     1     1     1     1     1     1     1     1     1     1     1     1
```

```text
┌───────┬──────────┬─────────────┬──────────────┬──────────────┬──────────────┐
│ ActID │ Duration │ Original ES │ Leveled ES   │ Original TF  │ Leveled TF   │
├───────┼──────────┼─────────────┼──────────────┼──────────────┼──────────────┤
│ ACT-A │ 4 Days   │ Day 0       │ Day 0        │ 0 Days       │ 0 Days       │
│ ACT-C │ 2 Days   │ Day 0       │ Day 4        │ 1 Day        │ -3 Days*     │
│ ACT-B │ 3 Days   │ Day 0       │ Day 6        │ 2 Days       │ -4 Days*     │
│ ACT-D │ 3 Days   │ Day 0       │ Day 9        │ 5 Days       │ -4 Days*     │
└───────┴──────────┴─────────────┴──────────────┴──────────────┴──────────────┘
* Note: If project completion milestone is constrained to Day 4 (the original CPM finish),
  shifting downstream activities consumes all float and creates negative float.
  In P6, you can choose "Level within total float" to forbid project end date expansion!
```

---

## 6. The Burgess-Killebrew Heuristic for Resource Smoothing

When resources are not hard-constrained, but you want to **smooth peaks and valleys** to minimize hiring and firing costs, P6 uses **Resource Smoothing**.

The classic mathematical metric to minimize is the **Sum of Squared Resource Demands** over time:

$$S = \sum_{t=1}^{T} \left( \sum_{j \in V_t} r_{jk} \right)^2$$

Where $V_t$ is the set of activities active at time $t$.

### Why Minimize the Sum of Squares?
Squaring penalizes large resource spikes much more heavily than flat, consistent usage:
- **Case 1 (Spike):** Day 1 demand = 10, Day 2 demand = 2.
  $$S_1 = 10^2 + 2^2 = 100 + 4 = 104$$
- **Case 2 (Smooth):** Day 1 demand = 6, Day 2 demand = 6 (Total demand is identical = 12).
  $$S_2 = 6^2 + 6^2 = 36 + 36 = 72$$

By shifting activities with positive float ($TF_j > 0$) within their valid windows $[ES_j, LS_j]$, the algorithm finds the schedule configuration that minimizes $S$ without delaying project completion.

---

## 7. Production-Grade Golang Leveling Implementation

Below is the clean, self-contained Golang implementation of a multi-resource Parallel Schedule Generation Scheme leveling engine:

```go
package scheduling

import (
	"fmt"
	"sort"
)

// Resource represents a finite renewable site resource (e.g., Crane, Piling Rig)
type Resource struct {
	ID       string
	Name     string
	Capacity int
}

// Activity represents a schedulable task
type Activity struct {
	ID           string
	Name         string
	Duration     int
	Predecessors []string
	Demand       map[string]int // ResourceID -> required units per day
	ES           int            // Early Start (CPM)
	EF           int            // Early Finish (CPM)
	LS           int            // Late Start (CPM)
	LF           int            // Late Finish (CPM)
	TotalFloat   int

	// Output fields post-leveling
	LeveledStart  int
	LeveledFinish int
	IsScheduled   bool
}

// LevelingEngine runs Parallel SGS with Min Total Float rule
type LevelingEngine struct {
	Activities map[string]*Activity
	Resources  map[string]*Resource
}

func NewLevelingEngine(acts []*Activity, res []*Resource) *LevelingEngine {
	actMap := make(map[string]*Activity)
	for _, a := range acts {
		actMap[a.ID] = a
	}
	resMap := make(map[string]*Resource)
	for _, r := range res {
		resMap[r.ID] = r
	}
	return &LevelingEngine{
		Activities: actMap,
		Resources:  resMap,
	}
}

// Level executes Parallel SGS
func (le *LevelingEngine) Level() {
	// Track resource utilization by time: timeBucket[resourceID][day] = unitsUsed
	usage := make(map[string]map[int]int)
	for rID := range le.Resources {
		usage[rID] = make(map[int]int)
	}

	clock := 0
	totalActivities := len(le.Activities)
	scheduledCount := 0

	for scheduledCount < totalActivities {
		// 1. Find all eligible activities at current clock
		eligible := []*Activity{}
		for _, act := range le.Activities {
			if act.IsScheduled {
				continue
			}
			// Check if all predecessors are completed by 'clock'
			predsDone := true
			for _, pID := range act.Predecessors {
				pred := le.Activities[pID]
				if !pred.IsScheduled || pred.LeveledFinish > clock {
					predsDone = false
					break
				}
			}
			// Must also satisfy original early start
			if predsDone && act.ES <= clock {
				eligible = append(eligible, act)
			}
		}

		// 2. Sort eligible tasks by Min Total Float, then Early Start, then ID
		sort.Slice(eligible, func(i, j int) bool {
			if eligible[i].TotalFloat != eligible[j].TotalFloat {
				return eligible[i].TotalFloat < eligible[j].TotalFloat
			}
			if eligible[i].ES != eligible[j].ES {
				return eligible[i].ES < eligible[j].ES
			}
			return eligible[i].ID < eligible[j].ID
		})

		// 3. Attempt to schedule eligible activities
		anyScheduledAtClock := false
		for _, act := range eligible {
			// Check if resources are available across all days [clock, clock + act.Duration)
			canFit := true
			for rID, dem := range act.Demand {
				capLimit := le.Resources[rID].Capacity
				for d := clock; d < clock+act.Duration; d++ {
					if usage[rID][d]+dem > capLimit {
						canFit = false
						break
					}
				}
				if !canFit {
					break
				}
			}

			if canFit {
				// Allocate resources
				for rID, dem := range act.Demand {
					for d := clock; d < clock+act.Duration; d++ {
						usage[rID][d] += dem
					}
				}
				act.LeveledStart = clock
				act.LeveledFinish = clock + act.Duration
				act.IsScheduled = true
				scheduledCount++
				anyScheduledAtClock = true
			}
		}

		// 4. Advance clock to next earliest event
		// If someone scheduled, clock can advance to next completion or next day
		// Advance by 1 day step to maintain granular checks
		clock++
	}
}
```

---

## 8. Summary of Key Operational Lessons

1. **Total Float is dynamic:** In standard CPM, total float belongs to the activity path. Once leveled, activities lose their float to resource queues. P6 stores this as **Resource Float**.
2. **Activity Splitting:** P6 allows an option to *"Split activities during leveling"*. If enabled, an activity can start, pause when a higher-priority task arrives, and resume later. For EPC construction (e.g., concrete pouring), splitting is strictly **prohibited**.
3. **Leveling within Float:** If the client contract imposes a hard liquidated damages deadline, uncheck *"Preserve scheduled early and late dates"* and enable *"Level only within total float"*. If a resource conflict cannot be resolved without delaying the project, P6 logs an unresolved resource conflict in the schedule log (`schedlog.txt`).
