# -Fourth_Project_Devlab

---

## 1. What is RFM?

RFM is a customer value framework built on three behavioral dimensions:

| Dimension | Question it answers | Unit |
|-----------|-------------------|------|
| **Recency (R)** | How recently did the customer buy? | Days since last order |
| **Frequency (F)** | How often do they buy? | # unique orders |
| **Monetary (M)** | How much do they spend? | Total USD sales |

Together, these three metrics capture the full behavioral profile of a customer without needing any demographic data. The model originates from direct-mail marketing (Hughes, 1994) but remains one of the most effective segmentation tools in modern CRM.

---

## 2. Dataset

- **Source:** `sales_data_sample.csv` — structured transactional data with columns: `ORDERNUMBER`, `ORDERDATE`, `SALES`, `CUSTOMERNAME`, `PRODUCTLINE`, etc.
- **Reference Date:** `2005-06-01` (one day after the latest order in the dataset)
- **Scope:** 92 unique customers across multiple product lines

### Key Dataset Statistics

| Metric | Min | Max | Mean |
|--------|-----|-----|------|
| Recency (days) | 1 | ~497 | ~77 |
| Frequency (orders) | 1 | 38 | ~15 |
| Monetary (USD) | ~3,450 | ~137,000 | ~56,000 |

---

## 3. RFM Calculation Logic

```python
reference_date = df['ORDERDATE'].max() + timedelta(days=1)

rfm = df.groupby('CUSTOMERNAME').agg(
    Recency   = ('ORDERDATE',   lambda x: (reference_date - x.max()).days),
    Frequency = ('ORDERNUMBER', 'nunique'),   # unique orders, not line items
    Monetary  = ('SALES',       'sum')
)
```

**Why `nunique` for Frequency?**  
Each order can have multiple line items. Counting unique `ORDERNUMBER` values avoids inflating frequency for customers who buy many SKUs per order.

---

## 4. Scoring Logic (1–4 Quantile Bins)

`pd.qcut` divides customers into 4 equal-frequency groups.

```python
# Recency: INVERT — lower days = more recent = better = score 4
R_Score = pd.qcut(Recency.rank(ascending=True),  q=4, labels=[4,3,2,1])

# Frequency: higher = better = score 4
F_Score = pd.qcut(Frequency.rank(ascending=False), q=4, labels=[4,3,2,1])

# Monetary: higher = better = score 4
M_Score = pd.qcut(Monetary.rank(ascending=False),  q=4, labels=[4,3,2,1])
```

**Why use `.rank()` before `qcut`?**  
Raw `pd.qcut` fails when there are duplicate values at bin boundaries. Ranking first guarantees unique cut points while preserving relative order.

**RFM Segment Code:** Each customer gets a 3-digit code like `"444"` (Champion) or `"111"` (Lost) by concatenating the three score strings.

---

## 5. Segment Definitions

| Segment | Score Logic | Description |
|---------|------------|-------------|
| 🏆 **Champions** | R≥3 AND F≥3 AND M≥3 | Best customers — recent, frequent, high-value |
| 💜 **Loyal** | F≥3 AND M≥3 (any R) | Frequent high-value buyers, slightly less recent |
| 🔵 **Needs Attention** | R≥3 AND F≤2 | Came back recently but haven't bought often |
| 🟠 **At Risk** | R≤2 AND F≥3 AND M≥3 | Were top customers — now lapsing |
| ⚫ **Lost** | All others (low across the board) | Inactive and low-value |

> **Note:** Segment rules are applied in priority order. Champions takes precedence over Loyal; At Risk captures formerly strong customers who have gone quiet.

---

## 6. Segment Actions (Marketing Playbook)

### 🏆 Champions (R≥3, F≥3, M≥3)
**Goal:** Maximize retention and turn into brand advocates.  
**Actions:**
- Enroll in a VIP loyalty tier with exclusive perks
- Offer referral bonuses — they're the most credible recommenders
- Give early access to new products/collections
- Send personalized thank-you notes with a dedicated account manager
- **KPI:** CLV growth, referral rate, retention rate

---

### 💜 Loyal (F≥3, M≥3)
**Goal:** Increase average order value and prevent churn.  
**Actions:**
- Cross-sell complementary product lines they haven't tried
- Create bundle offers with volume discounts
- Nudge toward a premium subscription tier
- Celebrate milestones (anniversary discounts, birthday offers)
- **KPI:** Average order value, purchase frequency uplift

---

### 🔵 Needs Attention (R≥3, F≤2)
**Goal:** Convert occasional buyers into repeat customers.  
**Actions:**
- Personalized product recommendations based on first purchase
- Onboarding email sequence: "Here's what you might love next"
- Time-limited offer to encourage a second purchase within 30 days
- Share use-case content or tutorials to deepen product engagement
- **KPI:** Repeat purchase rate within 60 days

---

### 🟠 At Risk (R≤2, F≥3, M≥3)
**Goal:** Win them back before they churn permanently.  
**Actions:**
- "We miss you" reactivation email with a personal subject line
- Offer a meaningful discount (15–20%) valid for 2 weeks
- Survey to understand why they stopped buying
- Assign to a customer success rep for direct outreach
- **KPI:** Re-engagement rate, time-to-next-purchase

---

### ⚫ Lost (Low R, F, M)
**Goal:** Attempt low-cost reactivation; sunset if unresponsive.  
**Actions:**
- One final "last chance" offer — no more than 2 touches
- Ask for opt-in re-permission to stay on list
- If no response after 30 days: remove from active campaigns
- Analyze what went wrong (price? product fit? service?)
- **KPI:** Reactivation rate, email unsubscribe rate, ROI per contact

---

## 7. Visualizations Produced

| File | Description |
|------|-------------|
| `seg_bar_chart.png` | Customer count per segment (bar chart) |
| `seg_scatter.png` | Recency vs Monetary scatter, colored by segment |
| `seg_heatmap.png` | Average R/F/M scores per segment (heatmap) |
| `seg_3d_scatter.png` | ⭐ Bonus: 3D scatter (Recency × Frequency × Monetary) |
| `rfm_report.csv` | ⭐ Bonus: Full RFM table exported for marketing team |

---

## 8. Key Findings

- **Champions** represent a small group of customers that generate a disproportionately large share of total revenue — protect this segment above all others.
- **At Risk** customers are the highest-priority recovery target: they were once valuable and the relationship can still be saved with the right touch.
- **Lost** customers should only receive a minimal, cost-efficient reactivation attempt before being sunset to avoid wasted marketing spend.
- **Needs Attention** is an underrated opportunity: these customers are already engaged (recent), they just need a reason to come back more often.

---

## 9. Limitations & Next Steps

- **Static snapshot:** RFM is computed at a single point in time. Set up a scheduled pipeline to recompute monthly and track segment migration.
- **Equal weights:** All three dimensions are weighted equally. Consider weighted RFM (e.g., Monetary × 1.5) for high-margin businesses.
- **Segment overlap:** Customers near thresholds may shift segment with minor changes. Consider fuzzy membership or clustering (K-Means) as a follow-up.
- **Causality:** RFM describes behavior; it doesn't explain why. Combine with qualitative feedback for complete understanding.

---



---

*Built as part of the Devlab Data Analytics Internship — Week 2 Advanced Project.*
