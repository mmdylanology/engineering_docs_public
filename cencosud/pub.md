# PUBLICADOR CSV Case Reference Guide

* **Data Snapshot Date**: July 10, 2026
* **Data Snapshot Time**: 11:41:24 CLT
* **HANA Matching Batch**: [vw_daily_nrt_0710_1141](file:///Users/malikmubarak/Desktop/JUMBO/data_samples/vw_daily_nrt_0710_1141)
* **Total Rows in Snapshot**: `1,225,228`
* **Action Breakdown**: `836,690` Published (68.3%) / `388,538` Blocked (31.7%)

---

## Group 1: Standard Catalog Rules (Stock & Velocity Checks)
These rules manage the standard daily catalog. They check if we have physical stock and if we have enough of it to satisfy demand.

### 1.1 `JERARQUIA CON STOCK REGULAR` (Regular Stock Hierarchy)
* **PDF Stage**: Stage 2.3 & 3 (Regular Assortment)
* **Rule Logic**: Publish if Days of Supply (DDS) ≥ 2, or if Stock ≥ 3 and there are zero sales in the last 5 days. Deducts a small safety buffer (1–5 units) to prevent overselling.
* **Volume**: **`409,284` rows** (33.4% of CSV)
  * **`PUBLICAR`**: `372,616` rows (91.0%)
  * **`NO PUBLICAR`**: `36,668` rows (9.0%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`1798142`** | `J408` | `UN` | `16.0` | `13.0` | **`PUBLICAR`** | Passed thresholds. Published with a `-3` unit safety buffer. |
| **`1822185`** | `J408` | `UN` | `33.0` | `33.0` | **`PUBLICAR`** | Passed thresholds. Published with a `0` unit buffer. |
| **`1713115`** | `J501` | `UN` | `1.0` | `1.0` | **`NO PUBLICAR`** | **Blocked!** Although we have `1` unit, it is under the safety threshold. |
| **`1939702`** | `J501` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Stock is `0`. |

---

### 1.2 `JERARQUIA PRODUCCION` (Production Hierarchy)
* **PDF Stage**: Stage 2.3 (Levels 5-6 Production Assortment)
* **Rule Logic**: Applies to in-store manufactured/prepared goods (bakery, deli). The rule publishes if:
  * **Level 5**: `DDS ≥ 1` OR (`Stock_UMB ≥ 3` AND `Venta_Unidades_5D = 0`)
  * **Level 6**: `DDS ≥ 2` OR (`Stock_UMB ≥ 5` AND `Venta_Unidades_5D = 0`)
  * Also subtracts a larger safety reserve buffer (median `-2.0` units, but can be larger for high-volume batches).
* **Volume**: **`119,407` rows** (9.7% of CSV)
  * **`PUBLICAR`**: `111,712` rows (93.6%)
  * **`NO PUBLICAR`**: `7,695` rows (6.4%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`294230`** | `J408` | `UN` | `37.0` | `33.0` | **`PUBLICAR`** | Published with a `-4` unit production safety buffer. |
| **`1214552`** | `J408` | `UN` | `384.0` | `379.0` | **`PUBLICAR`** | Published with a `-5` unit production safety buffer. |
| **`1577053`** | `J748` | `UN` | `12.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Deactivated in production schedule. |
| **`1862247`** | `J748` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Stock is `0`. |

---

### 1.3 `REGLA DDS` (Days of Supply Revalidation)
* **PDF Stage**: Stage 7 (DDS Revalidation)
* **Rule Logic**: A risk-control rule near the end of the pipeline. If an item has low stock compared to its recent sales velocity, it will block it or reduce the quantity to prevent unfulfillable orders.
* **Volume**: **`270,743` rows** (22.1% of CSV)
  * **`PUBLICAR`**: `216,472` rows (80.0%)
  * **`NO PUBLICAR`**: `54,271` rows (20.0%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`296634`** | `J408` | `UN` | `30.0` | `28.0` | **`PUBLICAR`** | Safe stock levels. Published with a `-2` unit reserve. |
| **`1952600`** | `J408` | `UN` | `2.0` | `9.0` | **`PUBLICAR`** | **Darkstore case!** Quantity inflated due to mother store stock. |
| **`1803090`** | `J501` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** No stock available. |
| **`1967227`** | `J501` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** No stock available. |

---

### 1.4 `PRODUCTO NUEVO` (New Product)
* **PDF Stage**: Stage 2.1 (New Items < 90 Days)
* **Rule Logic**: New items in the system. Publish only if stock > 0. Unlike regular items, no safety buffers are deducted.
* **Volume**: **`20,884` rows** (1.7% of CSV)
  * **`PUBLICAR`**: `10,352` rows (49.6%)
  * **`NO PUBLICAR`**: `10,532` rows (50.4%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`2075054`** | `J519` | `UN` | `37.0` | `37.0` | **`PUBLICAR`** | New item with stock. Published at exact quantity (`37`). |
| **`2081581`** | `J519` | `UN` | `1543.0` | `1541.0` | **`PUBLICAR`** | New item with stock (minimal safety reserve applied). |
| **`2073068`** | `J502` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Stock is `0`. |
| **`2079841`** | `J502` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Stock is `0`. |

---

### 1.5 `SET DE VENTA` (Sales Bundle / Kit)
* **PDF Stage**: Stage 2.2 (Bundles)
* **Rule Logic**: Multi-item bundle packs. Published only if all component items are available.
* **Volume**: **`1,043` rows** (0.1% of CSV)
  * **`PUBLICAR`**: `601` rows (57.6%)
  * **`NO PUBLICAR`**: `442` rows (42.4%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`2070613`** | `J519` | `KG` | `26.0` | `26.0` | **`PUBLICAR`** | Combo pack where all items have stock. Published at `26`. |
| **`2087097`** | `J519` | `UN` | `6.0` | `6.0` | **`PUBLICAR`** | Combo pack where all items have stock. Published at `6`. |
| **`1926545`** | `J762` | `KG` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Components are out of stock. |
| **`1930894`** | `J762` | `KG` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Components are out of stock. |

---

## Group 2: The Business Overrides (Force-Publish or Force-Block)
These rules represent deliberate business interventions that override normal stock levels.

### 2.1 `LISTADO BAJA` (Cancellation List)
* **PDF Stage**: Stage 2.7 (Delisted Assortment Override)
* **Rule Logic**: The absolute highest priority block rule. Delisted or discontinued items are **always blocked** from e-commerce, even if store shelves physically have stock.
* **Volume**: **`8,216` rows** (0.7% of CSV)
  * **`PUBLICAR`**: `0` rows (0.0%)
  * **`NO PUBLICAR`**: `8,216` rows (100.0%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`1406197`** | `J660` | `UN` | `16.0` | `16.0` | **`NO PUBLICAR`** | **Blocked!** Even with `16` units physically in stock. |
| **`1998709`** | `J660` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Discontinued and out of stock. |

---

### 2.2 `LISTADO FIJO SIN STOCK` (Fixed Assortment — Force Publish)
* **PDF Stage**: Stage 2.5 (Fixed List Override)
* **Rule Logic**: Applied to critical everyday staples (e.g. milk, eggs, bread). This rule **always publishes** the item so it stays visible online, even if the store has `0` or negative physical stock in HANA.
* **Volume**: **`14,407` rows** (1.2% of CSV)
  * **`PUBLICAR`**: `14,407` rows (100.0%)
  * **`NO PUBLICAR`**: `0` rows (0.0%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`2055763`** | `J502` | `UN` | `0.0` | `0.0` | **`PUBLICAR`** | Force-published online with a placeholder quantity. |
| **`1960849`** | `J502` | `KG` | `-9.8` | `-9.8` | **`PUBLICAR`** | Force-published online despite negative physical inventory. |

---

### 2.3 `LISTADO FIJO CON STOCK` (Fixed Assortment — Standard)
* **PDF Stage**: Stage 2.5 (Fixed List with Stock)
* **Rule Logic**: Critical staple items that currently have healthy physical stock in the store.
* **Volume**: **`10,887` rows** (0.9% of CSV)
  * **`PUBLICAR`**: `8,736` rows (80.2%)
  * **`NO PUBLICAR`**: `2,151` rows (19.8%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`1995451`** | `J501` | `UN` | `18.0` | `18.0` | **`PUBLICAR`** | Critical staple. Published with normal stock. |
| **`2020063`** | `J501` | `UN` | `19.0` | `19.0` | **`PUBLICAR`** | Critical staple. Published with normal stock. |
| **`1289602`** | `J748` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Evaluated under active stock constraints. |

---

### 2.4 `PROD CON VENTA` (Sales Rescue Override)
* **PDF Stage**: Stage 8 (Recent Sales Override)
* **Rule Logic**: If HANA reports `0` or negative stock but a store cash register recorded a physical sale in the last 24 hours, the system rescues the item and force-publishes it because physical stock must be present.
* **Volume**: **`1,503` rows** (0.1% of CSV)
  * **`PUBLICAR`**: `1,503` rows (100.0%)
  * **`NO PUBLICAR`**: `0` rows (0.0%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`295964`** | `J519` | `KG` | `-24.3` | `-29.3` | **`PUBLICAR`** | **Rescued!** Negative stock in HANA, but active sales registered. |
| **`1898914`** | `J519` | `UN` | `4.0` | `-1.0` | **`PUBLICAR`** | **Rescued!** Force-published with a low quantity buffer. |

---

### 2.5 `CASO FECHA HORA` (Time Override)
* **PDF Stage**: Stage 4 (Date/Time Specific Override)
* **Rule Logic**: Items that are force-published during specific hours (like hot roast chicken at lunch time or bakery items in the morning).
* **Volume**: **`229` rows** (0.02% of CSV)
  * **`PUBLICAR`**: `229` rows (100.0%)
  * **`NO PUBLICAR`**: `0` rows (0.0%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`254605`** | `J501` | `KG` | `8.7` | `8.7` | **`PUBLICAR`** | Prepared hot food item force-published during active hours. |
| **`254422`** | `J501` | `KG` | `7.3` | `7.3` | **`PUBLICAR`** | Prepared bakery item force-published during active hours. |

---

## Group 3: Private Labels & Imports

### 3.1 `MAESTRA MARCAS MMPP-IMPO` (Private Label / Imports)
* **PDF Stage**: Stage 2.4 (Brand Master — Own/Imported)
* **Rule Logic**: Specific stock check rules applied to Cencosud's private label (*Cuisine & Co*) or imported grocery lines.
* **Volume**: **`67,961` rows** (5.5% of CSV)
  * **`PUBLICAR`**: `55,192` rows (81.2%)
  * **`NO PUBLICAR`**: `12,769` rows (18.8%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`2042446`** | `J408` | `UN` | `295.0` | `295.0` | **`PUBLICAR`** | Safe stock levels. Published at `295`. |
| **`925249`** | `J408` | `UN` | `15.0` | `15.0` | **`PUBLICAR`** | Safe stock levels. Published at `15`. |
| **`1905800`** | `J502` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Out of stock. |

---

### 3.2 `MAESTRA MARCAS PRODUCCION` (Brand Production)
* **PDF Stage**: Stage 2.4 (Brand Master — In-Store Prepared)
* **Rule Logic**: Private label items that are manufactured inside the stores (bakery, pastry, or hot deli).
* **Volume**: **`27,738` rows** (2.3% of CSV)
  * **`PUBLICAR`**: `24,189` rows (87.2%)
  * **`NO PUBLICAR`**: `3,549` rows (12.8%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`2025537`** | `J501` | `UN` | `121.0` | `121.0` | **`PUBLICAR`** | Brand production item with healthy stock. |
| **`2052288`** | `J501` | `UN` | `24.0` | `24.0` | **`PUBLICAR`** | Brand production item with healthy stock. |
| **`2054729`** | `J501` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Out of stock. |

---

## Group 4: Darkstore & Catalog Mappings

### 4.1 `PRODUCTO TIENDA MADRE` (Mother Store Stock)
* **PDF Stage**: Stage 5 (Darkstore Stock Inheritance)
* **Rule Logic**: Darkstores inherit physical stock from their parent physical stores.
* **Volume**: **`30,390` rows** (2.5% of CSV)
  * **`PUBLICAR`**: `20,681` rows (68.1%)
  * **`NO PUBLICAR`**: `9,709` rows (31.9%)

#### Real Examples (Darkstore J414 inheriting from Mother J514):
| Item ID | Store | UoM | HANA Qty (J414) | HANA Qty (J514) | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :---: | :--- | :--- |
| **`113741`** | `J414` | `UN` | `0.0` | `42.0` | `42.0` | **`PUBLICAR`** | J414 inherits J514's stock (`42`) and publishes it. |
| **`2037234`** | `J414` | `UN` | `0.0` | `0.0` | `0.0` | **`NO PUBLICAR`** | Both are out of stock. Nothing to inherit. |
| **`1968067`** | `J414` | `UN` | `0.0` | `31.0` | `31.0` | **`NO PUBLICAR`** | **Blocked!** Stock inherited, but J414's catalog de-activated it. |

---

### 4.2 `PRODUCTO NO CAT EN DARKSTORE` (Not Catalogued in Darkstore)
* **PDF Stage**: Stage 5 (Darkstore Assortment Validation)
* **Rule Logic**: The system inherits stock from the mother store into the darkstore, but since the darkstore e-commerce catalog doesn't support the item, it is blocked.
* **Volume**: **`1,315` rows** (0.1% of CSV)
  * **`PUBLICAR`**: `0` rows (0.0%)
  * **`NO PUBLICAR`**: `1,315` rows (100.0%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty (J408) | HANA Qty (J508) | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :---: | :--- | :--- |
| **`2008069`** | `J408` | `UN` | `0.0` | `8.0` | `8.0` | **`NO PUBLICAR`** | **Blocked!** Inherits `8` units, but not catalogued in J408. |
| **`1929091`** | `J408` | `UN` | `0.0` | `66.0` | `66.0` | **`NO PUBLICAR`** | **Blocked!** Inherits `66` units, but not catalogued in J408. |

---

### 4.3 `JERARQUIA DESHABILITADA` (Disabled Catalog / Hierarchy)
* **PDF Stage**: Stage 3.8 (Disabled Assortment)
* **Rule Logic**: The catch-all block at the bottom of the priority cascade. If a product has stock on shelves but is missing its category mapping configuration in the database, the system blocks it.
* **Volume**: **`241,221` rows** (19.7% of CSV)
  * **`PUBLICAR`**: `0` rows (0.0%)
  * **`NO PUBLICAR`**: `241,221` rows (100.0%)

#### Real Examples:
| Item ID | Store | UoM | HANA Qty | PUBLICADOR Qty | Action | Explanation |
| :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **`2002373`** | `J519` | `UN` | `0.0` | `0.0` | **`NO PUBLICAR`** | Out of stock and disabled. |
| **`2046215`** | `J519` | `UN` | `15.0` | `0.0` | **`NO PUBLICAR`** | **Blocked!** Item has `15` units in stock but is missing catalog mapping. |

> [!IMPORTANT]
> **This rule is the primary driver of false OOS online**: It currently blocks **`105,003` active store-item pairs** representing **`1,875,704` physical stock units** in HANA simply due to missing hierarchy configuration in the database!
