# Benchmarking Auto Parts Sales with SQL-Based Market Comparison

### Table of Contents  
- [Project Overview](#project-overview)  
- [Objectives](#objectives)  
- [Data Sources](#data-sources)  
- [Tools](#tools)  
- [Data Preparation](#data-preparation)  
- [SQL Methodology](#sql-methodology)  
- [Benchmarking & Opportunity Analysis](#benchmarking--opportunity-analysis)  
- [KPIs & Pseudo Analysis](#kpis--pseudo-analysis)  
- [Next Steps](#next-steps)

---

### Project Overview

In the global auto parts industry, wholesale suppliers operate within a highly competitive and fragmented marketplace, where product demand, brand preferences, and pricing strategies vary dramatically by region. This project leverages SQL to benchmark our internal sales performance against regional market demand, enabling a deeper understanding of how well our product lines perform in each geographic area—including North America, Central America, South America, Europe, Africa, and the Middle East.

By mapping retailer demand data to our internal catalog using a dynamic Master Cross system, we are able to translate a wide variety of OEM and competitor part numbers (e.g., Moog, Mevotech, Dorman, MAS, Delphi) into our internal identifiers. This allows us to perform robust, SKU-level comparisons between our actual sales and the total market opportunity. Through this analysis, we can:

Measure market penetration for each part in each region

Identify coverage gaps where demand exists but our product was not sold

Recognize saturation zones where our sales align well with demand

Support strategic decisions on pricing, stocking, and catalog development based on real-world market behavior

Ultimately, the goal is to better understand regional market dynamics and improve our competitiveness by continuously adapting our offering to what each market truly demands.

---


### Objectives

- **Normalize and integrate** inconsistent, retailer-sourced sales and forecast data to create a unified foundation for market comparison  
- **Cross-reference** diverse competitor part numbers and OEM references against internal SKUs using a dynamic Master Cross system  
- **Benchmark our sales** performance against total regional demand in order to:  
  - Identify **underperforming SKUs** where sales exist but significantly trail behind demand  
  - Detect **coverage gaps** where demand exists but no sales were captured  
  - Measure **market penetration levels** across key regions and brands  
  - Pinpoint **strategic growth opportunities** in underserved areas  
- Support sales and catalog decisions by delivering results through **SQL-based outputs** and **Power BI live dashboards** connected to cleaned, structured data


---

### Data Sources

### Data Sources

- **Retailer Demand Files** (format varies by source)  
  These are Excel-based sales or forecast datasets collected externally by our sales representatives from retail partners.  
  - Common Metrics: `Qty Sold`, `Brand`, `Description`, `Region`, `Date`  
  - Timeframes: Varies (6M, 12M, 18M); all values are normalized to reflect **annual demand**

- **Internal Sales Data**  
  Internal sales performance data is accessed **directly within Power BI**, where it is maintained and refreshed.  
  - Not hosted on SQL Server  
  - Used in **live benchmarking analysis** by joining Power BI results to SQL outputs

- **Cross Reference Master Table**  
  The master catalog (`01_CrossMaster`) contains mappings between our internal SKUs and equivalent part numbers from OEMs and aftermarket brands (e.g., Moog, Delphi, MAS, Mevotech).  
  - Structure:  
    - `SusCatalog` (Internal Part Number)  
    - `OEM Number`  
    - Dozens of **brand-specific columns**, one for each competitor (e.g., `Moog`, `MevotechOG`, `DormanCond`, etc.)

  Because these crosses are spread across columns, we use the stored procedure `sp_CreateMasterCrossReferencesCK` to **flatten** the table into a clean, queryable structure:
  - `CrossName` – the brand column name  
  - `CrossingNumber` – the competitor or OEM part number  
  - `SusCatalog` – our internal part number

  This flattened structure enables efficient and consistent joins between external retailer data and internal catalog identifiers.

---

### Tools

- **SQL Server (T-SQL)**: Data cleaning, transformation, aggregation  
- **Power BI**: Used solely to retrieve internal sales figures for integration

---

### Data Preparation

Retailer demand files are received in a variety of formats and structures, depending on the source. Each file must be carefully cleaned and standardized before it can be integrated into the benchmarking model. Key steps and challenges include:

- **Inconsistent Part Numbers**  
  Part numbers often contain dashes, dots, extra spaces, or inconsistent casing. These are cleaned to generate **condensed versions** used for reliable matching.

- **Cross-Matching to Internal Catalog**  
  Cleaned part numbers from retailer files are joined to our internal SKUs using the `Master_CrossReferences_CK` table. This allows us to identify the corresponding internal `SusCatalog` for each external part reference.

- **Brand Filtering**  
  Only **OE-grade** aftermarket brands are included in the analysis. Budget or economy-tier brands are excluded to maintain consistency with our product positioning.

- **Region and Date Attribution**  
  Retailer files may lack explicit region or time period fields. These are manually inferred and appended based on the source of the file or accompanying metadata.

> **Note:** In cases where multiple external part numbers map to the same internal SKU, deduplication is handled through row-level filtering. Superseded or incorrect catalog crosses are updated manually in the master cross-reference table to ensure alignment across all projects.

---

### SQL Methodology

1. **Standardization**  
   - All part numbers are transformed into **condensed formats** by removing special characters (e.g., dashes, dots, spaces) to ensure consistent matching  
   - Sales data timeframes (6M, 12M, 18M) are **normalized to annual values** to enable fair, apples-to-apples comparison across sources

2. **Cross-Matching Process**  
   - Cleaned part numbers from retailer files are joined to the flattened `Master_CrossReferences_CK` table  
   - This resolves each external part number to our internal `SusCatalog` value  
   - If multiple matches are found due to overlap or format collisions, only one is retained using `ROW_NUMBER()` filtering  
   - This avoids double-counting and ensures that each retailer part number maps to a **single internal SKU**

3. **Demand Aggregation**  
   - After matching, all retailer demand data is **grouped by region and internal SKU**  
   - Aggregated quantities are calculated to reflect **total external demand** for each part-region pair

4. **Sales Integration**  
   - Internal sales performance is retrieved directly from **Power BI**  
   - These sales figures are **joined to the demand table** using our part number (`SusCatalog`) as the key  
   - This merged dataset enables precise benchmarking, gap analysis, and opportunity scoring

---


### SQL Procedures & Data Flow Explanation

This section documents the core SQL procedures used in the benchmarking workflow. These procedures are vital for transforming, unifying, and analyzing external sales data from retailers and internal catalogs.

---
#### Data Cleaning

After compiling and importing the raw sales data from global sales representatives into the SQL Server table `ALL_CHASSIS_SALES_RAW`, a preprocessing step is performed to clean part numbers and prepare the dataset for accurate cross-referencing.

- **Standardize Part Numbers**:  
  Raw part numbers often contain inconsistent formatting such as dashes, dots, slashes, or spaces. These characters are removed to generate a clean, condensed version.

- **Create `Part_Number_Cond` Field**:  
  A new column, `Part_Number_Cond`, is generated to store the cleaned version of each part number. This ensures that all matching and joining operations (particularly with the Master Cross Reference table) are based on consistent part number formatting.

```sql
SELECT
    ID,
    Brand,
    Part_Number,
    UPPER(REPLACE(REPLACE(REPLACE(REPLACE([Part_Number], '-', ''), '.', ''), ' ', ''), '/', '')) AS Part_Number_Cond,
    Last_12_Months_Sales,
    List,
    Region,
    Year_Received
INTO dbo.ALL_CHASSIS_SALES_RAW_CLEANED
FROM dbo.ALL_CHASSIS_SALES_RAW;
```

##### Example Transformation:

| Part_Number   | Part_Number_Cond |
|---------------|------------------|
| `1AS-BJ00132` | `1ASBJ00132`     |
| `TRQ.123 / A` | `TRQ123A`        |
| ` 123-456 `   | `123456`         |



#### `[sp_CreateMasterCrossReferencesCK]`  

**Purpose**:  
Transform the wide-format cross-reference catalog (`01_CrossMaster`) into a **flattened, vertically structured table** (`Master_CrossReferences_CK`) to support efficient and scalable joins between competitor part numbers and our internal catalog.

Retailer demand files often include competitor part numbers (e.g., from Moog, Delphi, MAS, etc.), but in the source catalog, each competitor's part number is stored in its own dedicated column. This stored procedure restructures the catalog into a unified reference table with the following columns:

- `CrossName` → The name of the original brand column (e.g., `Moog`, `DelphiCond`)  
- `CrossingNumber` → The competitor or OEM part number  
- `SusCatalog` → Our internal part number used across all analysis

Using a `UNION ALL` approach, this procedure vertically stacks all relevant columns from `01_CrossMaster` into a single table. It also filters out nulls to maintain clean and reliable joins.

The resulting `Master_CrossReferences_CK` table supports flexible cross-matching logic, whether for market benchmarking, catalog alignment, or sales performance analysis. Conditional (`Cond`) and related-brand fields are also included to maximize compatibility across all use cases, even if not all brands are used in every analysis.

This procedure is executed **on demand**, typically just before running a new benchmarking or sales comparison cycle.

>  *Note:* All part numbers used in matching are cleaned to a condensed format (removing dashes, dots, and spaces) to ensure consistent joins and prevent formatting mismatches.

```sql
IF OBJECT_ID('dbo.Master_CrossReferences_CK', 'U') IS NOT NULL
    DROP TABLE dbo.Master_CrossReferences_CK;

SELECT DISTINCT t.CrossName, t.CrossingNumber, t.SusCatalog
INTO dbo.Master_CrossReferences_CK
FROM (
    SELECT 'OEMCond' AS CrossName, OEMCond AS CrossingNumber, SusCatalog FROM Suspensia_New.dbo.01_CrossMaster
    UNION ALL
    SELECT 'Moog', Moog, SusCatalog FROM Suspensia_New.dbo.01_CrossMaster
    UNION ALL
    SELECT 'Delphi', Delphi, SusCatalog FROM Suspensia_New.dbo.01_CrossMaster
    -- +40 additional brand columns in full implementation
    UNION ALL
    SELECT 'Sankei', Sankei, SusCatalog FROM Suspensia_New.dbo.01_CrossMaster
) AS t
WHERE t.CrossingNumber IS NOT NULL AND t.SusCatalog IS NOT NULL;
```

#### Example Transformation

The original `01_CrossMaster` table stores competitor part numbers in a **wide format**, where each column represents a different brand:

| SusCatalog | Moog     | Delphi   | MAS       | OEMCond   |
|------------|----------|----------|-----------|-----------|
| SUS-10001  | K123456  | DS45678  | MS98765   | 12345678  |
| SUS-10002  | K789012  | NULL     | MS65432   | 87654321  |

The stored procedure `[sp_CreateMasterCrossReferencesCK]` transforms this into a **long (tall)** format for efficient querying and cross-referencing:

| CrossName | CrossingNumber | SusCatalog |
|-----------|----------------|------------|
| Moog      | K123456        | SUS-10001  |
| Delphi    | DS45678        | SUS-10001  |
| MAS       | MS98765        | SUS-10001  |
| OEMCond   | 12345678       | SUS-10001  |
| Moog      | K789012        | SUS-10002  |
| MAS       | MS65432        | SUS-10002  |
| OEMCond   | 87654321       | SUS-10002  |

This flattened format enables us to **match any part number, from any brand**, against our catalog without having to search across multiple columns.

---

### 2. `[Update_Chassis_Rank]`  
**Purpose**: Aggregate retailer sales data by region, resolve crosses to internal part numbers, and rank performance by part and geography.

- `ALL_CHASSIS_SALES_RAW`: Cleaned and compiled from Excel files submitted by sales reps.
- This table includes:  
  `Part_Number`, `Brand`, `Qty`, `Region`, `Date`, `Sales`, and `Part_Number_Cond` for normalized comparisons.


####  1st Part: Cross Reference Matching
- Joins `ALL_CHASSIS_SALES_RAW` with `Master_CrossReferences_CK` using the `Part_Number_Cond` field.
- Uses `ROW_NUMBER()` to deduplicate multiple matches for the same part by selecting only the first record.
  - This avoids **double aggregation** of sales data.
  - Ensures one-to-one mapping when multiple brands point to the same internal SKU.

#### 2nd Part: Regional Aggregation & Ranking
- Aggregates sales by `SusCatalog` across all major regions:
  - North America
  - Mexico
  - Puerto Rico
  - Europe
  - Africa
  - Central America
  - South America
  - Middle East
- Computes **Total Sales** and region-specific subtotals.
- Applies `RANK()` function per region to create region-based leaderboard metrics for each internal part number.

```sql
ALTER PROCEDURE [dbo].[Update_Chassis_Rank]
AS
BEGIN
    SET NOCOUNT ON;

    IF OBJECT_ID('CAN.dbo.ALL_CHASSIS_SALES_RAW_crossed', 'U') IS NOT NULL
        DROP TABLE CAN.dbo.ALL_CHASSIS_SALES_RAW_crossed;

    SELECT 
        A.ID, A.Brand, A.Part_Number, A.Part_Number_Cond,
        A.Last_12_Months_Sales, A.List, A.Region, A.Year_Received,
        F.CrossName, F.CrossingNumber, F.SusCatalog
    INTO CAN.dbo.ALL_CHASSIS_SALES_RAW_crossed
    FROM CAN.dbo.ALL_CHASSIS_SALES_RAW A
    LEFT JOIN (
        SELECT CrossName, CrossingNumber, SusCatalog
        FROM (
            SELECT *, ROW_NUMBER() OVER (PARTITION BY CrossingNumber ORDER BY CrossName) AS row_num
            FROM CAN.dbo.Master_CrossReferences_CK
        ) R
        WHERE row_num = 1
    ) F ON A.Part_Number_Cond = F.CrossingNumber;

    IF OBJECT_ID('CAN.dbo.Chassis_Rank', 'U') IS NOT NULL
        DROP TABLE CAN.dbo.Chassis_Rank;

    ;WITH RegionalSales AS (
        SELECT 
            SusCatalog,
            SUM(CASE WHEN Region = 'North America' THEN Last_12_Months_Sales ELSE NULL END) AS NA_Sales,
            SUM(CASE WHEN Region = 'Mexico' THEN Last_12_Months_Sales ELSE NULL END) AS Mexico_Sales,
            SUM(CASE WHEN Region = 'Puerto Rico' THEN Last_12_Months_Sales ELSE NULL END) AS PuertoRico_Sales,
            SUM(CASE WHEN Region = 'Europe' THEN Last_12_Months_Sales ELSE NULL END) AS Europe_Sales,
            SUM(CASE WHEN Region = 'Africa' THEN Last_12_Months_Sales ELSE NULL END) AS Africa_Sales,
            SUM(CASE WHEN Region = 'Central America' THEN Last_12_Months_Sales ELSE NULL END) AS Central_Sales,
            SUM(CASE WHEN Region = 'South America' THEN Last_12_Months_Sales ELSE NULL END) AS SA_Sales,
            SUM(CASE WHEN Region = 'Middle East' THEN Last_12_Months_Sales ELSE NULL END) AS ME_Sales,
            SUM(Last_12_Months_Sales) AS Total_Sales
        FROM CAN.dbo.ALL_CHASSIS_SALES_RAW_crossed
        WHERE SusCatalog IS NOT NULL
        GROUP BY SusCatalog
    )
    SELECT *, 
        RANK() OVER (ORDER BY Total_Sales DESC) AS Rank_Total
    INTO CAN.dbo.Chassis_Rank
    FROM RegionalSales;
END;
```

#### Example Transformation

**Raw Input** (`ALL_CHASSIS_SALES_RAW` — sample)

| ID | Brand  | Part_Number | Part_Number_Cond | Last_12_Months_Sales | Region        | Year_Received |
|----|--------|--------------|------------------|------------------------|----------------|----------------|
| 1  | Moog   | K123456      | K123456          | 1200                   | North America  | 2023           |
| 2  | MAS    | MS98765      | MS98765          | 600                    | Mexico         | 2023           |
| 3  | Delphi | DS45678      | DS45678          | 900                    | Europe         | 2023           |
| 4  | OEM    | 12345678     | 12345678         | 1500                   | North America  | 2023           |
| 5  | Moog   | K789012      | K789012          | 500                    | South America  | 2023           |
| 6  | MAS    | MS65432      | MS65432          | 300                    | Central America| 2023           |
| 7  | Delphi | DS11111      | DS11111          | 700                    | Mexico         | 2023           |
| 8  | OEM    | 87654321     | 87654321         | 1000                   | Europe         | 2023           |
| 9  | Moog   | K222333      | K222333          | 450                    | Middle East    | 2023           |

**Cross-Matching** (`Master_CrossReferences_CK` — simplified sample)

| CrossName | CrossingNumber | SusCatalog  |
|-----------|----------------|-------------|
| Moog      | K123456        | SUS-10001   |
| MAS       | MS98765        | SUS-10001   |
| Delphi    | DS45678        | SUS-10001   |
| OEMCond   | 12345678       | SUS-10001   |
| Moog      | K789012        | SUS-10002   |
| MAS       | MS65432        | SUS-10002   |
| Delphi    | DS11111        | SUS-10003   |
| OEMCond   | 87654321       | SUS-10003   |
| Moog      | K222333        | SUS-10004   |

**Joined + Aggregated Output** (`Chassis_Rank` — sample)

| SusCatalog | NA_Sales | Mexico_Sales | Europe_Sales | Central_Sales | SA_Sales | ME_Sales | Total_Sales | Rank_Total |
|------------|----------|--------------|--------------|----------------|----------|----------|--------------|-------------|
| SUS-10001  | 2700     | 600          | 900          | NULL           | NULL     | NULL     | 4200         | 1           |
| SUS-10002  | NULL     | NULL         | NULL         | 300            | 500      | NULL     | 800          | 3           |
| SUS-10003  | NULL     | 700          | 1000         | NULL           | NULL     | NULL     | 1700         | 2           |
| SUS-10004  | NULL     | NULL         | NULL         | NULL           | NULL     | 450      | 450          | 4           |

---

This final output provides the foundation for:

- Regional **performance dashboards**
- **Gap analysis** (e.g., no sales in high-demand regions)
- **Part-level benchmarking** for pricing, stocking, and catalog expansion

---

#### 🧠 Business Use
- Output table `Chassis_Rank` is saved for:
  - Power BI live dashboards
  - MS Access queries for business operations
- Used in multiple follow-up analyses:
  - Regional gap analysis
  - Sales trend detection
  - Inventory alignment and forecasting

### 🧪 Notes & Best Practices
- Part numbers should be cleaned (condensed) before entering the `ALL_CHASSIS_SALES_RAW` table.
- If crosses are not detected due to misformatted input, reprocessing is recommended.
- `ROW_NUMBER()` ordering can be adapted to prioritize specific brands in future updates.

---

## 🔄 Suggested Workflow Integration

1. **Run `sp_CreateMasterCrossReferencesCK`**
   - Generates up-to-date cross-reference table
2. **Update raw sales data (`ALL_CHASSIS_SALES_RAW`)**
   - Clean + condense part numbers
   - Normalize dates and quantities to annualized figures
3. **Run `Update_Chassis_Rank`**
   - Produces latest region-based ranking table
4. **Connect to Power BI or MS Access**
   - Perform regional benchmarking, opportunity analysis, and strategic planning

---

These procedures form the technical foundation for transforming complex, retailer-specific part number data into a powerful, regionally segmented benchmarking tool.


### Benchmarking & Opportunity Analysis

Once normalized and aligned, each part-region pair is analyzed for:

- **Sales vs Demand Delta**  
  - Positive delta = Growth achieved  
  - Negative delta = Potential missed opportunity

- **Market Presence**  
  - Presence in catalogs without sales  
  - Sales where no catalog demand was detected (indicating possible distribution gaps)

- **Brand Positioning Check**  
  - Comparing internal product against the type and tier of competitors in that region

---

### KPIs & Pseudo Analysis

#### 🔸 1. Lost Opportunity %

(Lost Sales / Total Demand) * 100

**Example**:  
- Region: North America  
- SKU: 123456  
- Demand: 12,000 units  
- Our Sales: 4,200 units  
- **Lost Opportunity** = (12,000 - 4,200) / 12,000 = **65%**

#### 🔸 2. Brand Penetration Rate

(Our Sales / Total Regional Demand for SKUs we offer)

**Example**:  
- Region: South America  
- Total Demand: 25,000 units  
- Our Sales: 8,750 units  
- **Penetration** = 35%

#### 🔸 3. Catalog Coverage Ratio

(Unique SKUs Sold / Unique SKUs with Demand)

**Example**:  
- SKUs with demand: 950  
- SKUs we sold: 440  
- **Coverage** = 46.3%

#### 🔸 4. Fill Rate Proxy

(Sales / Min(Sales, Demand)) [capped at 100%]

**Example**:  
- SKU 789012 in Middle East  
- Demand: 3,000 units  
- Sales: 2,750 units  
- **Fill Rate** ≈ 91.7%

---

### Next Steps

1. **Automate Matching Rules**  
   - Introduce rule-based suppression for economy crosses  
   - Automate detection of mismatched or multi-matched competitor SKUs

2. **Advanced Metrics**  
   - Develop weighted metrics that account for margin, freight, and tax variables by region

3. **Gap-Focused Alerts**  
   - Flag parts with demand but no sales in a region for follow-up by regional managers

4. **Improve Forecast Validation**  
   - Compare forecasted vs actual sales to tune demand reliability per retailer

---

> *This SQL-driven benchmarking framework provides the sales and product strategy teams with a structured way to monitor performance and identify regional growth opportunities based on live market demand.*
