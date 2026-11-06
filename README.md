# Retail Commercial & Margin Optimization Analytics

![Power BI](https://img.shields.io/badge/Power_BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-00758F?style=for-the-badge&logo=data&logoColor=white)
![Data Modeling](https://img.shields.io/badge/Data_Modeling-Star_Schema-blue?style=for-the-badge)
![Security](https://img.shields.io/badge/Security-Dynamic_RLS-darkgreen?style=for-the-badge)

An end-to-end commercial analytics solution analyzing **1,370 orders** (1,952 transaction line items totaling **$1.92M** in gross revenue). 

The goal of this project was to audit commercial unit economics, isolate product-level profit leakages, and determine whether high-volume corporate discounts generate real incremental margin or simply erode profitability.

---

## 🔗 Project Assets
- **Power BI Production Workbook:** [Download .pbix File](pbix/Retail_Margin_&_Pricing_Intelligence.pbix)
- **Executive Strategic Briefing:** [Download PDF Summary](Retail_Margin_Optimization_Executive_Summary.pdf)
- **Author LinkedIn:** [Szymon Khrapachenko](https://www.linkedin.com/in/szymon-khrapachenko)

---

## 📊 Interactive Demo & Visuals

![Dashboard Demo](screenshots/demo.gif)

### 1. Sales & Regional Overview
Monitors monthly revenue vs. margin % trends across four sales territories using Small Multiples, fully cross-filtered by fulfillment type and customer segment.

![Sales and Regional Overview](screenshots/01_sales_regional_overview.png)

### 2. Profitability & Cost Drivers
Evaluates discount elasticity and logistics costs. Identifies chronic loss-making sub-categories and visualizes profitability thresholds against volume-weighted discounts.

![Profitability and Cost Drivers](screenshots/02_profitability_cost_drivers.png)

---

## 💡 Key Business Findings

1. **The $21.7K Margin Drain:** While overall revenue reached **$1.92M** (delivering a 9.47% baseline margin), six sub-categories operated at a loss. The primary contributor was **Tables (-$7.24K net loss)** due to aggressive discounting and disproportionate logistics overhead.
2. **Corporate Segment Margin Compression:** Margins in the Corporate tier dropped to **2.62%** ($17.17K profit despite massive revenue volume), driven by an unmanaged weighted discount average of **5.06%**.
3. **Logistics & Freight Mismatch:** 
   - Shipping bulky items like **Tables** via **Regular Air** produced a catastrophic **-40.61% margin**.
   - Ground **Delivery Truck** fulfillment yielded only **1.22% margin** on corporate deliveries due to the absence of weight/dimensional surcharges.
4. **The Discount Cliff:** Once volume-weighted discounts exceed **4.5%**, unit margins plunge below zero without generating the sales velocity needed to offset the loss.

---

## 🛠️ Data Architecture & Governance

The semantic model is engineered as a strict, single-direction **Star Schema** to optimize VertiPaq memory footprint and ensure instant report interactivity:

```text
                       +----------------------+
                       |    Dim_Customer      |
                       +----------+-----------+
                                  | 1:N
                                  v
+-----------------+ 1:N +--------------------+ 1:N +------------------+
|   Dim_Product   |---->|     Fact_Sales     |<----|  Dim_Geography   |
+-----------------+     +---------+----------+     +------------------+
                                  ^
                                  | 1:N
                        +---------+----------+
                        |      Dim_Date      |
                        +--------------------+

[ Security_User_Access ] (Disconnected table for Dynamic RLS evaluation)
```

---

### Governance Best Practices Applied

* **Zero Implicit Measures:** All numeric fields in Fact_Sales are explicitly hidden to prevent unaggregated dragging into visuals.
* **Data Anomaly Sanitization:** Cleaned raw data discrepancies in Power Query (e.g., negative supplier COGS inflating margins >100%) to ensure clean mathematical integrity.
* **Dynamic Row-Level Security (RLS):** Implemented user-based access control via an isolated Security_User_Access table without breaking relationship cardinality.

---

## 💻 Core DAX Solutions

**1. Volume-Weighted Discount %**

A simple AVERAGE() distorts discount analysis because a 50% discount on a $5 item has the same mathematical weight as a 2% discount on a $10,000 order. This measure weights discount rates by actual gross revenue:

```dax
Weighted Avg Discount % = 
DIVIDE(
    SUMX(Fact_Sales, Fact_Sales[Discount] * Fact_Sales[Sales]),
    [Total Sales ($)],
    BLANK()
)
```

**2. Dynamic Loss-Making Subtitle**

Dynamically recalculates the cumulative dollar loss of unprofitable items inside the active filter context:

```dax
Loss Subtitle Dynamic = 
VAR _LossTable = 
    FILTER(
        VALUES(Dim_Product[Product Sub-Category]),
        [Total Profit ($)] < 0
    )
VAR _TotalLoss = SUMX(_LossTable, [Total Profit ($)])
RETURN
    IF(
        ISBLANK(_TotalLoss),
        "No loss-making items in current selection",
        "Total Loss for selected items: " & 
        FORMAT(DIVIDE(_TotalLoss, 1000), "$#,##0.00K") & 
        " | Hover over bars for details"
    )
```

**3. Dynamic Regional RLS (Disconnected Pattern)**

Enforces territory-level data isolation based on USERPRINCIPALNAME() using an isolated lookup table:

```dax
VAR _UserEmail = LOWER(USERPRINCIPALNAME())
VAR _UserRegions = 
    CALCULATETABLE(
        VALUES('Security_User_Access'[Region]),
        FILTER(
            ALL('Security_User_Access'),
            LOWER('Security_User_Access'[Email]) = _UserEmail
        )
    )
RETURN
    IF(
        "ALL" IN _UserRegions,
        TRUE(),
        'Dim_Geography'[Region] IN _UserRegions
    )
```

---

## 📁 Repository Structure

```text
├── pbix/
│   └── Retail_Margin_&_Pricing_Intelligence.pbix     # Production Power BI Workbook
├── screenshots/
│   ├── 01_sales_regional_overview.png                # Page 1 Visual Capture
│   ├── 02_profitability_cost_drivers.png             # Page 2 Visual Capture
│   └── demo.gif                                      # Interactive UI Preview
├── Retail_Margin_Optimization_Executive_Summary.pdf   # 5-Slide Boardroom PDF
└── README.md                                         # Project Documentation
```
