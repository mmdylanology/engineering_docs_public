# PUBLICADOR Inventory & Action Permutations Guide

This document maps all **21 active permutations** of physical inventory states and e-commerce publication decisions found in the July 10, 11:41 CLT snapshot.

## Pipeline Variables
Each row in the PUBLICADOR output is determined by three variables:
1. **HANA State**: The physical inventory level reported by SAP HANA (`HANA > 0`, `HANA = 0`, `HANA < 0`, or `HANA NULL` if the record is missing).
2. **PUB State**: The display inventory level calculated for the website (`PUB > 0`, `PUB = 0`, or `PUB < 0` after safety buffers).
3. **Action**: The publication decision sent to VTEX (`PUBLICAR` or `NO PUBLICAR`).

---

## The 21 Permutations Summary Table

| # | HANA State | PUB State | Action | Row Count (%) | All Matching Cases |
| :--- | :--- | :--- | :--- | :---: | :--- |
| 1 | `> 0` | `> 0` | **`PUBLICAR`** | `799,781 (65.28%)` | JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, PRODUCTO NUEVO, CASO FECHA HORA, MAESTRA MARCAS MMPP-IMPO, SET DE VENTA, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, PROD CON VENTA, MAESTRA MARCAS PRODUCCION, REGLA DDS, LISTADO FIJO CON STOCK |
| 2 | `= 0` | `= 0` | **`NO PUBLICAR`** | `245,589 (20.04%)` | SET DE VENTA, MAESTRA MARCAS PRODUCCION, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, JERARQUIA PRODUCCION, REGLA DDS, PRODUCTO NUEVO, LISTADO FIJO CON STOCK, PRODUCTO NO CAT EN DARKSTORE, MAESTRA MARCAS MMPP-IMPO, JERARQUIA DESHABILITADA, LISTADO BAJA |
| 3 | `> 0` | `> 0` | **`NO PUBLICAR`** | `121,570 (9.92%)` | MAESTRA MARCAS PRODUCCION, JERARQUIA DESHABILITADA, MAESTRA MARCAS MMPP-IMPO, REGLA DDS, LISTADO BAJA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, LISTADO FIJO CON STOCK, SET DE VENTA, JERARQUIA PRODUCCION, PRODUCTO NO CAT EN DARKSTORE |
| 4 | `= 0` | `> 0` | **`PUBLICAR`** | `25,011 (2.04%)` | PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE |
| 5 | `< 0` | `< 0` | **`NO PUBLICAR`** | `14,347 (1.17%)` | SET DE VENTA, MAESTRA MARCAS PRODUCCION, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, JERARQUIA PRODUCCION, REGLA DDS, PRODUCTO NUEVO, LISTADO FIJO CON STOCK, MAESTRA MARCAS MMPP-IMPO, JERARQUIA DESHABILITADA, LISTADO BAJA |
| 6 | `= 0` | `= 0` | **`PUBLICAR`** | `3,844 (0.31%)` | CASO FECHA HORA, PROD CON VENTA, LISTADO FIJO SIN STOCK, SET DE VENTA, PRODUCTO TIENDA MADRE, REGLA DDS |
| 7 | `= 0` | `> 0` | **`NO PUBLICAR`** | `3,511 (0.29%)` | PRODUCTO TIENDA MADRE, REGLA DDS, PRODUCTO NO CAT EN DARKSTORE |
| 8 | `< 0` | `< 0` | **`PUBLICAR`** | `2,813 (0.23%)` | CASO FECHA HORA, PROD CON VENTA, LISTADO FIJO SIN STOCK, SET DE VENTA, PRODUCTO TIENDA MADRE |
| 9 | `> 0` | `= 0` | **`PUBLICAR`** | `2,305 (0.19%)` | JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, SET DE VENTA, PRODUCTO NUEVO, MAESTRA MARCAS MMPP-IMPO, REGLA DDS, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, PROD CON VENTA, MAESTRA MARCAS PRODUCCION |
| 10 | `< 0` | `> 0` | **`PUBLICAR`** | `2,150 (0.18%)` | REGLA DDS, PRODUCTO TIENDA MADRE, PROD CON VENTA |
| 11 | `NULL` | `= 0` | **`NO PUBLICAR`** | `1,437 (0.12%)` | JERARQUIA PRODUCCION, PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA DESHABILITADA, LISTADO BAJA, SET DE VENTA |
| 12 | `= 0` | `< 0` | **`NO PUBLICAR`** | `747 (0.06%)` | PRODUCTO NUEVO, PRODUCTO NO CAT EN DARKSTORE, LISTADO FIJO CON STOCK, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, REGLA DDS, JERARQUIA PRODUCCION, MAESTRA MARCAS MMPP-IMPO, MAESTRA MARCAS PRODUCCION, LISTADO BAJA |
| 13 | `= 0` | `< 0` | **`PUBLICAR`** | `504 (0.04%)` | CASO FECHA HORA, PROD CON VENTA, LISTADO FIJO SIN STOCK, REGLA DDS, PRODUCTO TIENDA MADRE |
| 14 | `> 0` | `= 0` | **`NO PUBLICAR`** | `492 (0.04%)` | JERARQUIA PRODUCCION, PRODUCTO NO CAT EN DARKSTORE, MAESTRA MARCAS MMPP-IMPO, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, REGLA DDS, PRODUCTO NUEVO, LISTADO BAJA, MAESTRA MARCAS PRODUCCION, JERARQUIA DESHABILITADA |
| 15 | `NULL` | `> 0` | **`NO PUBLICAR`** | `372 (0.03%)` | PRODUCTO NO CAT EN DARKSTORE |
| 16 | `> 0` | `< 0` | **`NO PUBLICAR`** | `329 (0.03%)` | PRODUCTO NUEVO, REGLA DDS, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, MAESTRA MARCAS PRODUCCION, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION |
| 17 | `> 0` | `< 0` | **`PUBLICAR`** | `207 (0.02%)` | PROD CON VENTA, LISTADO FIJO SIN STOCK, PRODUCTO TIENDA MADRE |
| 18 | `< 0` | `= 0` | **`PUBLICAR`** | `75 (0.01%)` | PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE |
| 19 | `< 0` | `= 0` | **`NO PUBLICAR`** | `69 (0.01%)` | REGLA DDS, MAESTRA MARCAS MMPP-IMPO, SET DE VENTA, JERARQUIA PRODUCCION, PRODUCTO TIENDA MADRE |
| 20 | `< 0` | `> 0` | **`NO PUBLICAR`** | `66 (0.01%)` | PRODUCTO TIENDA MADRE, REGLA DDS, PRODUCTO NO CAT EN DARKSTORE |
| 21 | `NULL` | `< 0` | **`NO PUBLICAR`** | `9 (0.00%)` | PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE |

---

## Detailed Permutations Analysis

### Permutation 1: `> 0` | `> 0` | `PUBLICAR`
* **Volume**: `799,781 rows` (65.28%)
* **All Matching Cases**: JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, PRODUCTO NUEVO, CASO FECHA HORA, MAESTRA MARCAS MMPP-IMPO, SET DE VENTA, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, PROD CON VENTA, MAESTRA MARCAS PRODUCCION, REGLA DDS, LISTADO FIJO CON STOCK
* **Commercial Meaning**: Standard happy path where the item is physically in stock, the PUBLICADOR calculated quantity is positive, and it is published.
* **Rule Exceptions**: None. This is the normal flow.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1961117001`
* **VTEX SKU ID**: `126809`
* **Product Name**: `Yogurt Kéfir Colun Natural 120 g`
* **Store**: `J796`
* **HANA Qty**: `139.0`
* **PUBLICADOR Qty**: `139.0`
* **Example Specific Case (`caso`)**: `*JERARQUIA CON STOCK REGULAR*`

---
### Permutation 2: `= 0` | `= 0` | `NO PUBLICAR`
* **Volume**: `245,589 rows` (20.04%)
* **All Matching Cases**: SET DE VENTA, MAESTRA MARCAS PRODUCCION, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, JERARQUIA PRODUCCION, REGLA DDS, PRODUCTO NUEVO, LISTADO FIJO CON STOCK, PRODUCTO NO CAT EN DARKSTORE, MAESTRA MARCAS MMPP-IMPO, JERARQUIA DESHABILITADA, LISTADO BAJA
* **Commercial Meaning**: Standard out-of-stock behavior. Shelf stock is zero, so display stock is zero and it is blocked.
* **Rule Exceptions**: None.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1897968`
* **VTEX SKU ID**: `106770`
* **Product Name**: `Colado Orgánico Esencial Pera y Mango 90 g`
* **Store**: `J408`
* **HANA Qty**: `0.0`
* **PUBLICADOR Qty**: `0.0`
* **Example Specific Case (`caso`)**: `*LISTADO BAJA*`

---
### Permutation 3: `> 0` | `> 0` | `NO PUBLICAR`
* **Volume**: `121,570 rows` (9.92%)
* **All Matching Cases**: MAESTRA MARCAS PRODUCCION, JERARQUIA DESHABILITADA, MAESTRA MARCAS MMPP-IMPO, REGLA DDS, LISTADO BAJA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, LISTADO FIJO CON STOCK, SET DE VENTA, JERARQUIA PRODUCCION, PRODUCTO NO CAT EN DARKSTORE
* **Commercial Meaning**: Stock exists physically and is positive, but the catalog rule decides to block it (Action: NO PUBLICAR). This is the key leakage state.
* **Rule Exceptions**: JERARQUIA DESHABILITADA (missing database category config) is the primary driver here, blocking 1.8M units. Other rules include LISTADO BAJA (delisted items).

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1393540`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J519`
* **HANA Qty**: `3.0`
* **PUBLICADOR Qty**: `3.0`
* **Example Specific Case (`caso`)**: `*JERARQUIA DESHABILITADA*`

---
### Permutation 4: `= 0` | `> 0` | `PUBLICAR`
* **Volume**: `25,011 rows` (2.04%)
* **All Matching Cases**: PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE
* **Commercial Meaning**: Darkstore stock inheritance. The darkstore physically has 0 stock locally, but inherits positive stock from its assigned mother store.
* **Rule Exceptions**: PRODUCTO TIENDA MADRE performs a direct copy override, not an addition. If J514 has 34 and J414 has 0, the result is 34.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `2017978`
* **VTEX SKU ID**: `145718`
* **Product Name**: `Cepillo de Pelo It Cuadrado Grande Rubber Azul`
* **Store**: `J408`
* **HANA Qty**: `0.0`
* **PUBLICADOR Qty**: `25.0`
* **Example Specific Case (`caso`)**: `*PRODUCTO TIENDA MADRE*`

---
### Permutation 5: `< 0` | `< 0` | `NO PUBLICAR`
* **Volume**: `14,347 rows` (1.17%)
* **All Matching Cases**: SET DE VENTA, MAESTRA MARCAS PRODUCCION, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, JERARQUIA PRODUCCION, REGLA DDS, PRODUCTO NUEVO, LISTADO FIJO CON STOCK, MAESTRA MARCAS MMPP-IMPO, JERARQUIA DESHABILITADA, LISTADO BAJA
* **Commercial Meaning**: Negative stock clamping. HANA stock is negative due to administrative shrinkage logs. The system clamps it to zero and blocks it.
* **Rule Exceptions**: Safety buffers can make the quantity even more negative in the output, but the action is forced to NO PUBLICAR.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `2055551`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J408`
* **HANA Qty**: `-1.0`
* **PUBLICADOR Qty**: `-1.0`
* **Example Specific Case (`caso`)**: `*MAESTRA MARCAS PRODUCCION*`

---
### Permutation 6: `= 0` | `= 0` | `PUBLICAR`
* **Volume**: `3,844 rows` (0.31%)
* **All Matching Cases**: CASO FECHA HORA, PROD CON VENTA, LISTADO FIJO SIN STOCK, SET DE VENTA, PRODUCTO TIENDA MADRE, REGLA DDS
* **Commercial Meaning**: Zero stock staple rescue. HANA stock is zero, but the business force-publishes it online with zero or a placeholder so customers can still view it.
* **Rule Exceptions**: LISTADO FIJO SIN STOCK and PROD CON VENTA are the only cases that can bypass zero stock to publish.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `2055107`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J660`
* **HANA Qty**: `0.0`
* **PUBLICADOR Qty**: `0.0`
* **Example Specific Case (`caso`)**: `*LISTADO FIJO SIN STOCK*`

---
### Permutation 7: `= 0` | `> 0` | `NO PUBLICAR`
* **Volume**: `3,511 rows` (0.29%)
* **All Matching Cases**: PRODUCTO TIENDA MADRE, REGLA DDS, PRODUCTO NO CAT EN DARKSTORE
* **Commercial Meaning**: Darkstore catalog override. The darkstore inherits positive stock from its mother store, but is blocked from sale locally by catalog rules.
* **Rule Exceptions**: PRODUCTO NO CAT EN DARKSTORE is the main case, representing products not enabled in the darkstore web catalog.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1615787`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J403`
* **HANA Qty**: `0.0`
* **PUBLICADOR Qty**: `9.0`
* **Example Specific Case (`caso`)**: `*PRODUCTO TIENDA MADRE*`

---
### Permutation 8: `< 0` | `< 0` | `PUBLICAR`
* **Volume**: `2,813 rows` (0.23%)
* **All Matching Cases**: CASO FECHA HORA, PROD CON VENTA, LISTADO FIJO SIN STOCK, SET DE VENTA, PRODUCTO TIENDA MADRE
* **Commercial Meaning**: Negative stock staple rescue. HANA stock is negative, but the item is force-published because it is a critical everyday staple.
* **Rule Exceptions**: LISTADO FIJO SIN STOCK force-publishes these items, preserving the negative quantity in the file to show active sync.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `2050148`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J501`
* **HANA Qty**: `-2.0`
* **PUBLICADOR Qty**: `-2.0`
* **Example Specific Case (`caso`)**: `*LISTADO FIJO SIN STOCK*`

---
### Permutation 9: `> 0` | `= 0` | `PUBLICAR`
* **Volume**: `2,305 rows` (0.19%)
* **All Matching Cases**: JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, SET DE VENTA, PRODUCTO NUEVO, MAESTRA MARCAS MMPP-IMPO, REGLA DDS, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, PROD CON VENTA, MAESTRA MARCAS PRODUCCION
* **Commercial Meaning**: Low stock buffer clamping. Stock is positive but very low (e.g. 1 unit). A safety buffer (e.g. -1) reduces the display quantity to exactly 0, but the item remains published.
* **Rule Exceptions**: Under JERARQUIA CON STOCK REGULAR, if stock is 1 and average daily sales is 0, it publishes with exactly 0 after buffer. This is a rare edge case.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1237163`
* **VTEX SKU ID**: `633`
* **Product Name**: `Jugo en Polvo Livean Maracuyá 7 g`
* **Store**: `J748`
* **HANA Qty**: `310.0`
* **PUBLICADOR Qty**: `0.0`
* **Example Specific Case (`caso`)**: `*JERARQUIA PRODUCCION*`

---
### Permutation 10: `< 0` | `> 0` | `PUBLICAR`
* **Volume**: `2,150 rows` (0.18%)
* **All Matching Cases**: REGLA DDS, PRODUCTO TIENDA MADRE, PROD CON VENTA
* **Commercial Meaning**: Negative darkstore inheritance. The darkstore local stock is negative in HANA, but it inherits positive stock from its mother store and is published.
* **Rule Exceptions**: PRODUCTO TIENDA MADRE overrides the local negative stock with the mother's positive stock.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1865679`
* **VTEX SKU ID**: `100186`
* **Product Name**: `Snack Perro Churu Pollo y Atún 56 g`
* **Store**: `J414`
* **HANA Qty**: `-3.0`
* **PUBLICADOR Qty**: `68.0`
* **Example Specific Case (`caso`)**: `*REGLA DDS*`

---
### Permutation 11: `NULL` | `= 0` | `NO PUBLICAR`
* **Volume**: `1,437 rows` (0.12%)
* **All Matching Cases**: JERARQUIA PRODUCCION, PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA DESHABILITADA, LISTADO BAJA, SET DE VENTA
* **Commercial Meaning**: Missing HANA inventory record. The SKU is in the store catalog, but there are no physical stock records in HANA. Clamped to zero and blocked.
* **Rule Exceptions**: JERARQUIA DESHABILITADA and LISTADO BAJA apply here for discontinued or unmapped items with no inventory footprint.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `2052664`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J502`
* **HANA Qty**: `NULL`
* **PUBLICADOR Qty**: `0.0`
* **Example Specific Case (`caso`)**: `*JERARQUIA DESHABILITADA*`

---
### Permutation 12: `= 0` | `< 0` | `NO PUBLICAR`
* **Volume**: `747 rows` (0.06%)
* **All Matching Cases**: PRODUCTO NUEVO, PRODUCTO NO CAT EN DARKSTORE, LISTADO FIJO CON STOCK, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, REGLA DDS, JERARQUIA PRODUCCION, MAESTRA MARCAS MMPP-IMPO, MAESTRA MARCAS PRODUCCION, LISTADO BAJA
* **Commercial Meaning**: Zero stock with negative buffer block. Physical stock is zero, and a safety buffer (e.g. -1) makes the display stock negative. Blocked.
* **Rule Exceptions**: Normal out-of-stock items under JERARQUIA CON STOCK REGULAR.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1962096`
* **VTEX SKU ID**: `126596`
* **Product Name**: `Cerveza Quilmes Lager 4.9° 300 cc`
* **Store**: `J660`
* **HANA Qty**: `0.0`
* **PUBLICADOR Qty**: `-2.0`
* **Example Specific Case (`caso`)**: `*REGLA DDS*`

---
### Permutation 13: `= 0` | `< 0` | `PUBLICAR`
* **Volume**: `504 rows` (0.04%)
* **All Matching Cases**: CASO FECHA HORA, PROD CON VENTA, LISTADO FIJO SIN STOCK, REGLA DDS, PRODUCTO TIENDA MADRE
* **Commercial Meaning**: Zero stock with negative buffer rescue. Physical stock is zero, buffer makes it negative, but force-published due to staple status.
* **Rule Exceptions**: LISTADO FIJO SIN STOCK and PROD CON VENTA force-publish the item with the negative quantity.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1965049`
* **VTEX SKU ID**: `139536`
* **Product Name**: `Corona Jumbo un`
* **Store**: `J633`
* **HANA Qty**: `0.0`
* **PUBLICADOR Qty**: `-3.0`
* **Example Specific Case (`caso`)**: `*LISTADO FIJO SIN STOCK*`

---
### Permutation 14: `> 0` | `= 0` | `NO PUBLICAR`
* **Volume**: `492 rows` (0.04%)
* **All Matching Cases**: JERARQUIA PRODUCCION, PRODUCTO NO CAT EN DARKSTORE, MAESTRA MARCAS MMPP-IMPO, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, REGLA DDS, PRODUCTO NUEVO, LISTADO BAJA, MAESTRA MARCAS PRODUCCION, JERARQUIA DESHABILITADA
* **Commercial Meaning**: Clamped to zero block. Low stock (e.g. 1 unit) has a buffer reduce it to 0, which triggers an active block.
* **Rule Exceptions**: JERARQUIA CON STOCK REGULAR and REGLA DDS block the item because the stock falls below the minimum safety threshold.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `263813`
* **VTEX SKU ID**: `21727`
* **Product Name**: `Vino Jerez Fino Tío Pepe Muy Seco 750 cc`
* **Store**: `J502`
* **HANA Qty**: `1.0`
* **PUBLICADOR Qty**: `0.0`
* **Example Specific Case (`caso`)**: `*JERARQUIA PRODUCCION*`

---
### Permutation 15: `NULL` | `> 0` | `NO PUBLICAR`
* **Volume**: `372 rows` (0.03%)
* **All Matching Cases**: PRODUCTO NO CAT EN DARKSTORE
* **Commercial Meaning**: Darkstore catalog block (No HANA). The darkstore has no stock record in HANA, inherits positive stock from the mother store, but local catalog blocks it.
* **Rule Exceptions**: PRODUCTO NO CAT EN DARKSTORE blocks it because the item is deactivated in the darkstore storefront.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `2052200`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J403`
* **HANA Qty**: `NULL`
* **PUBLICADOR Qty**: `5.0`
* **Example Specific Case (`caso`)**: `*PRODUCTO NO CAT EN DARKSTORE*`

---
### Permutation 16: `> 0` | `< 0` | `NO PUBLICAR`
* **Volume**: `329 rows` (0.03%)
* **All Matching Cases**: PRODUCTO NUEVO, REGLA DDS, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, MAESTRA MARCAS PRODUCCION, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION
* **Commercial Meaning**: Negative buffer block. Shelf stock is positive (e.g. 4) but a large buffer (e.g. -5) results in negative display stock and a block.
* **Rule Exceptions**: JERARQUIA PRODUCCION and JERARQUIA CON STOCK REGULAR apply buffers that exceed physical stock, resulting in a safety block.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `568750`
* **VTEX SKU ID**: `5947`
* **Product Name**: `Agua Destilada Prodisa Desmineralizada Bidón 5 L`
* **Store**: `J408`
* **HANA Qty**: `8.0`
* **PUBLICADOR Qty**: `-1.0`
* **Example Specific Case (`caso`)**: `*REGLA DDS*`

---
### Permutation 17: `> 0` | `< 0` | `PUBLICAR`
* **Volume**: `207 rows` (0.02%)
* **All Matching Cases**: PROD CON VENTA, LISTADO FIJO SIN STOCK, PRODUCTO TIENDA MADRE
* **Commercial Meaning**: Negative buffer staple rescue. Large safety buffer makes display stock negative, but the item is force-published because it is a critical staple.
* **Rule Exceptions**: LISTADO FIJO SIN STOCK and PROD CON VENTA preserve the negative quantity in the file to keep the item buyable.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1898914`
* **VTEX SKU ID**: `108814`
* **Product Name**: `Vino Cousiño Macul Don Matías Gran Reserva Carmenere 750 cc`
* **Store**: `J519`
* **HANA Qty**: `4.0`
* **PUBLICADOR Qty**: `-1.0`
* **Example Specific Case (`caso`)**: `*PROD CON VENTA*`

---
### Permutation 18: `< 0` | `= 0` | `PUBLICAR`
* **Volume**: `75 rows` (0.01%)
* **All Matching Cases**: PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE
* **Commercial Meaning**: Negative stock rescue. HANA stock is negative, but rescued to exactly 0 display stock and published because of recent store sales.
* **Rule Exceptions**: PROD CON VENTA rescues the negative inventory and sets it to 0 or 1 unit.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `2069041`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J408`
* **HANA Qty**: `-1.0`
* **PUBLICADOR Qty**: `0.0`
* **Example Specific Case (`caso`)**: `*PRODUCTO TIENDA MADRE*`

---
### Permutation 19: `< 0` | `= 0` | `NO PUBLICAR`
* **Volume**: `69 rows` (0.01%)
* **All Matching Cases**: REGLA DDS, MAESTRA MARCAS MMPP-IMPO, SET DE VENTA, JERARQUIA PRODUCCION, PRODUCTO TIENDA MADRE
* **Commercial Meaning**: Negative stock clamped to zero block. HANA stock is negative, and display stock is clamped to 0 and blocked.
* **Rule Exceptions**: JERARQUIA PRODUCCION and REGLA DDS clamp negative physical inventory to exactly 0 and block it.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1696045`
* **VTEX SKU ID**: `57196`
* **Product Name**: `Agua Saborizada Uva 190 ml`
* **Store**: `J780`
* **HANA Qty**: `-7.0`
* **PUBLICADOR Qty**: `0.0`
* **Example Specific Case (`caso`)**: `*JERARQUIA PRODUCCION*`

---
### Permutation 20: `< 0` | `> 0` | `NO PUBLICAR`
* **Volume**: `66 rows` (0.01%)
* **All Matching Cases**: PRODUCTO TIENDA MADRE, REGLA DDS, PRODUCTO NO CAT EN DARKSTORE
* **Commercial Meaning**: Negative darkstore catalog block. Darkstore local stock is negative, inherits positive mother stock, but is blocked locally by the catalog.
* **Rule Exceptions**: PRODUCTO NO CAT EN DARKSTORE and REGLA DDS block the inherited stock because the darkstore catalog disables the SKU.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `2068197`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J408`
* **HANA Qty**: `-1.0`
* **PUBLICADOR Qty**: `1.0`
* **Example Specific Case (`caso`)**: `*PRODUCTO TIENDA MADRE*`

---
### Permutation 21: `NULL` | `< 0` | `NO PUBLICAR`
* **Volume**: `9 rows` (0.00%)
* **All Matching Cases**: PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE
* **Commercial Meaning**: Missing HANA record with negative buffer block. No stock record exists, a negative buffer is applied, and the item is blocked.
* **Rule Exceptions**: PRODUCTO NO CAT EN DARKSTORE and PRODUCTO TIENDA MADRE apply to these unmapped items with no inventory footprint.

#### Real-Data Example for this State:
* **SAP SKU (Item ID)**: `1972796`
* **VTEX SKU ID**: `N/A`
* **Product Name**: `Unknown Product`
* **Store**: `J414`
* **HANA Qty**: `NULL`
* **PUBLICADOR Qty**: `-13.0`
* **Example Specific Case (`caso`)**: `*PRODUCTO TIENDA MADRE*`

---
