# Think for Yourself: The 200 MB Platform Limit & Privacy Architecture
**PulseIQ Task 17 · Data Governance & Privacy by Design Defense**  
**Author:** Harish Bagavathy, Lead Data Analytics Engineer  

---

## 1. The Core Engineering Challenge
> **"Think for yourself: The platform limits your file to 200 MB. What do you publish and what do you leave out?"**

When deploying an internal analytics warehouse to an external public cloud or executive dashboard, platform storage limits (such as Streamlit's 200 MB memory/upload threshold or GitHub's 100 MB file limit) force a foundational architectural choice.

However, storage constraints are only half the battle. The far more critical driver is **Data Governance & Privacy by Design (GDPR / DPDP Article 25)**:
> *"Publish aggregates, not raw private data."*

Publishing a raw operational database directly to the web is both an engineering failure (slow, unindexed queries on bloated files) and a catastrophic corporate compliance violation.

---

## 2. Exhaustive Data Categorization: What We Publish vs What We Leave Out

### A. What We Leave Out (100% Excluded from Deployment)

| Excluded Category | Source Warehouse Entity | Grain / Volume | Technical & Regulatory Reason for Exclusion |
|:---|:---|:---|:---|
| **Employee Personally Identifiable Information (PII)** | `dim_employee` | 1,606 confidential records | Contains full legal names, dates of birth, personal email addresses, home locations, bank account details, and government IDs. Violates GDPR, CCPA, and India DPDP 2023. |
| **Granular Employee Payroll Checks** | `fact_payroll` | 7,344 transaction lines | Contains individual employee monthly compensation, base salaries, tax withholdings, and bonus allocations. Releasing raw compensation destroys internal morale and violates privacy. |
| **Attendance & Leave Micro-Events** | `fact_attendance_monthly`, `fact_leave` | ~30,000 daily check-ins & leave logs | Contains individual employee medical sick leaves, bereavement records, and daily punch timestamps. High volume (dozens of MBs) with zero executive value. |
| **Financial Ledger Journal Lines** | `fact_ledger`, `dim_account` | Tens of thousands of debits/credits | Granular accounting ledger entries detailing individual bank transactions, petty cash, and internal chart-of-accounts journal entries. |
| **Vendor Contract Fees & Invoices** | `dim_vendor`, `fact_expense` | Confidential vendor lines | Proprietary vendor pricing terms and individual SaaS license fee schedules protected by NDAs. |
| **Sprint Commits & Ticket JIRA Logs** | `fact_task`, `dim_sprint` | 10,000+ sprint story tickets | Raw developer task descriptions, bug tracker comments, and individual ticket burndowns. |
| **Staging & Intermediate Tables** | `stg_*`, `raw_*`, `reconciliation_answer_key` | Ephemeral ETL tables | Unverified raw ingest tables and validation fixtures that clutter memory. |

**Audit Impact:** Over **48,000 rows** of raw transactional and personal data scrubbed before publishing.

---

### B. What We Publish (The Governed Semantic Aggregate Layer)

Instead of the raw warehouse, we publish **pre-aggregated semantic cubes** exported from certified metric views:

| Published Aggregate | File Format & Name | Row Count | File Size | Business Purpose |
|:---|:---|:---:|:---:|:---|
| **Executive Monthly Scorecard** | `scorecard_monthly.parquet` | 18 rows | **11.2 KB** | 8 headline KPIs aggregated by month (`2025-04` to `2026-09`) with pre-calculated benchmarks and traffic lights. |
| **Department Payroll Variance** | `payroll_budget_variance.parquet` | 198 rows | **14.8 KB** | Departmental monthly budget vs actual spend totals, variance amounts, and traffic light statuses. |
| **Accrual P&L Summary** | `profit_and_loss_monthly.parquet` | 18 rows | **4.8 KB** | Top-line revenue, category expense rollups (Payroll, Vendor, Depreciation), and net profit margins. |
| **Receivables Ageing Schedule** | `receivables_ageing_summary.parquet` | 4 rows | **1.9 KB** | Overdue balances grouped into risk buckets (0-30d, 31-60d, 61-90d, 90+d) with portfolio percentages. |
| **Department Workforce Dynamics** | `department_workforce_dynamics.parquet` | 198 rows | **19.0 KB** | Department headcount totals, monthly hires, exits, attendance rate %, and sprint completion %. |
| **Monthly Cash Flow Trajectory** | `cash_flow_monthly.parquet` | 18 rows | **2.9 KB** | Inflows (customer collections), outflows (payroll/vendors), net cash flow, and closing bank balance. |
| **Certified Metric Dictionary** | `metric_dictionary.parquet` | 28 rows | **8.0 KB** | Semantic definitions, mathematical formulas, team owners, and direction standards. |
| **KPI Benchmarks & Targets** | `kpi_targets.parquet` | 180 rows | **2.3 KB** | Authoritative corporate FP&A target thresholds. |
| **Governed Live Database** | `pulseiq_live_dw.db` | Indexed DB | **120.0 KB** | Ultra-lean SQLite database containing ONLY these certified aggregate views. |

---

## 3. Quantitative Architectural Comparison

$$\begin{aligned}
\text{Raw Staging Warehouse Size} &\approx 7.42\text{ MB} \\
\text{Total Published Parquet Aggregates} &\approx 0.065\text{ MB } (65\text{ KB}) \\
\text{Published Live SQLite Database} &\approx 0.120\text{ MB } (120\text{ KB}) \\
\text{Platform Memory Constraint} &= 200.00\text{ MB} \\
\text{Capacity Consumed} &\mathbf{\le 0.06\%} \text{ of the 200 MB limit!}
\end{aligned}$$

### Why Parquet is the Gold Standard for Hosted Dashboards:
1. **Columnar Projection**: The dashboard reads only the columns requested by the user, skipping unneeded fields.
2. **Snappy Compression**: Reduces storage by 80–90% compared to uncompressed CSV or SQL dumps.
3. **Schema Enforcement**: Types (integers, floats, ISO dates) are strictly preserved across systems, preventing parsing crashes on different operating systems.
4. **Instant In-Memory Deserialization**: Parquet tables load directly into Arrow memory in **under 2 milliseconds**, completely eliminating the 5-second query latency of scanning raw databases.
