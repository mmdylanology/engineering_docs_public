# Inventory Status & Blocked Stock Valuation Report

This report evaluates the financial value of JUMBO supermarket items that are physically in stock on store shelves but blocked from being sold online due to system catalog and publication rules.

---

## 1. Executive Summary
An audit of the **July 10, 2026, 11:41 CLT** inventory snapshot across **41 physical stores** reveals the volume of shelf inventory that is marked as `NO PUBLICAR` (unpublished/hidden) in the e-commerce files:

* **Total Blocked Store-SKU Records**: **122,213 unique combinations** (deduplicated by Unit of Measure)
* **Total Blocked Physical Units on Shelves**: **2,239,455 units**
* **Total Value (Cost of Goods)**: **`6,858,883,171 CLP`** (Approx. **`$7,455,308 USD`**)
* **Disabled Category Blocks (`JERARQUIA DESHABILITADA`)**: **`5,805,498,615 CLP`** (Approx. **`$6,310,325 USD`**), representing **`84.6%`** of all blocked inventory capital.

---

## 2. Methodology & Calculations

### Data Matching Pipeline
1. **Scope Filter**: Filtered the PUBLICADOR CSV output file for all rows marked as **`NO PUBLICAR`** (unpublished/hidden).
2. **Deduplication**: Some SKUs contain multiple rows in the e-commerce files for different packaging formats (e.g. `CS` for Case, `PAK` for Pack, `UN` for Unit). To prevent multiplying the physical stock count, the publicador file was grouped by `(Item_Id, Location_Id)` and filtered only if **all** packaging formats were marked as unpublished.
3. **ERP Join**: Joined the deduplicated list with the SAP HANA physical inventory parquets matching store code and zero-padding the unpadded Item ID:
   `LPAD(p.Item_Id, 18, '0') = h.Item_Id AND p.Location_Id = h.Location_Id`
4. **Active Stock Filter**: Retained only rows where physical inventory on hand was positive: `Ending_On_Hand_Qty > 0`.

### Valuation Math (How the numbers are calculated)
We valued the blocked inventory using the **Weighted Average Cost (CPP - Costo Promedio Ponderado)** recorded by the ERP system in SAP. This represents the book value of the stock.

```
Blocked Inventory Value (CLP) = SUM(Ending_On_Hand_CPP_Cost_Amt)
```

Which is mathematically equivalent to:

```
Blocked Inventory Value (CLP) = SUM(Ending_On_Hand_Qty * Item_CPP_Cost_Amt)
```

*Note: USD conversion is calculated using a standard reporting exchange rate of **`1 USD = 920 CLP`**.*

---

## 3. Financial Breakdown by Motive Rule (Case)
Grouping the blocked inventory by its rule reason (`caso`) distinguishes the different catalog filters applied:

| Motive Rule Case (`caso`) | Blocked Rows | Blocked Value (CLP) | Blocked Value (USD) | % of Block | Database Description & Business Context |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **`JERARQUIA DESHABILITADA`** | 105,003 | 5,805,498,615 CLP | $6,310,325 USD | 84.64% | The product belongs to a category/hierarchy node that is currently disabled in the e-commerce publication mappings. *(Note: This can represent either a configuration gap or an intentional decision to exclude certain categories from online sale).* |
| **`LISTADO BAJA`** | 5,497 | 897,161,934 CLP | $975,176 USD | 13.08% | Product is commercially delisted or discontinued. |
| **`REGLA DDS`** | 3,320 | 46,546,893 CLP | $50,594 USD | 0.68% | Blocked because days of supply is below safety levels. |
| **`JERARQUIA CON STOCK REGULAR`** | 5,143 | 44,365,533 CLP | $48,223 USD | 0.65% | Blocked because physical stock is below the minimum safety threshold. |
| *Other Minor Rules combined* | 1,250 | 65,310,201 CLP | $70,989 USD | 0.95% | Private label blocks, sets, and new product rules. |
| **TOTAL** | **122,213** | **6,858,883,171 CLP** | **$7,455,308 USD** | **100.00%** | |

---

## 4. Key Business Assumptions (Requires Validation)

> [!WARNING]
> ### Crucial Business Assumptions to Flag
>
> 1. **Intentional vs. Unintentional Exclusion (`JERARQUIA DESHABILITADA`)**: 
>    We cannot assume that all disabled categories are "errors" or "opportunity loss." It is common for certain categories (e.g., fresh rotisserie items, heavy bulk goods, temperature-sensitive items, or regulatory restricted products) to be sold in physical stores but intentionally excluded from the online catalog. 
>    * **Required Action**: The commercial and catalog teams must audit this list to identify which of the 105,003 disabled category rows are intentional exclusions vs. actual configuration gaps.
>
> 2. **Delisted Stock Liquidation (`LISTADO BAJA`)**:
>    Products marked as delisted or discontinued are hidden from e-commerce but may still occupy physical shelf space. 
>    * **Required Action**: Validate if there is a store clearing process to liquidate this inventory or if a digital clearance mechanism could be used.

---

## 5. Reference Artifacts & Data Export
* **Exhaustive Blocked Inventory Export**: [blocked_stock_valuation.csv](file:///Users/malikmubarak/Desktop/JUMBO/analysis/blocked_stock_valuation.csv)
  * This CSV contains all `122,213` deduplicated rows. It can be opened in Excel and contains the resolved product name, SAP ID, VTEX SKU ID, Store, Qty, Unit Cost, and Case Reason.
* **Permutations Reference Guide**: [permutations.pub.md](file:///Users/malikmubarak/Desktop/JUMBO/permutations.pub.md)
  * This Markdown file contains the detailed definition and drill-downs (A, B, and C) with product examples for each of the 21 database inventory states.
