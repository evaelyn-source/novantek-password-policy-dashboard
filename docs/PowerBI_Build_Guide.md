# Global Password Policy — Multi-Region Control Testing Dashboard
### Power BI Build Guide

**Dataset:** `Novantek_PasswordPolicy_ControlTesting.xlsx`
**Scenario:** Novantek Global (fictional device manufacturer, modeled on a real multinational's regional footprint) runs one legacy global password policy across four regions. This dashboard tests that policy against each region's actual framework and turns the technical gaps into a decision a non-technical exec can act on: **standardize on one strict global policy, or maintain region-specific policies?**

---

## 1. Data model

Import all four sheets from the workbook as separate Power BI tables, then build a star schema:

```
Dim_Region ----\
Dim_Framework ---> Fact_ControlTest <--- Dim_Control
```

| Relationship | Cardinality | Direction |
|---|---|---|
| Dim_Region[RegionID] → Fact_ControlTest[RegionID] | 1:many | Single |
| Dim_Framework[FrameworkID] → Fact_ControlTest[FrameworkID] | 1:many | Single |
| Dim_Control[ControlID] → Fact_ControlTest[ControlID] | 1:many | Single |

Set `ComplianceStatus` as a text column with a sort-order helper if you want Fail/Partial/Pass to sort in that order on visuals: add a calculated column `StatusSortOrder = SWITCH(Fact_ControlTest[ComplianceStatus], "Fail", 1, "Partial", 2, "Pass", 3)` and sort the ComplianceStatus column by it (Column tools → Sort by Column).

---

## 2. Core DAX measures

```dax
Total Tests = COUNTROWS(Fact_ControlTest)

Fails = CALCULATE([Total Tests], Fact_ControlTest[ComplianceStatus] = "Fail")

Compliance Rate =
DIVIDE(
    CALCULATE([Total Tests], Fact_ControlTest[ComplianceStatus] = "Pass"),
    [Total Tests]
)

Total Remediation Cost = SUM(Fact_ControlTest[RemediationCostUSD])

Total Remediation Effort (Days) = SUM(Fact_ControlTest[RemediationEffortDays])

Users Exposed (Open Gaps) =
CALCULATE(
    SUM(Fact_ControlTest[UsersImpactedEstimate]),
    Fact_ControlTest[ComplianceStatus] IN {"Fail","Partial"}
)

High Risk Open Gaps =
CALCULATE(
    [Total Tests],
    Fact_ControlTest[ComplianceStatus] IN {"Fail","Partial"},
    Fact_ControlTest[RiskRating] = "High"
)
```

**Remediation trend (cycle-over-cycle), the measure that carries the business story:**

```dax
Compliance Rate by Cycle =
CALCULATE(
    [Compliance Rate],
    ALLEXCEPT(Fact_ControlTest, Fact_ControlTest[TestCycle], Fact_ControlTest[RegionID], Fact_ControlTest[ControlID])
)

Gaps Closed Since Baseline =
VAR BaselineFails =
    CALCULATE([Fails], Fact_ControlTest[TestCycle] = "Q1 2026 Baseline")
VAR RetestFails =
    CALCULATE([Fails], Fact_ControlTest[TestCycle] = "Q2 2026 Retest")
RETURN
    BaselineFails - RetestFails
```

---

## 3. Page layout (3 pages — this is what makes it "for non-technical stakeholders")

### Page 1 — Executive Summary (the decision page)
This is the page a director opens first and the only one some of them will read. No jargon, no raw settings.
- **KPI cards across the top:** Compliance Rate (current cycle), Total Remediation Cost (open gaps), Users Exposed, High Risk Open Gaps
- **Trend visual:** Compliance Rate by Cycle, Q1 → Q2, split by Region (clustered column or line) — this is the "are we getting better" chart
- **Matrix/heatmap:** Region (rows) × Control (columns), values = ComplianceStatus, conditional-formatted red/yellow/green — a non-technical viewer reads this in five seconds
- **One text box, plain language:** the decision framing — e.g., "A single global password policy fails 3 of 4 regional standards. Standardizing on the strictest common requirement (Korea's) would close all regional gaps for an estimated $X and Y days of work, vs. maintaining four separate regional policies."

### Page 2 — Regional Detail
- Slicer: Region
- Table: Control, Required Value, Configured Value, Status, Gap Description, Recommended Action, Owner — filtered to the selected region
- Card: Total Remediation Cost for this region

### Page 3 — Remediation Tracker
- Slicer: TestCycle
- Bar chart: Remediation Cost by Control, colored by Status
- Table: only open (Fail/Partial) items, sorted by RiskRating, with Owner and RecommendedAction — this becomes the literal punch list a PM would run from

---

## 4. The business-decision framing to put in your portfolio README / interview talking points

The technical finding: one global password policy cannot satisfy Korea (K-ISMS-P), the EU (ISO 27001 + NIS2), the US (NIST 800-63B), and Brazil (LGPD) simultaneously — some regions need stricter controls than the global default, and in one case (forced 90-day rotation) the global policy is *stricter than current best practice* and actively contradicts NIST guidance while adding user friction.

The business translation: this isn't "buy more security tools" — it's a governance decision between two real options, each with a cost and a trade-off:
1. **One strict global baseline** (adopt the toughest regional requirement everywhere) — simpler to govern, higher upfront cost, some regions over-controlled.
2. **Region-specific policies** — cheaper and right-sized, but harder to audit and more moving parts for IT to maintain.

That framing — cost, risk, and a real trade-off, not just a red/green checklist — is what separates a portfolio piece that looks like a compliance report from one that reads like a business case.

---

## 5. Notes on the data
All test results, dates, costs, and Novantek Global itself are fictional, built for a portfolio demo. The frameworks (K-ISMS-P, ISO/IEC 27001, NIS2, NIST SP 800-63B, LGPD) and the regions they apply to are real and sourced publicly — see the `SourceCitation` column in `Dim_Framework`. General framework positions (e.g., NIST's current stance against forced periodic rotation) reflect real, publicly documented guidance; specific pass/fail thresholds in this dataset are illustrative, not verbatim regulatory text.
