# Omnichannel Unit Economics & Subscription Retention Diagnostics
**Adapting IIM Ahmedabad’s *Arvind Fashions* Strategic Framework to Multi-Channel Commerce (KALPA)**

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue.svg)](https://www.python.org/)
[![Pandas](https://img.shields.io/badge/Library-Pandas%20%7C%20NumPy-orange.svg)](https://pandas.pydata.org/)
[![Domain](https://img.shields.io/badge/Domain-Omnichannel%20Retail%20%26%20SaaS%2FMembership-success.svg)](#)

---

## 📌 Executive Summary & Project Objective
Many retail and subscription analytics projects evaluate datasets in a vacuum, calculating surface-level metrics without a coherent management framework. 

This project bridges **MBA strategic case analysis** and **rigorous relational data science**. First, it deconstructs the **IIM Ahmedabad case study *Arvind Fashions: Omnichannel Retailing in Digital India* (IIMA/IS0147)** to extract executive decision frameworks, KPI hierarchies, and omnichannel unit economics. Second, it translates and adapts those analytical methods to **KALPA’s 9-table relational dataset** (developed at IIT Gandhinagar), diagnosing channel profitability, omnichannel customer value multiples, and subscription renewal leakage.

### Key Analytical Contributions
1. **P&L Revenue Recognition Hygiene:** Rectified top-line reconciliation discrepancies by isolating `COMPLETED` orders from cancelled/non-completed transactions across the full contribution waterfall.
2. **Segment-Adjusted Omnichannel Benchmarking:** Tested Arvind Fashions' benchmark (*omnichannel shoppers generate $2.5\times$ the revenue of single-channel shoppers*) on KALPA's customer base across `STORE`, `APP`, and `WEB`, explicitly controlling for B2B (`Business` segment) bulk-purchasing skew.
3. **Right-Censoring Correction in Cohort Retention:** Identified and removed open renewal opportunities whose deadlines fell after the dataset's maximum event timestamp (`renewal_deadline > max(event_time)`), correcting a **~5 percentage-point understatement** in the 2026 first-renewal rate (**~55% true closed rate vs. ~50% naive rate**).
4. **Involuntary vs. Voluntary Churn Forensics:** Built an event-level diagnostic pipeline separating closed renewal losses into **Passive Disengagement (*"No payment attempt"*)** and **Involuntary Technical Churn (*"Attempted but failed"*)**, mapping failure `detail_code` logs and `REMINDER_SENT` delivery failures to automated recovery workflows.

---

## 🏛️ Part 1: Strategic Foundation — The Arvind Fashions Case (IIMA/IS0147)
In the IIM Ahmedabad case study, Arvind Fashions Ltd. (AFL)—operating 1,300+ Exclusive Brand Outlets (EBOs) and 5,000+ Multi-Brand Outlets (MBOs) with FY23 revenues of INR 44,210 million—faced margin erosion from early-lifecycle discounting on third-party e-commerce portals (Myntra, Flipkart, Amazon) and zero end-consumer visibility across its ~30% wholesale distribution channel.

### Core Case Pillars Adapted to KALPA
* **"One Customer, One Inventory" & Omnichannel Multiplier:** AFL discovered that customers shopping both online and offline generated **$2.5\times$ the sales** of pure offline or pure online buyers, maximizing LTV and lowering blended Customer Acquisition Cost (CAC).
* **Input vs. Output KPI Hierarchy:** AFL’s data science team layered leading operational input indicators (e.g., discount levers, payment failure alerts, store fulfilment rates) beneath executive output KRAs (Channel Margin, Net Contribution, Retention).
* ** Solving "Cohorts Within Cohorts":** AFL’s leadership noted an ongoing struggle with multi-dimensional cohort retention analysis—a gap directly addressed and solved in our KALPA membership renewal pipeline.

---

## 🗄️ Part 2: KALPA Data Architecture & Schema
Unlike Arvind Fashions (an apparel franchise conglomerate operating across wholesale, marketplaces, and D2C), **KALPA** operates a **direct multi-channel retail and recurring membership model** across physical stores (`STORE`) and digital touchpoints (`APP`, `WEB`).

The analytical pipeline ingests and validates **9 relational CSV tables**:

| Table Name | Primary Key | Foreign Keys / Joins | Core Attributes & Analytical Purpose |
| :--- | :--- | :--- | :--- |
| `orders.csv` | `order_id` | `customer_id`, `channel_id`, `period_id` | `order_date`, `status` (`COMPLETED`, etc.), `gross_amount`, `discount_amount`, `refund_amount` |
| `order_costs.csv` | `order_id` | `order_id` (1:1 with `orders`) | Variable unit economics: `product_cost` (COGS), `fulfilment_cost`, `payment_cost` |
| `customers.csv` | `customer_id` | — | Customer segmentation: `segment` (Consumer tiers vs. B2B `Business` segment) |
| `channels.csv` | `channel_id` | — | Channel definitions: `STORE` (Offline) vs. `APP` and `WEB` (`DIGITAL`) |
| `marketing_spend.csv` | — | `period_id`, `channel_id` | Channel & campaign investments: `purpose` (Acquisition/Retention), `amount` |
| `periods.csv` | `period_id` | — | Longitudinal reporting period definitions |
| `members.csv` | `member_id` | — | Subscription profile: `status_2025_10_09`, `status_2026_10_09`, `plan_code`, `acquisition_channel` |
| `renewal_opportunities.csv` | `renewal_id` | `member_id` | Cohort tracking: `expiry_date`, `renewal_deadline`, `renewal_number` (1st vs. subsequent renewals) |
| `member_events.csv` | `event_id` | `renewal_id` | Telemetry logs: `event_type` (`PAYMENT_ATTEMPT`, `REMINDER_SENT`), `outcome` (`SUCCESS`, `FAILED`), `detail_code`, `event_time`, `event_cost` |

---

## 🧮 Part 3: Analytical Methodology & Mathematical Definitions

### 1. Order-Level P&L Contribution Waterfall
Evaluated strictly on `status == "COMPLETED"` orders across `period_id`, `channel_id`, `segment`, and `order_month`:

$$\text{Net Sales} = \text{Gross Amount} - \text{Discount Amount} - \text{Refund Amount}$$

$$\text{Gross Profit} = \text{Net Sales} - \text{Product Cost (COGS)}$$

$$\text{Order Contribution} = \text{Gross Profit} - \text{Fulfilment Cost} - \text{Payment Cost}$$

$$\text{Net Contribution} = \text{Order Contribution} - \text{Marketing Spend}$$

**Key Diagnostic Ratios:**
* $\text{Average Order Value (AOV)} = \frac{\text{Net Sales}}{\text{Completed Orders}}$
* $\text{Discount Rate (\%)} = \frac{\text{Discount Amount}}{\text{Gross Amount}} \times 100$
* $\text{Gross Margin (\%)} = \frac{\text{Gross Profit}}{\text{Net Sales}} \times 100$
* $\text{Net Contribution Margin (\%)} = \frac{\text{Net Contribution}}{\text{Net Sales}} \times 100$
* $\text{Unit Fulfilment \& Payment Cost} = \frac{\text{Fulfilment Cost}}{\text{Orders}}, \quad \frac{\text{Payment Cost}}{\text{Orders}}$

### 2. Omnichannel Customer Classification
For each `period_id`, active customers are partitioned into three mutually exclusive behavioral cohorts based on transaction history:
* **Omnichannel (`Store + Digital`):** $\text{Has Store Order} \land (\text{Has App Order} \lor \text{Has Web Order})$
* **Pure Offline (`Store Only`):** $\text{Has Store Order} \land \neg(\text{Has App Order} \lor \text{Has Web Order})$
* **Pure Digital (`App/Web Only`):** $\neg(\text{Has Store Order}) \land (\text{Has App Order} \lor \text{Has Web Order})$

The **Omnichannel Spend Multiple** is computed both *Across All Segments* and *Excluding the `Business` Segment* to prevent institutional bulk orders from distorting consumer cross-sell targets:

$$\text{Omnichannel Spend Multiple} = \frac{\text{Mean Net Sales}_{\text{Omnichannel}}}{\text{Mean Net Sales}_{\text{Single-Channel Cohort}}}$$

### 3. Right-Censoring & Renewal Funnel Diagnostics
Let $t_{\max} = \max(\text{event\_time})$ represent the data extraction cut-off timestamp:
* **Open vs. Closed Renewals:** A renewal opportunity $i$ is classified as **Open (Right-Censored)** if $\text{renewal\_deadline}_i > t_{\max}$, and **Closed** if $\text{renewal\_deadline}_i \le t_{\max}$.
* **True Renewal Rate:** Evaluated strictly on closed opportunities (`ren_closed`), where success requires at least one `PAYMENT_ATTEMPT` with `outcome == "SUCCESS"`.
* **Lost Renewal Root-Cause Split:** Closed failed renewals are bifurcated by payment attempt count ($k$):
  * **Passive Churn (`No payment attempt`):** $k = 0$ (Cross-checked against `REMINDER_SENT` delivery outcomes).
  * **Involuntary Churn (`Attempted but failed`):** $k \ge 1$ with zero successes (Diagnosed via `detail_code` failure distributions and `event_cost`).

---

## 🔄 Part 4: Case-to-Data Mapping Matrix (Arvind → KALPA)

| Strategic Theme (Arvind Case) | KALPA Data Implementation | Feasibility | Scope & Structural Boundaries |
| :--- | :--- | :--- | :--- |
| **Channel Mix & Profitability** *(Exhibit 5)* | P&L waterfall by `channel_id` (`STORE`, `APP`, `WEB`) & `period_id` | ✅ **Fully Feasible** | All KALPA channels are direct; no 3rd-party wholesale/marketplace split. |
| **2.5x Omnichannel Spend Multiple** *(Page 9)* | Customer cohorting (`Omnichannel` vs. `Pure Offline` vs. `Pure Digital`) | ✅ **Fully Feasible** | Isolates B2B `Business` skew; lacks pre-purchase cross-channel browsing logs. |
| **Discounting & Unit Cost Leaks** *(Exhibit 6d)* | Monthly time-series (`order_month`) tracking `disc_pct`, `fulfil_per_order`, `payment_per_order` | ✅ **Fully Feasible** | Captures monthly regime shifts; lacks SKU-level price elasticity or season age. |
| **D2C Marketing & CAC Efficiency** *(Exhibit 6a, Page 12)* | `marketing_spend` joined to channel `order_contribution` by `purpose` | ⚠️ **Partially Feasible** | Computes true `net_contribution` margin; lacks ad click/impression logs or new-buyer flags for marginal CAC. |
| **"Cohorts Within Cohorts" Retention** *(Page 11)* | Multi-dimensional renewal rates by `expiry_year`, `renewal_number`, `plan_code`, `acquisition_channel` | ✅ **Fully Feasible** | Exceeds Arvind's stated cohort capability for subscription members; checks ID overlap with retail `customers`. |
| **Automated Payment Failure Alerts** *(Page 10)* | `member_events` diagnostics on `PAYMENT_ATTEMPT` (`FAILED`), `detail_code`, and `REMINDER_SENT` | ✅ **Fully Feasible** | Quantifies involuntary vs. voluntary churn and activity-based `event_cost`. |
| **Store SOH & Endless Aisle** *(Exhibits 6b, 7, 8)* | *Not Present in Schema* | ❌ **Not Applicable** | KALPA dataset contains no SKU catalog, inventory stock-on-hand (SOH), or ship-from-store routing tables. |

---

## 🚀 Part 5: Key Findings & Actionable Management Recommendations

| Priority | Strategic Initiative | Analytical Evidence in KALPA | Target Measurable KPIs |
| :--- | :--- | :--- | :--- |
| **1. Immediate (Quick Win)** | **Deploy Automated Payment Retry & Reminder Fallback Workflows** | Lost closed renewals split cleanly into *Attempted but failed* (gateway/payment `detail_code` errors) and *No payment attempt* (correlated with failed `REMINDER_SENT` events). | • Payment Attempt Success Rate (%)<br>• Closed Renewal Rate (`is_success`)<br>• Reminder Delivery Rate (%) |
| **2. Immediate (Quick Win)** | **Enforce Guardrails on Monthly Discounting & Unit Fulfilment/Payment Step-Changes** | Monthly trend decomposition (`order_month`) exposes step-change spikes in `disc_pct`, `fulfil_per_order`, and `payment_per_order` that erode `order_contrib_pct`. | • Monthly `disc_pct`<br>• `fulfil_per_order` & `payment_per_order`<br>• Channel `order_contrib_pct` |
| **3. Medium-Term** | **Incentivize Single-Channel to Omnichannel Customer Migration** | Non-Business `Omnichannel` shoppers exhibit higher order frequency and spend multiples than `Pure Store` or `Pure Digital` buyers. | • % Active Customers in `Omnichannel` tier<br>• Non-Business `sales_per_cust`<br>• Omnichannel Spend Multiple |
| **4. Medium-Term** | **Reallocate Marketing Spend by Net Contribution & Cohort Renewal LTV** | Joining `marketing_spend` (`purpose`, `channel_id`) to `order_contribution` and `acquisition_channel` renewal rates reveals true bottom-line return on marketing. | • Channel `net_contrib_pct`<br>• 1st & 2nd Closed Renewal Rate by `acquisition_channel` |
| **5. Long-Term (Architecture)** | **Bridge `customer_id` and `member_id` into a Unified "One Customer" Graph** | Entity integrity check (`set(customers) & set(members)`) highlights the need to link retail order frequency with subscription renewal propensity, mirroring Arvind DCoE's master data lake. | • % Retail Orders linked to Active `member_id`<br>• Retail Spend Multiple (Members vs. Non-Members) |

---

## 💻 Repository Structure & How to Run

```text
├── data/                           # Directory containing the 9 relational CSV files
│   ├── channels.csv
│   ├── customers.csv
│   ├── marketing_spend.csv
│   ├── members.csv
│   ├── member_events.csv
│   ├── orders.csv
│   ├── order_costs.csv
│   ├── periods.csv
│   └── renewal_opportunities.csv
├── docs/
│   └── IIM_A_CASE_STUDY.pdf        # Arvind Fashions: Omnichannel Retailing in Digital India (IIMA/IS0147)
├── Kalpa_case_study_clean.ipynb    # Reproducible end-to-end Python analytical notebook
└── README.md                       # Project documentation & consulting synthesis
```

### Prerequisites & Execution
1. Clone the repository and ensure Python 3.9+ is installed with `pandas` and `numpy`:
   ```bash
   pip install pandas numpy jupyter
   ```
2. Place the 9 KALPA CSV files in the root directory (or update `DATA_DIR = Path(".")` in Cell 1 of the notebook).
3. Launch Jupyter and execute `Kalpa_case_study_clean.ipynb` top-to-bottom:
   ```bash
   jupyter notebook Kalpa_case_study_clean.ipynb
   ```
   *Note: The notebook is architected for strict reproducibility—every CSV is read exactly once, primary keys are asserted for uniqueness (`assert data[name][key].is_unique`), and reusable helper functions (`pnl()`, `with_marketing()`, `omni_profile()`, `renewal_rates()`) guarantee consistent definitions across all views.*

---

## 🎓 Acknowledgments & Case Citation
* **Case Study Reference:** Rao, H. R., Bose, I., Bhattacharya, S., & Lahiri, S. (2024). *Arvind Fashions: Omnichannel Retailing in Digital India* (Case No. IIMA/IS0147). Indian Institute of Management Ahmedabad.
* **Dataset Context:** KALPA Multi-Channel Commerce & Membership Case Dataset (IIT Gandhinagar).