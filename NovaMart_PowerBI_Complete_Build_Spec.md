# NovaMart Power BI — Complete Build Specification
All 4 pages, every DAX measure, every visual mapped to its data field, plus Row-Level Security.

---

# PART A — ALL DAX MEASURES (create these first, before building any visual)

Create each measure in the table noted, via: right-click table in Fields pane → **New Measure**.

## Measures on `sales_master`
```dax
Total Order Items = COUNTROWS(sales_master)

Total Revenue = SUM(sales_master[price])

Total Orders = DISTINCTCOUNT(sales_master[order_id])

Avg Order Value = 
DIVIDE(
    [Total Revenue],
    [Total Orders]
)

Active Sellers = DISTINCTCOUNT(sales_master[seller_id])
```

## Measures on `delivery_satisfaction`
```dax
Pct Late Orders = 
DIVIDE(
    CALCULATE(COUNTROWS(delivery_satisfaction), delivery_satisfaction[delivery_delay_days] > 0),
    COUNTROWS(delivery_satisfaction)
)

Avg Delivery Days = AVERAGE(delivery_satisfaction[delivery_days_actual])

Avg Delay (Late Only) = 
CALCULATE(
    AVERAGE(delivery_satisfaction[delivery_delay_days]),
    delivery_satisfaction[delivery_delay_days] > 0
)

Avg Review Score = AVERAGE(delivery_satisfaction[review_score])

Delay Satisfaction Correlation = -0.267
-- Hardcoded since MySQL/DAX don't have a native CORREL(); this was
-- calculated once in Python (pandas .corr()) and cross-validated in SQL.
-- Update manually only if the underlying dataset changes.
```

## Measures/Calculated Column on `seller_scorecard`
```dax
-- CALCULATED COLUMN (not a measure) — right-click table → New Column
-- This mirrors SQL Query 4.4 exactly, so the logic stays consistent
-- across your SQL analysis and the dashboard

Seller Status = 
IF(
    seller_scorecard[avg_review_score] < 3 && seller_scorecard[avg_delivery_delay] > 0,
    "At Risk",
    IF(
        seller_scorecard[avg_review_score] < 3 || seller_scorecard[avg_delivery_delay] > 5,
        "Watch",
        "Healthy"
    )
)
```

```dax
-- MEASURES on seller_scorecard
Total Sellers Scored = DISTINCTCOUNT(seller_scorecard[seller_id])

Pct Sellers At Risk = 
DIVIDE(
    CALCULATE(COUNTROWS(seller_scorecard), seller_scorecard[Seller Status] = "At Risk"),
    COUNTROWS(seller_scorecard)
)

Top Seller Revenue = MAX(seller_scorecard[total_revenue])
```

---

# PART B — PAGE 0: OVERVIEW

*(Full spec already covered — see prior message. Summary: Title/subtitle text boxes, 4 KPI cards using `Total Order Items`, `Pct Late Orders`, `Delay Satisfaction Correlation`, `Total Sellers Scored`, 3 nav buttons with Page Navigation actions, footer text boxes.)*

---

# PART C — PAGE 1: SALES OVERVIEW

### KPI Cards (4)
| Card | Measure | Format |
|---|---|---|
| Total Revenue | `Total Revenue` | ₹ currency, 0 decimals |
| Total Orders | `Total Orders` | thousands separator |
| Avg Order Value | `Avg Order Value` | ₹ currency, 2 decimals |
| Active Sellers | `Active Sellers` | thousands separator |

### Chart 1: Revenue by Category
- **Visual type:** Clustered bar chart (horizontal)
- **Axis:** `sales_master[product_category_name_english]`
- **Values:** `Total Revenue` measure
- **Sort:** Descending by Total Revenue, Top N filter = 10
- **Colors:** Apply categorical sequence from palette doc (`#2D6CDF`, `#7A5AF8`, `#0BA5EC`, `#F79009`, `#12B76A`)

### Chart 2: Revenue by State
- **Visual type:** Filled Map (or horizontal bar chart if map rendering is inconsistent)
- **Location:** `sales_master[customer_state]`
- **Color saturation:** `Total Revenue` measure
- **Gradient:** Format visual → Colors → set 3-stop gradient `#DCE8FC → #6CA6F7 → #2D6CDF`

### Chart 3: Monthly Revenue Trend
- **Visual type:** Line chart
- **X-axis:** `sales_master[order_purchase_timestamp]` — set to Month-level hierarchy (right-click field → change to Date Hierarchy, or create a calculated column `Order Month = FORMAT(sales_master[order_purchase_timestamp], "YYYY-MM")`)
- **Y-axis:** `Total Revenue` measure
- **Line color:** `#2D6CDF`

### Slicers
- Date range slicer → field: `order_purchase_timestamp`
- Category slicer → field: `product_category_name_english`
- State slicer → field: `customer_state`

---

# PART D — PAGE 2: DELIVERY & SATISFACTION

### KPI Cards (4)
| Card | Measure | Format |
|---|---|---|
| Avg Delivery Days | `Avg Delivery Days` | 1 decimal, "days" suffix via Format → Display units custom |
| % Late Orders | `Pct Late Orders` | percentage, 1 decimal |
| Avg Delay (Late) | `Avg Delay (Late Only)` | 1 decimal |
| Avg Review Score | `Avg Review Score` | 2 decimals, out of 5 |

### Chart 1: Late Delivery Rate by State
- **Visual type:** Clustered bar chart (horizontal), sorted descending
- **Axis:** need `customer_state` joined in — since `delivery_satisfaction` doesn't have state, either:
  - (a) Add a relationship-based lookup, or
  - (b) Simpler: create this **calculated column on `delivery_satisfaction`** using RELATED, IF a relationship to `sales_master` exists:
  ```dax
  Customer State = RELATED(sales_master[customer_state])
  ```
  *(Note: this only works cleanly if the relationship is 1-to-many from delivery_satisfaction to sales_master, which matches your model)*
- **Values:** `Pct Late Orders` measure
- **Colors:** Conditional formatting gradient `#D1FADF → #FEDF89 → #F04438` (green=good/low, red=bad/high)

### Chart 2: Review Score by Delivery Status
- **Visual type:** Clustered column chart
- **Need a calculated column first:**
  ```dax
  Delivery Status = IF(delivery_satisfaction[delivery_delay_days] > 0, "Late", "On-time/Early")
  ```
- **Axis:** `Delivery Status`
- **Values:** `Avg Review Score` measure
- **Colors:** "Late" bar = `#F04438`, "On-time/Early" bar = `#12B76A` (set manually per data point: Format visual → Colors → click each bar)

### Chart 3: Delay vs Review Score (Scatter)
- **Visual type:** Scatter chart
- **X-axis:** `delivery_satisfaction[delivery_delay_days]`
- **Y-axis:** `delivery_satisfaction[review_score]`
- **Values (for dot sizing, optional):** Count of orders
- This visualizes your -0.267 correlation directly — the cloud of points should trend slightly downward left-to-right

### Slicers
- State slicer → `Customer State` (calculated column above)
- Delivery status slicer → `Delivery Status` (calculated column above)

---

# PART E — PAGE 3: SELLER PERFORMANCE SCORECARD

### KPI Cards (3)
| Card | Measure | Format |
|---|---|---|
| Active Sellers | `Total Sellers Scored` | thousands separator |
| % Sellers At Risk | `Pct Sellers At Risk` | percentage, 1 decimal |
| Top Seller by Revenue | `Top Seller Revenue` | ₹ currency |

### Main Table: Seller Scorecard
- **Visual type:** Table (or Matrix)
- **Columns, in order:** `seller_id`, `seller_state`, `total_orders`, `total_revenue`, `avg_review_score`, `avg_delivery_delay`, `Seller Status`
- **Minimum order filter:** Add visual-level filter → `total_orders` ≥ 10
- **Conditional formatting on `Seller Status` column:**
  - Format visual → Cell elements → pick `Seller Status` field → Background color → Rules:
    - "At Risk" → `#F04438` background, white text
    - "Watch" → `#F79009` background, dark text
    - "Healthy" → `#12B76A` background, white text
  *(Set this via Format visual → Conditional formatting → Background color → Format by "Rules" won't directly do text-match easily; simplest method: Format by "Field value" using a helper measure that maps status to a color hex, OR manually set via Rules with "is" condition per category if Power BI version supports categorical rules)*

### Supporting Chart: Seller Count by Status
- **Visual type:** Donut chart or simple column chart
- **Legend/Axis:** `Seller Status`
- **Values:** Count of `seller_id`
- **Colors:** match status colors exactly (`#F04438` / `#F79009` / `#12B76A`)

### Slicers
- Seller state slicer → `seller_state`
- Min order threshold slicer → `total_orders` (slider mode)

---

# PART F — ROW-LEVEL SECURITY (RLS)

**What this demonstrates:** each seller (or region manager) only sees their own data when they log in — directly mirroring the Burns & McDonnell JD's enterprise BI security requirement.

### Step 1: Add a login-mapping approach
Since your dataset doesn't have real seller emails, the standard practice for a portfolio project is to **simulate** RLS using seller_id directly, and demonstrate it via Power BI's "View As Roles" feature (which doesn't require real logins).

### Step 2: Create the Role
1. Go to **Modeling** tab → **Manage Roles**
2. Click **Create**, name it `Seller Access`
3. Select the `seller_scorecard` table
4. In the DAX filter expression box, enter:
```dax
[seller_id] = "PASTE_ONE_REAL_SELLER_ID_HERE"
```
*(Grab any real `seller_id` value from your table to test with — right-click a cell in Data view → Copy Value)*

5. **Also apply the same filter concept to `sales_master`**, since sellers should only see their own order-level data too — create a second role or extend this one to filter `sales_master[seller_id]` the same way (Power BI will propagate the filter through the relationship automatically if the relationship direction supports it — check that your `sales_master ↔ seller_scorecard` relationship has **cross-filter direction set to "Both"**, Modeling tab → double-click the relationship → Cross-filter direction → Both)

### Step 3: Test it
1. Modeling tab → **View As**
2. Check the `Seller Access` role → OK
3. The whole report should now filter down to just that one seller's data across all pages — confirming RLS is working correctly

### Step 4: For your case study / interview talk-track
Explain that in a real deployment, this role would be dynamically driven by `USERPRINCIPALNAME()` (the logged-in user's email) matched against a seller-to-email mapping table, rather than a hardcoded ID — the hardcoded version here is a demonstration of the mechanism, which is standard practice for a portfolio project without real user accounts.

**Advanced version (if you want to show dynamic RLS), add a small mapping table:**
```
seller_login_map
| seller_id | login_email |
|-----------|-------------|
| xxxx...   | seller1@novamart.com |
```
Then the DAX filter becomes:
```dax
[seller_id] = LOOKUPVALUE(seller_login_map[seller_id], seller_login_map[login_email], USERPRINCIPALNAME())
```
This is the "real" production pattern — worth mentioning you know it exists even if you demo the simpler static version.

---

# BUILD ORDER CHECKLIST
1. [ ] Create all DAX measures in Part A first
2. [ ] Build Page 0 (Overview)
3. [ ] Build Page 1 (Sales Overview)
4. [ ] Build Page 2 (Delivery & Satisfaction) — including the 2 calculated columns
5. [ ] Build Page 3 (Seller Scorecard) — including conditional formatting
6. [ ] Set up Row-Level Security (Part F)
7. [ ] Apply the color theme globally (View → Customize current theme) so all pages stay visually consistent
8. [ ] Final polish pass: check alignment, consistent fonts, test all slicers and nav buttons
