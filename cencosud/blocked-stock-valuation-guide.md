# Reference Guide: Blocked Stock Valuation Dataset

This document details the contents and structure of the reference file **`blocked_stock_valuation.csv`**, which lists all physical supermarket inventory currently blocked from the online website.

---

## 1. Dataset Overview
* **File Name**: `blocked_stock_valuation.csv`
* **Row Count**: **122,391 rows** (Each row represents a specific SKU-store-UOM inventory position)
* **Status Filter**: 100% of rows are marked as **`NO PUBLICAR`** (unpublished/hidden)
* **Stock Filter**: 100% of rows have a positive physical inventory balance in stores (**`HANA > 0`**)

### Column Definitions

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| **`Item_Id`** | Numeric | The unique SAP code for the product. |
| **`vtex_sku_id`** | Numeric / Text | The e-commerce identifier (VTEX SKU ID). Shows `N/A` if the product has no active e-commerce mapping. |
| **`product_name`** | Text | The commercial description of the product. Shows `Unknown Product` if the description cannot be resolved. |
| **`Location_Id`** | Text | The store code (e.g. `J514` for Costanera Center, `J508` for Portal La Reina). |
| **`unit_of_measure`** | Text | The Unit of Measure (UOM) for this specific row (e.g. `UN` for Unit, `CS` for Case, `KG` for Kilogram). |
| **`hana_qty`** | Decimal | The physical quantity of the item currently sitting on store shelves. |
| **`pub_qty`** | Decimal | The stock quantity calculated by the e-commerce engine for publication. |
| **`unit_cost_clp`** | Decimal | The SAP Weighted Average Cost (CPP - Costo Promedio Ponderado) for a single unit. |
| **`total_cost_clp`** | Decimal | The total book cost value of this inventory in the store (`hana_qty * unit_cost_clp`). |
| **`action`** | Text | The final publication decision. Always set to `NO PUBLICAR`. |
| **`case_reason`** | Text | The specific business rule/case responsible for blocking the item. |

---

## 2. Business Cases & Percentage Breakdown
The 122,391 blocked inventory rows are distributed across **12 distinct business cases**:

| # | Business Case (`case_reason`) | Row Count | % of Dataset | Business Context |
| :---: | :--- | :---: | :---: | :--- |
| 1 | **`JERARQUIA DESHABILITADA`** | `105,003` | **`85.79%`** | Product category/hierarchy is disabled online. *(Note: This can represent either a configuration gap or an intentional decision to exclude certain categories from online sale).* |
| 2 | **`LISTADO BAJA`** | `5,502` | **`4.50%`** | Product is commercially delisted or discontinued. |
| 3 | **`JERARQUIA CON STOCK REGULAR`** | `5,212` | **`4.26%`** | Stock is below minimum safety threshold (blocked to prevent overselling). |
| 4 | **`REGLA DDS`** | `3,367` | **`2.75%`** | Days of supply is below safety limits based on sales velocity. |
| 5 | **`MAESTRA MARCAS MMPP-IMPO`** | `1,029` | **`0.84%`** | Private label or imported brand exclusion rules. |
| 6 | **`JERARQUIA PRODUCCION`** | `1,026` | **`0.84%`** | Exclusions for items prepared inside the store. |
| 7 | **`MAESTRA MARCAS PRODUCCION`** | `467` | **`0.38%`** | Production brand guidelines exclusions. |
| 8 | **`PRODUCTO TIENDA MADRE`** | `451` | **`0.37%`** | Darkstore specific mother-store link rules. |
| 9 | **`LISTADO FIJO CON STOCK`** | `252` | **`0.21%`** | Active override list for in-stock items. |
| 10 | **`SET DE VENTA`** | `32` | **`0.03%`** | Combo pack / sales set exclusions. |
| 11 | **`PRODUCTO NUEVO`** | `29` | **`0.02%`** | Newly registered product rules. |
| 12 | **`PRODUCTO NO CAT EN DARKSTORE`** | `21` | **`0.02%`** | Items not categorized for darkstore fulfillment. |
| | **TOTAL** | **`122,391`** | **`100.00%`** | |

---

## 3. Explanatory Note: Why Some Products are "Unknown"

> [!NOTE]
> ### Why we have "Unknown Product" and missing VTEX IDs in the list:
> 
> * **No Names in the Source File**: The raw output of the pipeline (`source.csv`) only contains the numeric SAP `Item_Id` and has no product names, descriptions, or e-commerce identifiers.
> * **Excluded from Mappings**: To find the names, we must match the SAP ID against the active e-commerce catalog integration database. However, items that are delisted (`LISTADO BAJA`) or belong to disabled categories (`JERARQUIA DESHABILITADA`) **do not exist in the active online catalog**.
> * **No Transactional Trace**: Because these products are blocked online, they have never been purchased or ordered online. Therefore, they leave no description records in the digital order history logs either.
> * **The Result**: For these items, only the raw physical SAP ID, quantity, and cost are known from store inventories. They represent real, physical assets on the shelves, but are digitally "invisible" across all integration databases.
