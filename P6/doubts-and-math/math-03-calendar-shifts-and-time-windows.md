# Calendar Shifts & Time Windows: Demystifying Work Intervals & Timestamp Arithmetic

---

## 1. Executive Summary & The Core Mental Model

### 1.1 The Axiom: Physical Time vs. Contractual Working Time
In naive project management software (such as basic spreadsheets or sprint boards), time is treated as a simple, continuous number line. If a task begins on Friday and takes 3 days, naive arithmetic assumes it finishes on Monday ($1 + 3 = 4$).

In enterprise CPM engines like **Oracle Primavera P6**, this naive assumption is fatal.

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        THE CALENDAR AS A MATHEMATICAL FILTER MASK                      │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                        │
│   PHYSICAL TIME (THE UNIVERSAL TICK):                                                  │
│   Continuous, immutable progression of UTC seconds. 24 hours/day, 7 days/week, 365d/yr.│
│   Concrete cures, steel oxidizes, and loan interest accrues in physical time.          │
│                                                                                        │
│                                           │                                            │
│                                           ▼                                            │
│                              [CALENDAR FILTER MASK]                                    │
│                     (Shifts, Lunch Breaks, Weekends, Holidays)                         │
│                                           │                                            │
│                                           ▼                                            │
│                                                                                        │
│   CONTRACTUAL WORKING TIME (THE ENGINE TICK):                                          │
│   Time ticks ONLY when labor and machinery are legally permitted to work on site.      │
│   When the calendar is CLOSED (e.g. Sunday), the CPM engine clock FREEZES.             │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

A **Calendar** in Primavera P6 is a mathematical step function—a binary mask over physical time:
$$M(t) = \begin{cases} 1 & \text{if timestamp } t \text{ is an open working minute} \\ 0 & \text{if timestamp } t \text{ is non-work (night, weekend, holiday)} \end{cases}$$

An activity with Planned Duration $D$ (expressed in hours) completes only when the accumulated integral of working time equals $D$:
$$\int_{ES}^{EF} M(t) \, dt = D$$

This fundamental truth explains why:
* **3 Working Days** can span **105 physical hours** (over a weekend).
* **3 Working Days** can span **57 physical hours** (on a 7-day week).
* **3 Calendar Days** can span **72 physical hours** (continuous 24/7 curing).

---

## 2. Anatomy of a P6 Calendar: Shifts, Hours & Conversions

To predict CPM date arithmetic with 100% precision, one must master the internal configuration of a P6 calendar.

### 2.1 The Standard 5-Day Workweek Definition (8 Hours/Day)
The most common industrial calendar configured on construction projects is the **5-Day 8-Hour Workweek**:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         STANDARD 5-DAY WORKWEEK DAILY SCHEDULE                         │
├──────────────────────┬──────────────────────┬───────────────────┬──────────────────────┤
│        PERIOD        │      TIME WINDOW     │   WORKING HOURS   │     CALENDAR MASK    │
├──────────────────────┼──────────────────────┼───────────────────┼──────────────────────┤
│ Morning Shift        │ 08:00 AM - 12:00 PM  │     4.0 Hours     │ M(t) = 1 (OPEN)      │
│ Lunch Break          │ 12:00 PM - 01:00 PM  │     0.0 Hours     │ M(t) = 0 (CLOSED)    │
│ Afternoon Shift      │ 01:00 PM - 05:00 PM  │     4.0 Hours     │ M(t) = 1 (OPEN)      │
│ Night / Off-Shift    │ 05:00 PM - 08:00 AM  │     0.0 Hours     │ M(t) = 0 (CLOSED)    │
├──────────────────────┴──────────────────────┼───────────────────┼──────────────────────┤
│ TOTAL DAILY WORK CAPACITY                   │     8.0 Hours     │                      │
└─────────────────────────────────────────────┴───────────────────┴──────────────────────┘
```

* **Work Days:** Monday, Tuesday, Wednesday, Thursday, Friday.
* **Non-Work Days:** Saturday, Sunday.

---

### 2.2 The Conversion Factor Setting Trap
In Primavera P6, when you type `3d` into the Duration field, how does P6 know what `3d` means?

Under **Admin $\to$ Admin Preferences $\to$ Time Periods**, P6 defines global conversion multipliers:

```text
┌────────────────────────────────────────────────────────┐
│ Hours per Day:    8.0 Hours                            │
│ Hours per Week:   40.0 Hours                           │
│ Hours per Month:  172.0 Hours                          │
│ Hours per Year:   2000.0 Hours                         │
└────────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **The 10-Hour Shift Mismatch Trap:**  
> Suppose your site runs on a 10-hour shift (07:00 to 18:00 with 1 hour lunch).  
> If the global Admin Preference is still set to `8.0 Hours/Day`, and a planner types `3d`, P6 multiplies:
> $$3 \times 8.0 = 24.0\text{ working hours}$$
> But the activity's calendar has 10 working hours per day!  
> Therefore, P6 will schedule the task to consume:
> * Day 1: 10 hours
> * Day 2: 10 hours
> * Day 3: 4 hours
> The task finishes at 11:00 AM on Day 3! To the planner who expected 3 full days of work, the schedule appears broken. P6 is not broken—the conversion settings do not match the calendar shift definition!

---

## 3. Case Study 1: The 5-Day Calendar Weekend Skip Walkthrough

Let us solve the exact scenario that perplexes junior planners:

> **The Problem:**  
> Activity A: *Central Inverter Skid Foundation Pit Excavation*.  
> Planned Duration = **3 Work Days (24 Work Hours)**.  
> Scheduled Start = **Friday Morning at 08:00 AM**.  
> Calendar = **Standard 5-Day Workweek (Mon–Fri 08:00–17:00, Sat–Sun Non-work)**.  
> **Question: Exactly when does Activity A finish?**

### 3.1 The Rookie Mistakes
* **Mistake 1 (Naive 24-Hour Addition):**  
  Friday 08:00 + 3 days (72 hours) = Monday 08:00 AM. $\to$ **WRONG!** (Ignores shifts and weekends).
* **Mistake 2 (Naive Calendar Day Addition):**  
  Friday + 3 days = Saturday, Sunday, Monday $\to$ Monday 17:00 PM. $\to$ **WRONG!** (Ignores that Saturday and Sunday are non-working days).

---

### 3.2 The Hour-by-Hour Mathematical Accounting

Let us trace the execution minute by minute:

```text
════════════════════════════════════════════════════════════════════════════════════════════════
DAY 1: FRIDAY (WORKDAY 1)
════════════════════════════════════════════════════════════════════════════════════════════════
• 08:00 AM: Activity A Starts! (Elapsed Work = 0.0 hrs, Remaining = 24.0 hrs)
• 08:00 AM - 12:00 PM: 4.0 work hours completed.
• 12:00 PM - 01:00 PM: Lunch Break (0.0 work hours). Clock paused!
• 01:00 PM - 05:00 PM: 4.0 work hours completed.
• 05:00 PM (Friday Close):
  - Work completed today: 4.0 + 4.0 = 8.0 Hours (1 full work day).
  - Cumulative Work Done: 8.0 Hours.
  - Remaining Work: 24.0 - 8.0 = 16.0 Hours.

════════════════════════════════════════════════════════════════════════════════════════════════
THE WEEKEND FREEZE: SATURDAY & SUNDAY (NON-WORK DAYS)
════════════════════════════════════════════════════════════════════════════════════════════════
• Friday 05:00 PM to Monday 08:00 AM:
  - Total Physical Elapsed Time: 63 Clock Hours.
  - Calendar Mask M(t) = 0.
  - Work Completed: 0.0 Hours.
  - Remaining Work: 16.0 Hours (UNCHANGED).

════════════════════════════════════════════════════════════════════════════════════════════════
DAY 2: MONDAY (WORKDAY 2)
════════════════════════════════════════════════════════════════════════════════════════════════
• 08:00 AM: Site resumes.
• 08:00 AM - 12:00 PM: 4.0 work hours completed.
• 12:00 PM - 01:00 PM: Lunch Break (0.0 work hours).
• 01:00 PM - 05:00 PM: 4.0 work hours completed.
• 05:00 PM (Monday Close):
  - Work completed today: 8.0 Hours.
  - Cumulative Work Done: 8.0 + 8.0 = 16.0 Hours (2 full work days).
  - Remaining Work: 24.0 - 16.0 = 8.0 Hours.

════════════════════════════════════════════════════════════════════════════════════════════════
DAY 3: TUESDAY (WORKDAY 3 - FINAL DAY)
════════════════════════════════════════════════════════════════════════════════════════════════
• 08:00 AM: Site resumes.
• 08:00 AM - 12:00 PM: 4.0 work hours completed. (Remaining = 4.0 hrs).
• 12:00 PM - 01:00 PM: Lunch Break (0.0 hrs).
• 01:00 PM - 05:00 PM: Final 4.0 work hours completed!
• 05:00 PM (Tuesday Close):
  - Work completed today: 8.0 Hours.
  - Cumulative Work Done: 16.0 + 8.0 = 24.0 Hours (3 full work days).
  - Remaining Work: 0.0 Hours!
  - ACTIVITY A IS 100% COMPLETE.
```

### 3.3 The Rigorous Output Dates
* **Early Start ($ES$):** `Friday 08:00 AM`
* **Early Finish ($EF$):** `Tuesday 05:00 PM (17:00)`
* **Immediate Successor Early Start ($ES_{\text{succ}}$):** `Wednesday 08:00 AM` (the next available working minute!).

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                           THE PHYSICAL TIME DISPARITY                                  │
├────────────────────────────────────────────────────────────────────────────────────────┤
│   • Contractual Working Duration:   24 Hours (3 Work Days)                             │
│   • Physical Elapsed Calendar Time: 105 Hours (4 Calendar Days and 9 Hours)            │
│   • Ratio of Elapsed Time to Work:  4.375x !                                           │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Visual Comparison: 5-Day Standard vs. 7-Day Curing Calendar

Now let us contrast this with a foundation concrete curing operation.

### 4.1 The Physics of Concrete Curing
Portland cement does not check a calendar or respect union weekends. The chemical reaction between water and calcium silicate (hydration) occurs continuously, 24 hours a day, 7 days a week.

Therefore, civil engineering specifications prescribe curing on a **7-Day Calendar**:
* **Calendar A:** `7-Day 8-Hour Calendar` (for supervised water sprinkling/curing inspection).
* **Calendar B:** `7-Day 24-Hour Continuous Calendar` (for uninterrupted curing membrane/ponding).

---

### 4.2 Side-by-Side ASCII Timeline Grid

Let us visualize a 3-day duration starting **Friday morning at 08:00 AM** under both calendars:

```text
Day:         │  FRIDAY   │ SATURDAY  │  SUNDAY   │  MONDAY   │  TUESDAY  │ WEDNESDAY │
Hour Window: │08       17│08       17│08       17│08       17│08       17│08       17│
─────────────┼───────────┼───────────┼───────────┼───────────┼───────────┼───────────┤
SCENARIO 1:  │           │           │           │           │           │           │
5-Day Earth  │ █████████ │           │           │ █████████ │ █████████ │           │
Excavation   │  Day 1    │ [WEEKEND: │ [WEEKEND: │  Day 2    │  Day 3    │ Successor │
(24 Work Hrs)│  (8 hrs)  │  FROZEN]  │  FROZEN]  │  (8 hrs)  │  (8 hrs)  │  Starts!  │
             │           │           │           │           │▲ FINISH:  │▲ START:   │
             │           │           │           │           │ Tue 17:00 │ Wed 08:00 │
─────────────┼───────────┼───────────┼───────────┼───────────┼───────────┼───────────┤
SCENARIO 2:  │           │           │           │           │           │           │
7-Day Curing │ █████████ │ █████████ │ █████████ │           │           │           │
Inspection   │  Day 1    │  Day 2    │  Day 3    │ Successor │           │           │
(24 Work Hrs)│  (8 hrs)  │  (8 hrs)  │  (8 hrs)  │  Starts!  │           │           │
             │           │           │▲ FINISH:  │▲ START:   │           │           │
             │           │           │ Sun 17:00 │ Mon 08:00 │           │           │
─────────────┼───────────┼───────────┼───────────┼───────────┼───────────┼───────────┤
Key: ██ Active Work Hours (8 hrs/day)      Blank Space = Non-work / Off-shift
```

### 4.3 Key Observations
1. **The 5-Day Task** finishes on **Tuesday at 17:00 PM**.
2. **The 7-Day Task** finishes on **Sunday at 17:00 PM**.
3. **The Calendar Difference Saved 2 Full Calendar Days (48 Hours)** of project time simply by recognizing the true physical calendar of the trade!

---

## 5. The Multi-Calendar Clash: Where Calendars Collide Across Logic Ties

When two activities with different calendars are linked together, which calendar governs the transition?

### 5.1 Case Study: 7-Day Curing Connecting to 5-Day Heavy Rigging
Consider the sequential construction logic at our Surya 100MW Inverter Station:
* **Predecessor Task 1:** *Transformer Plinth Concrete Curing*.
  * Calendar: `7-Day 8-Hour Curing Calendar`.
  * Start: Friday 08:00 AM $\implies$ Finish: **Sunday 17:00 PM**.
* **Successor Task 2:** *33kV / 3.3MVA Inverter Transformer Rigging & Placement*.
  * Calendar: `5-Day Standard Construction Calendar` (Heavy mobile cranes cannot operate on weekends due to district transport authority road permits).
  * Relationship: `Finish-to-Start (FS = 0)`.

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              THE HANDOFF OVER SUNDAY NIGHT                             │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                        │
│   Task 1 (Curing) Finishes:     Sunday at 17:00 PM                                     │
│                                                                                        │
│   Physical Next Minute:         Sunday at 17:01 PM                                     │
│   Does Task 2 Start Sunday?     NO! Task 2's calendar is CLOSED on Sunday.             │
│                                                                                        │
│   Does Task 2 Start Monday?     YES! Monday 08:00 AM is the FIRST OPEN MINUTE          │
│                                 on Task 2's calendar following Task 1's finish.        │
│                                                                                        │
│   Effective Calendar Delay:     15 hours (overnight non-work). Zero working days lost! │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### 5.2 The Notorious "Relationship Lag Calendar" Dilemma
What happens when you add a **Lag** between two activities on different calendars?

> **The Setup:**  
> Task X (5-Day Calendar) $\xrightarrow{FS + 2\text{ Days Lag}}$ Task Y (7-Day Calendar).  
> Task X finishes on **Friday at 17:00 PM**.  
> When does Task Y start?

The answer depends entirely on a hidden setting inside Primavera P6!
Under **Schedule Options ($F9$) $\to$ Calendar for scheduling Relationship Lag**:

```text
┌──────────────────────────────────────────────────────────────────┐
│ Calendar for scheduling Relationship Lag:                        │
│                                                                  │
│   ( ) Predecessor Activity Calendar                              │
│   ( ) Successor Activity Calendar                                │
│   (•) 24-Hour Calendar                                           │
│   ( ) Project Default Calendar                                   │
└──────────────────────────────────────────────────────────────────┘
```

#### Outcome A: If Lag uses "Predecessor Calendar" (5-Day)
* Task X finishes Friday 17:00 PM.
* Lag = 2 days on a 5-day calendar.
* Saturday & Sunday are NON-WORK for the lag!
* Lag consumes **Monday** (Lag Day 1) and **Tuesday** (Lag Day 2).
* Task Y starts on **Wednesday at 08:00 AM**!

#### Outcome B: If Lag uses "Successor Calendar" (7-Day)
* Task X finishes Friday 17:00 PM.
* Lag = 2 days on a 7-day calendar.
* Saturday and Sunday ARE OPEN for the lag!
* Lag consumes **Saturday** (Lag Day 1) and **Sunday** (Lag Day 2).
* Task Y starts on **Monday at 08:00 AM**!

> [!WARNING]
> **A 48-Hour Contractual Variance!**  
> Simply changing this one dropdown in P6 from *Predecessor Calendar* to *Successor Calendar* moved the successor start date by **2 entire calendar days (from Wednesday to Monday)**!  
> On an EPC contract with LDs of $50,000/day, this setting can swing liability by $100,000 without a single shovel touching dirt.

---

## 6. The #1 UI Trap in Primavera P6: The 14:00 Start Disaster

Every senior scheduler has faced this panicky question from a project manager:
> *"I added a 1-day task to P6 with Start on Oct 12 and Duration = 1d. Why is P6 showing Finish as Oct 13?! It says 1 day, but it's taking 2 days!"*

### 6.1 The Root Cause: Hidden Timestamps
By default, P6 displays dates in date-only format (`DD-Mon-YY`), hiding the time component.
What actually happened behind the scenes:
1. The predecessor finished at **14:00 PM (2:00 PM)** on Oct 12.
2. The successor activity's Early Start was set to **Oct 12 at 14:00 PM**.
3. The activity needs **8 working hours** (1 day).
4. On Oct 12: Work runs from 14:00 to 17:00 = **3 hours consumed**.
5. Day ends! 5 hours remain.
6. On Oct 13: Work resumes at 08:00 AM. 5 hours runs until **13:00 PM (1:00 PM)**.
7. Result:
   * Start: `12-Oct-26 14:00`
   * Finish: `13-Oct-26 13:00`
8. When timestamps are hidden, P6 displays:
   * Start: `12-Oct-26`
   * Finish: `13-Oct-26`
9. To an untrained eye, a 1-day task looks like it spans two full days!

### 6.2 The Mentor's Remedy: Always Turn On Timestamps
To diagnose and prevent timestamp mismatches:
1. Go to **Edit $\to$ User Preferences**.
2. Select the **Dates** tab.
3. Under **Time**, select **`Time (12 hour)`** or **`Time (24 hour)`** (e.g., `08:30`).
4. Now the Gantt table immediately shows `12-Oct-26 14:00` to `13-Oct-26 13:00`, instantly revealing the 3-hour afternoon start.

---

## 7. Practical Rules of Thumb & Interview Mastery

### 7.1 Quick Mental Math Shortcuts for Schedulers
1. **The Friday Rule for 5-Day Calendars:**  
   Any task starting Friday morning with duration $D$ work days will add $2 \times \lfloor (D - 1) / 5 \rfloor + 2$ calendar weekend days to its elapsed duration.  
   *(A 3-day task adds 2 days for the weekend: $3 + 2 = 5$ calendar days).*
2. **The Curing Rule:**  
   Always assign concrete curing, grouting, paint drying, and environmental testing to a **7-Day or 24-Hour Calendar**. Never let chemical hydration pause for human weekends!
3. **The Milestone Zero-Duration Rule:**  
   A Start Milestone has $ES = EF = \text{Start of Workday (08:00 AM)}$.  
   A Finish Milestone has $ES = EF = \text{End of Workday (17:00 PM)}$.

---

### 7.2 Interview Questions for Senior Planning Engineers

#### Q1: If an activity on a 5-day calendar has a Finish-to-Start predecessor on a 7-day calendar that finishes on Saturday at 17:00, when does the successor start?
**Answer:**  
The successor starts on **Monday at 08:00 AM**.  
Although the predecessor finishes on Saturday afternoon, the successor cannot begin work until its own calendar enters an open working interval, which occurs on Monday at 08:00 AM.

---

#### Q2: What causes "Phantom Float" in multi-calendar schedules?
**Answer:**  
Phantom Float occurs when an activity appears to have Total Float solely due to calendar work period differences rather than true project flexibility. For instance, if a critical activity on a 7-day calendar feeds an activity on a 5-day calendar on Friday evening, Saturday and Sunday may appear as 2 days of float on the predecessor simply because the successor's calendar is closed. Planners must scrutinize float paths across calendar boundaries to ensure critical work is not falsely reported as non-critical.

---

#### Q3: Why does AACE International recommend using a 24-Hour calendar for Relationship Lags in complex EPC schedules?
**Answer:**  
Using a 24-Hour calendar ensures that lag represents true physical elapsed time (such as 48 hours of curing or shipping transit) independent of any specific trade's shift schedule. This eliminates disputes between subcontractors over whose calendar governs the lag and avoids unexpected weekend skips for physical processes that occur continuously.
