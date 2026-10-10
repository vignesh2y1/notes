# EVM Arithmetic Demystified: The Earned Value Management Mathematical Guide

---

## 1. Executive Summary & The Core Mental Model

### 1.1 The Fundamental Flaw of Traditional Project Accounting
In construction and heavy infrastructure—such as the **Surya 100MW Solar IPP** in Rajasthan—traditional project accounting compares only two numbers:
1. **Budget Planned to Date** (What finance expected to spend).
2. **Actual Money Spent to Date** (What accounts payable disbursed).

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        THE TRADITIONAL ACCOUNTING TRAP (A FAULTY REALITY)              │
├────────────────────────────────────────────────────────────────────────────────────────┤
│   Scenario at Month 6:                                                                 │
│   • Planned Budget to Date: ₹6.00 Crore ($720k)                                        │
│   • Actual Invoices Paid:   ₹5.50 Crore ($660k)                                        │
│                                                                                        │
│   Chief Financial Officer (CFO) Conclusion:                                            │
│   "Excellent news! We planned to spend ₹6.00 Cr, but we only spent ₹5.50 Cr.           │
│    We have saved ₹50 Lakhs! The project is under budget and running smoothly."        │
│                                                                                        │
│                                           ❌                                           │
│                                                                                        │
│   Job-Site Reality (Project Director on the Ground):                                   │
│   "Disaster! We only completed 24 inverter foundations out of 30 planned.              │
│    The work we actually did was only worth ₹4.80 Crore.                                │
│    We spent ₹5.50 Crore to get ₹4.80 Crore of real physical assets.                    │
│    We are BOTH behind schedule AND severely bleeding cash!"                            │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

Traditional accounting cannot detect this crisis because it lacks the third dimension: **Physical Progress (Earned Value)**.

### 1.2 The Golden Triangle of Earned Value Management (EVM)
Earned Value Management (EVM) resolves this blindness by uniting three distinct dimensions into a single mathematical coordinate system:

```text
                           SCOPE & BUDGET
                           Baseline (BAC)
                                 ▲
                                / \
                               /   \
       PLANNED VALUE (PV)     /     \     EARNED VALUE (EV)
     "What was scheduled?"   /       \   "What was physically built?"
                            /         \
                           ▼           ▼
                   TIME / SCHEDULE ◄───► COST / EXPENDITURE
                               ACTUAL COST (AC)
                             "What was invoiced?"
```

1. **Planned Value ($PV$) / BCWS (Budgeted Cost of Work Scheduled):**  
   The authorized budget allocated to work scheduled to be accomplished up to the Data Date.  
   $$\text{Question: } \text{\textit{"How much work should we have finished by today?"}}$$

2. **Earned Value ($EV$) / BCWP (Budgeted Cost of Work Performed):**  
   The value of physical work actually completed, quantified using the original baseline budget rates.  
   $$\text{Question: } \text{\textit{"How much is the work we actually built worth according to our contract baseline?"}}$$

3. **Actual Cost ($AC$) / ACWP (Actual Cost of Work Performed):**  
   The total realized cost incurred in executing the work completed up to the Data Date.  
   $$\text{Question: } \text{\textit{"How much cash has actually left our bank account to deliver that work?"}}$$

---

## 2. The Industrial Benchmark: Surya 100MW Solar IPP Case Study

To eliminate abstract theory, every formula in this guide is derived using pure numbers from an active, mission-critical engineering package:

### 2.1 Package Specification: Inverter Station Plinths & Transformer Skids
* **Scope:** 50 Heavy Reinforced Concrete Inverter & Transformer Foundation Plinths (50 units).
* **Location:** Surya 100MW Solar IPP, Bhadla, Rajasthan.
* **Contract Budget at Completion ($BAC$):** **₹10,00,00,000 (₹10.00 Crore / ~\$1,200,000 USD)**.
* **Unit Budget per Plinth:**
  $$\text{Unit Budget} = \frac{BAC}{50 \text{ Plinths}} = \frac{\text{₹}10,00,00,000}{50} = \text{₹}20,00,000 \text{ per Plinth (₹20 Lakhs / \$24k USD)}$$
* **Baseline Schedule:** 10 Months (Uniform baseline plan of 5 plinths per month).
* **Current Status Date (Primavera P6 Data Date):** **End of Month 6**.

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│             SURYA 100MW SOLAR IPP: INVERTER FOUNDATION PACKAGE PROFILE                │
├───────────────────┬───────────────────────────────────┬────────────────────────────────┤
│ Parameter         │ Metric (Indian Rupee - INR)       │ US Dollar Equivalent (USD)     │
├───────────────────┼───────────────────────────────────┼────────────────────────────────┤
│ Scope Size        │ 50 Foundations (Civil + Conduits) │ 50 Foundations                 │
│ Total Budget (BAC)│ ₹10,00,00,000 (10.00 Crore)       │ $1,200,000                     │
│ Unit Rate / Plinth│ ₹20,00,000 (20 Lakhs)             │ $24,000                        │
│ Baseline Timeline │ 10 Months (5 plinths/month)       │ 10 Months                      │
│ Data Date (Cutoff)│ Month 6.0                         │ Month 6.0                      │
└───────────────────┴───────────────────────────────────┴────────────────────────────────┘
```

---

## 3. The Core Trio: Step-by-Step Arithmetic at Month 6

At the Month 6 Data Date, project controls and site engineering report the following ground realities:
* **Planned Schedule Target:** Exactly 6 months of work should be completed.
  $$\text{Planned Plinths} = 6 \text{ months} \times 5 \text{ plinths/month} = 30 \text{ plinths}$$
* **Physical Progress Survey (Signed by Quality Inspector):** Only **24 plinths** have completed 28-day concrete curing and electrical conduit sign-off.
* **Accounts Payable Ledger (ERP SAP/Oracle Cost Records):** Civil contractor invoices, rebar mill certs, ready-mix concrete bills, and site plant hire total **₹5,50,00,000 (₹5.50 Crore)**.

Let us compute the Core Trio:

### 3.1 Planned Value ($PV$ / BCWS)
$$\text{Planned Value } (PV) = \text{Planned \% Complete} \times BAC$$
$$\text{Planned \% Complete} = \frac{30 \text{ plinths planned}}{50 \text{ total plinths}} = 60.00\%$$
$$PV = 60.00\% \times \text{₹}10,00,00,000 = \mathbf{\text{₹}6,00,00,000 \quad (₹6.00 \text{ Crore})}$$

*Plain-English Meaning:* According to our Primavera P6 baseline schedule, we committed to have finished ₹6.00 Crore worth of civil foundations by Month 6.

---

### 3.2 Earned Value ($EV$ / BCWP)
$$\text{Earned Value } (EV) = \text{Physical \% Complete} \times BAC$$
$$\text{Physical \% Complete} = \frac{24 \text{ plinths cast and approved}}{50 \text{ total plinths}} = 48.00\%$$
$$EV = 48.00\% \times \text{₹}10,00,00,000 = \mathbf{\text{₹}4,80,00,000 \quad (₹4.80 \text{ Crore})}$$

*Plain-English Meaning:* The physical assets standing on the ground are worth exactly ₹4.80 Crore in contract value.

> [!IMPORTANT]
> **Notice the Golden Rule of Earned Value:**  
> $EV$ is calculated by multiplying physical progress by the **Baseline Unit Rate**, NOT what you paid for it!  
> 24 plinths $\times$ ₹20 Lakhs baseline rate = ₹4.80 Crore.

---

### 3.3 Actual Cost ($AC$ / ACWP)
$$AC = \mathbf{\text{₹}5,50,00,000 \quad (₹5.50 \text{ Crore})}$$

*Plain-English Meaning:* The EPC consortium has actually spent ₹5.50 Crore in cash to build those 24 plinths.

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        MONTH 6 CORE METRICS AT A GLANCE                                │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                        │
│   PV (What we planned to do):        ₹6.00 Crore  (30 plinths planned)                 │
│                                                                                        │
│   AC (What we spent):                ₹5.50 Crore  (Realized expenditures)              │
│                                                                                        │
│   EV (What we actually achieved):    ₹4.80 Crore  (24 plinths built @ ₹20L)            │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Variances Arithmetic: Cost Variance & Schedule Variance

Variances represent absolute financial deltas measured in currency (INR / USD).

```text
                     EVM VARIANCE FORMULA AXIOM:
                     ---------------------------
              Variance = Earned Value (EV) - Comparison Baseline
              
              • Cost Variance:     CV = EV - AC  (EV compared to Money Spent)
              • Schedule Variance: SV = EV - PV  (EV compared to Schedule Plan)

              RULE: 
              Positive (+) = Favorable (Good)
              Zero     (0) = Exactly on Target
              Negative (-) = Unfavorable (Bad / Slippage / Overrun)
```

### 4.1 Cost Variance ($CV$) Arithmetic
$$CV = EV - AC$$
$$CV = \text{₹}4,80,00,000 - \text{₹}5,50,00,000 = \mathbf{-\text{₹}70,00,000 \quad (-₹0.70 \text{ Crore / } -\$84,000 \text{ USD})}$$

#### Boardroom Interpretation of $CV = -₹70 \text{ Lakhs}$:
* The package has experienced a **cost overrun of ₹70 Lakhs**.
* To create ₹4.80 Crore worth of foundations, the civil team consumed ₹5.50 Crore.
* Every foundation built to date has cost:
  $$\text{Realized Unit Cost} = \frac{\text{₹}5.50 \text{ Crore}}{24 \text{ Plinths}} = \text{₹}22.916 \text{ Lakhs per Plinth}$$
  We are over-budget by **₹2.916 Lakhs per foundation** (due to unforeseen rock excavation, bentonite slurry pumping, and overtime labor).

---

### 4.2 Schedule Variance ($SV$) Arithmetic
$$SV = EV - PV$$
$$SV = \text{₹}4,80,00,000 - \text{₹}6,00,00,000 = \mathbf{-\text{₹}1,20,00,000 \quad (-₹1.20 \text{ Crore / } -\$144,000 \text{ USD})}$$

#### Boardroom Interpretation of $SV = -₹1.20 \text{ Crore}$:
* The package is running **₹1.20 Crore behind scheduled delivery**.
* What does a negative currency variance mean for schedule?
  $$\text{Plinth Deficit} = \frac{-\text{₹}1,20,00,000}{\text{₹}20,00,000 \text{ baseline rate}} = -6 \text{ Plinths}$$
* The site is missing 6 complete inverter foundations. At the baseline burn rate of 5 plinths/month:
  $$\text{Time Slippage} = \frac{6 \text{ Plinths}}{5 \text{ Plinths/month}} = 1.2 \text{ Months behind schedule!}$$

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                 DATA DATE CROSS-SECTION (VISUALIZING THE VARIANCES)                   │
├────────────────────────────────────────────────────────────────────────────────────────┤
│   Value (₹ Cr)                                                                         │
│                                                                                        │
│     6.00 Cr ───┬───────────────────────────────► PV (Baseline Plan)                     │
│                │                                                                       │
│                │ ◄── Schedule Variance (SV) = EV - PV = -₹1.20 Cr                      │
│                │     (The Work Slippage Gap)                                           │
│     5.50 Cr ───┼───────────► AC (Actual Cash Spent)                                    │
│                │                                                                       │
│                │ ◄── Cost Variance (CV) = EV - AC = -₹0.70 Cr                          │
│                │     (The Cash Burn Overrun Gap)                                       │
│     4.80 Cr ───┴───────────────────────────────► EV (Physical Work Achieved)           │
│                                                                                        │
│               ─────────────────▲────────────────                                       │
│                           DATA DATE (Month 6)                                          │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Performance Indices: CPI and SPI

While variances give absolute money, indices normalize performance into efficiency ratios. They answer: *"What return are we getting for our inputs?"*

```text
                     EVM EFFICIENCY INDEX AXIOM:
                     ---------------------------
              Index = Earned Value (EV) / Comparison Metric
              
              • Cost Performance:     CPI = EV / AC  (Bang for every Rupee spent)
              • Schedule Performance: SPI = EV / PV  (Velocity against schedule plan)

              BENCHMARK:
              Index > 1.00  ──► Favorable (Highly Efficient)
              Index = 1.00  ──► Exactly on Baseline Target
              Index < 1.00  ──► Unfavorable (Deficient / Inefficient)
```

### 5.1 Cost Performance Index ($CPI$)
$$CPI = \frac{EV}{AC} = \frac{\text{₹}4,80,00,000}{\text{₹}5,50,00,000} \approx \mathbf{0.8727 \quad (0.873)}$$

#### What $CPI = 0.873$ Means to the Project Director:
* For every **₹1.00** that the company spends on this inverter package, it receives only **₹0.873** (87.3 paise) of real physical asset value.
* We are destroying **12.7 paise per Rupee** spent.
* If converted to USD: For every **\$1.00** spent, we earn only **\$0.87** of installed work.

---

### 5.2 Schedule Performance Index ($SPI$)
$$SPI = \frac{EV}{PV} = \frac{\text{₹}4,80,00,000}{\text{₹}6,00,00,000} = \mathbf{0.8000 \quad (0.80)}$$

#### What $SPI = 0.800$ Means to the Project Director:
* The project team is executing work at only **80% of the planned velocity**.
* For every 10 hours scheduled, the team only delivers 8 hours worth of completed output.
* 20% of planned progress is evaporating each month due to site logistical friction.

---

### 5.3 The Executive 4-Quadrant Matrix

Plotting $CPI$ versus $SPI$ places your project package into one of four critical operational quadrants:

```text
   CPI (Cost Efficiency)
        ▲
        │
   1.20 │     QUADRANT IV: PENNY-PINCHING       │       QUADRANT I: DREAM PROJECT
        │   • Under Budget (CPI > 1.0)          │   • Under Budget (CPI > 1.0)
        │   • Behind Schedule (SPI < 1.0)       │   • Ahead of Schedule (SPI > 1.0)
   1.00 ┼───────────────────────────────────────┼───────────────────────────────────────
        │                                       │
        │   QUADRANT III: CRISIS (OUR PROJECT)  │   QUADRANT II: BUYING SPEED
        │   • Over Budget (CPI = 0.873)         │   • Over Budget (CPI < 1.0)
        │   • Behind Schedule (SPI = 0.800)     │   • Ahead of Schedule (SPI > 1.0)
   0.80 │   * Immediate Intervention Required!  │   • Massive Overtime / Air Freight
        │                                       │
        └───────────────────────────────────────┴───────────────────────────────────────►
       0.70                0.90                1.00                1.10             SPI
                                                                           (Schedule Velocity)
```

* **Quadrant I ($CPI > 1, SPI > 1$):** Peak industrial excellence. Contractor earns early completion bonus.
* **Quadrant II ($CPI < 1, SPI > 1$):** Deliberate fast-tracking. Spending extra capex (night shifts, two cranes) to guarantee milestone achievement and avoid LDs (Liquidated Damages).
* **Quadrant III ($CPI < 1, SPI < 1$):** **The Danger Zone**. Surya Inverter Package is here. Double jeopardy—wasting capital while slipping toward contractual default.
* **Quadrant IV ($CPI > 1, SPI < 1$):** False economy. Spending less money simply because work is stalled (waiting for engineering drawings or permits).

---

## 6. The Geometry of the S-Curve & The "Banana Envelope"

Project expenditure and physical progress do not follow a straight line. They form an **S-Curve**:

```text
 Cumulative
 Progress / Cost
   100% ┼─────────────────────────────────────────────────────────────╭─── BAC (100%)
        │                                                         ╭──╯
        │                                                     ╭──╯ ◄── Commissioning & Handover
        │                                                 ╭──╯         (Slow finish / Snagging)
        │                                             ╭──╯
        │                                         ╭──╯ ◄── Peak Execution
        │                                     ╭──╯         (Maximum daily concrete & rebar)
        │                                 ╭──╯
        │                             ╭──╯
        │                         ╭──╯
        │                     ╭──╯
        │                 ╭──╯ ◄── Mobilization & Site Setup
        │             ╭──╯         (Slow initial ramp-up)
     0% ┼─────────╭──╯───────────────────────────────────────────────────────────────
        0         1     2     3     4     5     6     7     8     9    10 Months
```

### 6.1 The Early Dates vs Late Dates S-Curve (The Banana Envelope)
In Primavera P6, when you calculate CPM float, every activity has an **Early Start/Early Finish** and a **Late Start/Late Finish**.
* **Early S-Curve ($PV_{Early}$):** Cumulative cost curve if every single activity starts on its $ES$. This represents the fastest possible financial burn.
* **Late S-Curve ($PV_{Late}$):** Cumulative cost curve if every single activity is delayed to its $LS$. This represents the slowest possible progress without moving the contract completion date.

The area bounded between these two curves is called the **"Banana Curve Envelope"**:

```text
 Cumulative
 Cost (₹ Cr)
  10.00 ┼─────────────────────────────────────────────────────────────╭─── BAC (₹10 Cr)
        │                                                   ╭─────────╯
        │                                         ╭────────╯    .·´
        │                               ╭────────╯          .·´  ◄── Early S-Curve (ES)
        │                     ╭────────╯                .·´
        │           ╭────────╯                      .·´
   5.00 │ ╭────────╯       BANANA ENVELOPE      .·´
        │ │                                 .·´
        │ │                             .·´
        │ │                         .·´  ◄── Late S-Curve (LS)
        │ │                     .·´
        │ │                 .·´
        │ │             .·´
   0.00 ┼─┴─────────.·´──────────────────────────────────────────────────────────────
        0         2         4         6         8         10        12   Timeline (Months)
```

### 6.2 The Industrial Rule of the Banana Envelope
1. **As long as $EV$ stays inside the Banana Envelope:**  
   The project has positive total float. Even if $EV$ is below the Early curve, the project will still hit the deadline provided activities do not slip past their Late dates.
2. **The Moment $EV$ Drops Below the Late S-Curve ($EV < PV_{Late}$):**  
   **The project has breached its Critical Path!** Negative Float is now generated. The contractual Commercial Operation Date (COD) is officially broken unless critical path crashing is applied immediately.

---

## 7. Forecasting Arithmetic: EAC, ETC, and VAC

Executive leadership does not just ask *"Where are we today?"*  
The Board of Directors asks: **"How much will this project ultimately cost when we finish ($EAC$), and how much over budget will we be ($VAC$)?"**

There are four industry-standard mathematical formulas to calculate **Estimate at Completion ($EAC$)**, each based on different project risk assumptions:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        THE 4 STANDARD EAC FORECASTING METHODS                          │
├─────────┬───────────────────────────────┬──────────────────────────────────────────────┤
│ Method  │ Formula                       │ Underlying Management Assumption             │
├─────────┼───────────────────────────────┼──────────────────────────────────────────────┤
│ Case 1  │ EAC = BAC / CPI               │ Current cost efficiency continues to the end │
├─────────┼───────────────────────────────┼──────────────────────────────────────────────┤
│ Case 2  │ EAC = AC + (BAC - EV)         │ Current overrun was an atypical one-off blip │
├─────────┼───────────────────────────────┼──────────────────────────────────────────────┤
│ Case 3  │ EAC = AC + [(BAC - EV) /      │ Both cost efficiency AND schedule velocity   │
│         │            (CPI × SPI)]       │ will compound remaining work (Worst Case)    │
├─────────┼───────────────────────────────┼──────────────────────────────────────────────┤
│ Case 4  │ EAC = AC + ETC_bottom_up      │ Comprehensive re-estimate of remaining work  │
└─────────┴───────────────────────────────┴──────────────────────────────────────────────┘
```

Let us calculate each case for our Surya Solar Inverter Package:
* $BAC = \text{₹}10,00,00,000$ (₹10.00 Cr)
* $EV = \text{₹}4,80,00,000$ (₹4.80 Cr)
* $AC = \text{₹}5,50,00,000$ (₹5.50 Cr)
* Remaining Budgeted Work $(BAC - EV) = \text{₹}10.00\text{ Cr} - \text{₹}4.80\text{ Cr} = \text{₹}5,20,00,000$ (₹5.20 Cr)
* $CPI = 0.8727$
* $SPI = 0.8000$

---

### 7.1 Case 1: Typical Future Performance ($EAC = BAC / CPI$)
*Assumption:* The site team will continue to experience the exact same ground rock excavation challenges and concrete inefficiencies for the remaining 26 plinths ($CPI$ remains 0.8727).

$$EAC = \frac{BAC}{CPI} = \frac{\text{₹}10,00,00,000}{0.872727} = \mathbf{\text{₹}11,45,83,333 \quad (₹11.46 \text{ Crore / } \$1.375\text{M USD})}$$

#### Estimate to Complete ($ETC$):
$$ETC = EAC - AC = \text{₹}11.4583\text{ Cr} - \text{₹}5.5000\text{ Cr} = \mathbf{\text{₹}5,95,83,333 \quad (₹5.96 \text{ Crore})}$$

#### Variance at Completion ($VAC$):
$$VAC = BAC - EAC = \text{₹}10,00,00,000 - \text{₹}11,45,83,333 = \mathbf{-\text{₹}1,45,83,333 \quad (-₹1.46 \text{ Crore Overrun})}$$

*Takeaway:* The contractor will breach the contract by **₹1.46 Crore (14.6% budget overrun)**.

---

### 7.2 Case 2: Atypical Performance ($EAC = AC + [BAC - EV]$)
*Assumption:* The ₹70 Lakh overrun in Months 1–6 was caused by an unprecedented monsoon flood and rock strike. For the remaining 26 plinths, geology improves, and the contractor will execute strictly according to the baseline budget ($CPI_{future} = 1.00$).

$$EAC = AC + (BAC - EV)$$
$$EAC = \text{₹}5,50,00,000 + (\text{₹}10,00,00,000 - \text{₹}4,80,00,000)$$
$$EAC = \text{₹}5,50,00,000 + \text{₹}5,20,00,000 = \mathbf{\text{₹}10,70,00,000 \quad (₹10.70 \text{ Crore})}$$

#### Variance at Completion ($VAC$):
$$VAC = BAC - EAC = \text{₹}10.00\text{ Cr} - \text{₹}10.70\text{ Cr} = \mathbf{-\text{₹}70,00,000 \quad (-₹70 \text{ Lakhs Overrun})}$$

*Takeaway:* The historical overrun of ₹70 Lakhs is "locked in", but no further bleed occurs.

---

### 7.3 Case 3: Combined Cost and Schedule Factor ($EAC = AC + \frac{BAC - EV}{CPI \times SPI}$)
*Assumption (The Construction Reality Case):* When a contractor is behind schedule ($SPI < 1.0$), they must pay liquidated damages or hire extra shifts, additional batching plants, and night-shift tower lights to catch up. Therefore, poor schedule performance *drags down* cost efficiency.

$$\text{Combined Factor} = CPI \times SPI = 0.872727 \times 0.8000 = 0.69818$$

$$ETC = \frac{BAC - EV}{CPI \times SPI} = \frac{\text{₹}5,20,00,000}{0.69818} = \text{₹}7,44,79,167 \quad (₹7.45 \text{ Crore})$$

$$EAC = AC + ETC = \text{₹}5,50,00,000 + \text{₹}7,44,79,167 = \mathbf{\text{₹}12,94,79,167 \quad (₹12.95 \text{ Crore / } \$1.55\text{M USD})}$$

#### Variance at Completion ($VAC$):
$$VAC = BAC - EAC = \text{₹}10.00\text{ Cr} - \text{₹}12.95\text{ Cr} = \mathbf{-\text{₹}2,94,79,167 \quad (-₹2.95 \text{ Crore Overrun})}$$

*Takeaway (Worst-case projection):* If the schedule delay forces emergency crashing, the final cost will blow out by **nearly ₹3.00 Crore (29.5% overrun)**!

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                      EAC FORECAST COMPARISON FOR BOARD REPORT                          │
├────────────────────────────────┬──────────────────┬──────────────────┬─────────────────┤
│ Forecast Model                 │ Projected EAC    │ Projected VAC    │ Expected Finish │
├────────────────────────────────┼──────────────────┼──────────────────┼─────────────────┤
│ Baseline Contract (Target)     │ ₹10.00 Crore     │ ₹0.00            │ Month 10.0      │
│ Case 2: Atypical (Optimistic)  │ ₹10.70 Crore     │ -₹0.70 Crore     │ Month 11.5      │
│ Case 1: Typical CPI (Realistic)│ ₹11.46 Crore     │ -₹1.46 Crore     │ Month 12.5      │
│ Case 3: CPI × SPI (Pessimistic)│ ₹12.95 Crore     │ -₹2.95 Crore     │ Month 13.8      │
└────────────────────────────────┴──────────────────┴──────────────────┴─────────────────┘
```

---

## 8. To-Complete Performance Index ($TCPI$): The Feasibility Test

$TCPI$ answers the hardest question of all:  
*"At what cost efficiency must our site team perform for all remaining work to finish within the target budget?"*

```text
                        TCPI FORMULA AXIOM:
                        -------------------
                 Remaining Work to Accomplish     BAC - EV
        TCPI = ──────────────────────────────── = ────────
                 Remaining Available Capital      Target - AC
```

### 8.1 $TCPI$ Based on Original Budget ($BAC$)
Can the team still finish within the contract budget of ₹10.00 Crore?

$$TCPI_{BAC} = \frac{BAC - EV}{BAC - AC} = \frac{\text{₹}10,00,00,000 - \text{₹}4,80,00,000}{\text{₹}10,00,00,000 - \text{₹}5,50,00,000} = \frac{\text{₹}5,20,00,000}{\text{₹}4,50,00,000} \approx \mathbf{1.1556 \quad (1.16)}$$

#### Interpretation of $TCPI_{BAC} = 1.16$:
* To meet the original ₹10.00 Crore budget, every remaining ₹1.00 spent must generate **₹1.16 worth of physical output**.
* The team was running at $CPI = 0.873$. Demanding a jump from $0.873 \to 1.16$ represents a **32.8% sudden efficiency improvement**.
* In civil contracting, this is rarely possible without drastic engineering value-engineering or scope reduction.

---

### 8.2 $TCPI$ Based on Revised Forecast ($EAC_{typical} = ₹11.46 \text{ Cr}$)
If management accepts Case 1's revised budget ($EAC = ₹11.46 \text{ Cr}$), what efficiency is required?

$$TCPI_{EAC} = \frac{BAC - EV}{EAC - AC} = \frac{\text{₹}5,20,00,000}{\text{₹}11,45,83,333 - \text{₹}5,50,00,000} = \frac{\text{₹}5,20,00,000}{\text{₹}5,95,83,333} \approx \mathbf{0.8727 \quad (0.87)}$$

#### The Mathematical Proof:
$$TCPI_{EAC} = CPI_{current} = 0.873$$
This proves that if the project maintains its current pace of performance, it will precisely achieve the $EAC_{typical}$ forecast.

### 8.3 The "Point of No Return" Rule ($TCPI > 1.20$)
```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        TCPI REALITY CHECK BENCHMARK RULE                               │
├───────────────────┬────────────────────────────────────────────────────────────────────┤
│ TCPI < 1.00       │ Safe / Realistic. Project has budget buffer to absorb friction.    │
├───────────────────┼────────────────────────────────────────────────────────────────────┤
│ 1.00 ≤ TCPI ≤ 1.10│ Challenging but achievable with tight site supervision.            │
├───────────────────┼────────────────────────────────────────────────────────────────────┤
│ 1.11 ≤ TCPI ≤ 1.20│ Severe risk. Requires management restructuring and productivity.   │
├───────────────────┼────────────────────────────────────────────────────────────────────┤
│ TCPI > 1.20       │ STATISTICALLY IMPOSSIBLE FANTASY.                                  │
│                   │ Historical empirical research across 700+ mega-projects proves     │
│                   │ that no major construction contract has ever recovered to BAC once │
│                   │ TCPI exceeds 1.20. Management MUST issue a formal re-baseline!     │
└───────────────────┴────────────────────────────────────────────────────────────────────┘
```

---

## 9. The Fatal Traps of EVM: What Textbooks Never Tell You

A senior project controls specialist knows that applying EVM formulas blindly without critical schedule analysis can mislead executive leadership.

### Trap 1: The End-Game Fallacy ($SV \to 0$ and $SPI \to 1.0$ at Project Close)
Look at the formula for Schedule Variance:
$$SV = EV - PV$$
At the very end of any project, all scope is eventually completed:
$$EV = BAC$$
And since the baseline time has long passed:
$$PV = BAC$$
Therefore:
$$SV = BAC - BAC = 0.00$$
$$SPI = \frac{BAC}{BAC} = 1.000$$

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                           THE ABSURD END-GAME EVM ANOMALY                              │
├────────────────────────────────────────────────────────────────────────────────────────┤
│   A 10-month solar project finishes 8 MONTHS LATE (Takes 18 months).                   │
│                                                                                        │
│   At Month 18:                                                                         │
│   • EV = ₹10.00 Crore (All 50 plinths built)                                           │
│   • PV = ₹10.00 Crore (Baseline was ₹10.00 Cr)                                         │
│                                                                                        │
│   The Formula reports:                                                                 │
│   • SV  = ₹10.00 Cr - ₹10.00 Cr = ₹0.00 (Schedule Variance = Zero!)                    │
│   • SPI = ₹10.00 Cr / ₹10.00 Cr = 1.00 (Schedule Performance = Perfect!)              │
│                                                                                        │
│   The CEO asks: "Why does the EVM report say schedule performance is 1.00              │
│   when the client is suing us for 8 months of delay liquidated damages?!"              │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

#### The Solution: Earned Schedule ($ES$)
To fix this fatal flaw, modern project controls uses **Earned Schedule ($ES$)**, pioneered by Lipke:
Instead of measuring schedule variance in dollars ($SV$), measure it in **Time Units ($SV_t$)**:
* Look at the $PV$ curve to find the exact point in time when planned value equaled our current $EV$ (₹4.80 Cr).
* ₹4.80 Cr of work was planned to be reached at **Month 4.8**.
* Therefore, $\text{Earned Schedule } (ES) = 4.8 \text{ Months}$.
* Actual Time elapsed $(AT) = 6.0 \text{ Months}$.
* **Schedule Variance in Time ($SV_t$):**
  $$SV_t = ES - AT = 4.8 - 6.0 = \mathbf{-1.2 \text{ Months}}$$
* **Time-based Schedule Performance Index ($SPI_t$):**
  $$SPI_t = \frac{ES}{AT} = \frac{4.8}{6.0} = \mathbf{0.80}$$

Unlike standard $SPI$, $SPI_t$ never collapses back to 1.0 at project close if the project was late!

---

### Trap 2: The Critical Path Blind Spot
EVM aggregates every single activity into a single monetary sum. It does not know graph theory!

* **Disaster Scenario:**
  * Activity 1: Boundary wall fencing & tree planting (Non-critical, 90 days total float, budget ₹2.0 Cr).
  * Activity 2: 132kV Main Substation Power Transformer Foundation (On the Critical Path, budget ₹2.0 Cr).
  * The site contractor completes 100% of the fencing ahead of time ($EV = \text{₹}2.0\text{ Cr}$).
  * The contractor did zero work on the substation foundation ($EV = \text{₹}0.0\text{ Cr}$).
* **EVM Result:**
  * $EV = \text{₹}2.0\text{ Cr}$, $PV = \text{₹}2.0\text{ Cr} \implies SPI = 1.00$.
  * Dashboard shows **GREEN ("On Track")**!
  * **Ground Reality:** The project completion date is delayed by 3 months because the critical path is completely frozen!

> [!WARNING]
> **Primavera P6 Professional Mandate:**  
> Never analyze EVM in isolation from CPM Total Float.  
> Always filter your EVM reports by **WBS Critical Path Activities only**!

---

### Trap 3: Primavera P6 Percent Complete Types & Fake EVM
Primavera P6 provides three options for an activity's `% Complete Type`:
1. **Duration % Complete:**  
   $$\text{Duration \%} = \frac{\text{Original Duration} - \text{Remaining Duration}}{\text{Original Duration}}$$
   *DANGER:* If an engineer updates the schedule simply by letting the clock tick, P6 automatically increments Duration % Complete. This generates **Phantom Earned Value** without a single bag of cement being poured on site!
2. **Units % Complete:**  
   Driven by labor and machine hours spent. If labor is inefficient and works double shifts, Units % jumps up, inflating EV falsely.
3. **Physical % Complete (THE ONLY VALID EVM METHOD):**  
   Independent of time and hours. Site quantity surveyor enters the verified physical progress (e.g., "24 plinths out of 50 = 48.0%").

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                   HOW P6 PERCENT COMPLETE TYPES ALTER EVM CALCULATION                  │
├───────────────────────┬──────────────────────────────────┬─────────────────────────────┤
│ P6 % Complete Type    │ How P6 Calculates EV             │ Industrial Validity for EVM │
├───────────────────────┼──────────────────────────────────┼─────────────────────────────┤
│ Physical % Complete   │ EV = BAC × Physical % Complete   │ HIGH (Gold Standard)        │
│ Duration % Complete   │ EV = BAC × Duration % Complete   │ FLAWED (Clock-driven)       │
│ Units % Complete      │ EV = BAC × Labor Units %         │ MODERATE (Labor-biased)     │
└───────────────────────┴──────────────────────────────────┴─────────────────────────────┘
```

---

## 10. Primavera P6 Step-by-Step Implementation Guide

To implement rigorous Earned Value tracking in **Oracle Primavera P6 Enterprise Project Portfolio Management (EPPM / PPM Professional)**:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        PRIMAVERA P6 EVM CONFIGURATION WORKFLOW                         │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                        │
│   Step 1: Assign Cost & BAC ────► Step 2: Establish Target Baseline (B1)               │
│                                                   │                                    │
│                                                   ▼                                    │
│   Step 4: Update Actuals & DD  ◄──── Step 3: Configure EV Calculations in Preferences │
│             │                                                                          │
│             ▼                                                                          │
│   Step 5: Run Schedule (F9) ───► Step 6: Layout Custom Columns (PV, EV, AC, CPI, SPI) │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### Step 1: Cost-Load Activities
* Assign Resource or Expense to each activity.
* For Surya Inverter Foundations: Activity `ACT-1020` has an assigned Budgeted Total Cost = `₹10,00,00,000`.

### Step 2: Freeze the Project Baseline
1. Navigate to: `Project` $\to$ `Maintain Baselines...`
2. Click `Add` $\to$ `Save a copy of the current project as a new baseline`.
3. Name it: `Surya-100MW-Rev0-Approved-Contract-Baseline`.
4. Navigate to: `Project` $\to$ `Assign Baselines...`
5. Set `Project Baseline` = `Surya-100MW-Rev0` and `User Baseline 1 (B1)` = `Surya-100MW-Rev0`.

### Step 3: Set Project EV Preferences
1. Open the `Projects` window (`Ctrl+W`).
2. Select the `Calculations` tab in project details.
3. Under **Earned Value calculation**, configure:
   * **Baseline for earned value calculations:** `Project Baseline`.
   * **Technique for computing activity percent complete:** Select `Physical Percent Complete`.
   * **Earned value cost calculation:** `Budgeted values with current dates`.

### Step 4: Progress Update at Data Date
1. Set Activity `% Complete Type` to `Physical`.
2. Move the **Data Date** to `30-Jun-2026` (End of Month 6).
3. Set `Physical % Complete` = `48.0%`.
4. Enter Actual Cost in the `Actual Total Cost` field = `₹5,50,00,000`.
5. Press **`F9` (Schedule)** and click `Schedule`.

### Step 5: Configure Columns in P6 Gantt View
Right-click on table headers $\to$ `Columns` $\to$ Add the following fields:
* `Planned Value Cost` (Displays $PV$)
* `Earned Value Cost` (Displays $EV$)
* `Actual Total Cost` (Displays $AC$)
* `Cost Variance` (Displays $CV$)
* `Schedule Variance` (Displays $SV$)
* `Cost Performance Index` (Displays $CPI$)
* `Schedule Performance Index` (Displays $SPI$)
* `Estimate At Completion Cost` (Displays $EAC$)
* `Variance At Completion` (Displays $VAC$)
* `To Complete Performance Index` (Displays $TCPI$)

---

## 11. Executive Translation Cheat Sheet

Use this reference table during project status meetings and boardroom presentations to translate mathematical symbols into immediate commercial action:

```text
┌────────┬─────────────────────────┬──────────────────────────────┬──────────────────────────────────────────┐
│ Symbol │ Formal Name             │ Formula                      │ Commercial Executive Meaning             │
├────────┼─────────────────────────┼──────────────────────────────┼──────────────────────────────────────────┤
│ BAC    │ Budget at Completion    │ Total Contract Cost Baseline │ Total approved budget for the package.   │
│ PV     │ Planned Value (BCWS)    │ Planned % × BAC              │ The value of work promised by today.     │
│ EV     │ Earned Value (BCWP)     │ Physical % × BAC             │ Real physical value created on site.     │
│ AC     │ Actual Cost (ACWP)      │ Realized Invoices / Costs    │ Cash disbursed from project treasury.    │
├────────┼─────────────────────────┼──────────────────────────────┼──────────────────────────────────────────┤
│ CV     │ Cost Variance           │ EV - AC                      │ Negative = Over budget. Money burnt.     │
│ SV     │ Schedule Variance       │ EV - PV                      │ Negative = Delayed. Deliverables missed. │
├────────┼─────────────────────────┼──────────────────────────────┼──────────────────────────────────────────┤
│ CPI    │ Cost Performance Index  │ EV / AC                      │ Value earned per ₹1.00 spent.            │
│        │                         │                              │ < 1.00 means financial loss.             │
├────────┼─────────────────────────┼──────────────────────────────┼──────────────────────────────────────────┤
│ SPI    │ Schedule Performance    │ EV / PV                      │ Progress velocity ratio.                 │
│        │ Index                   │                              │ < 1.00 means slower than baseline plan.  │
├────────┼─────────────────────────┼──────────────────────────────┼──────────────────────────────────────────┤
│ EAC    │ Estimate at Completion  │ BAC / CPI (Typical)          │ Forecasted final package check to write. │
├────────┼─────────────────────────┼──────────────────────────────┼──────────────────────────────────────────┤
│ VAC    │ Variance at Completion  │ BAC - EAC                    │ Final projected cost overrun/underrun.   │
├────────┼─────────────────────────┼──────────────────────────────┼──────────────────────────────────────────┤
│ TCPI   │ To-Complete Performance │ (BAC - EV) / (BAC - AC)      │ Efficiency required on remaining work.   │
│        │ Index                   │                              │ > 1.20 means recovery is impossible.     │
└────────┴─────────────────────────┴──────────────────────────────┴──────────────────────────────────────────┘
```

---

## 12. Summary Checklist for Senior Project Engineers

Before signing off on an EVM report, verify:
* [ ] Is Earned Value derived from **Physical % Complete** rather than Duration % Complete?
* [ ] Has Actual Cost ($AC$) been reconciled with ERP finance ledgers including unpaid accrued liabilities?
* [ ] Is the $SPI$ checked against the **CPM Critical Path Total Float** to prevent false green status?
* [ ] Has Earned Schedule ($ES_t$) been checked to confirm schedule variance in months?
* [ ] If $TCPI > 1.20$, has an executive variance warning and baseline change request (BCR) been prepared?
