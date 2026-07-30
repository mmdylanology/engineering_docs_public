# PUBLICADOR Inventory & Action Permutations Guide

This document maps all **21 active permutations** of physical inventory states and e-commerce publication decisions found in the July 10, 11:41 CLT snapshot.

---

## 1. Document Overview & Context
This document serves as an exhaustive reference of all physical and calculated stock states in the Cencosud JUMBO inventory pipeline. It is compiled by cross-referencing physical store inventory records with the generated e-commerce catalog update files.

* **Analysis Scope**: **1,225,228** unique SKU-store inventory combinations.
* **Timestamp analyzed**: **July 10, 2026, 11:41:24 CLT** (Chilean Local Time).
* **Pipeline Goal**: Translate physical warehouse stocks (SAP HANA parquets) to display web stock balances (VTEX SQS queues) while applying commercial overrides, buffers, and catalog restrictions.

### Resources & Artifact Paths
* **PUBLICADOR Output CSV**: [source.csv](file:///Users/malikmubarak/Desktop/JUMBO/data/publicador/source.csv)
* **Physical Inventory Batch (HANA)**: [vw_daily_nrt_0710_1141](file:///Users/malikmubarak/Desktop/JUMBO/data_samples/vw_daily_nrt_0710_1141)
* **VTEX to SAP Catalog Mapping**: [vtex_sap_mapping_FULL.xlsx](file:///Users/malikmubarak/Desktop/JUMBO/analysis/vtex_sap_mapping_FULL.xlsx)
* **Business Rules Core Specification**: [rules.pdf](file:///Users/malikmubarak/Desktop/JUMBO/rules.pdf)

---

## 2. The 21 Permutations Summary Table

| # | HANA State | PUB State | Action | Total Rows (%) | Exact Match <br> (HANA = PUB) | HANA > PUB <br> (Physical > Web) | PUB > HANA <br> (Web > Physical) | Cases Responsible |
| :---: | :--- | :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| 1 | `> 0` | `> 0` | **`PUBLICAR`** | `799,781 (65.28%)` | `714,750 (89.4%)` | `78,785 (9.9%)` | `6,246 (0.8%)` | SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, PRODUCTO NUEVO, CASO FECHA HORA, REGLA DDS, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, PROD CON VENTA, MAESTRA MARCAS PRODUCCION, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, LISTADO FIJO CON STOCK |
| 2 | `= 0` | `= 0` | **`NO PUBLICAR`** | `245,589 (20.04%)` | `245,589 (100.0%)` | `0 (0.0%)` | `0 (0.0%)` | PRODUCTO NUEVO, LISTADO FIJO CON STOCK, MAESTRA MARCAS PRODUCCION, REGLA DDS, JERARQUIA PRODUCCION, SET DE VENTA, PRODUCTO NO CAT EN DARKSTORE, MAESTRA MARCAS MMPP-IMPO, JERARQUIA DESHABILITADA, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, LISTADO BAJA |
| 3 | `> 0` | `> 0` | **`NO PUBLICAR`** | `121,570 (9.92%)` | `120,705 (99.3%)` | `756 (0.6%)` | `109 (0.1%)` | JERARQUIA DESHABILITADA, JERARQUIA PRODUCCION, REGLA DDS, LISTADO BAJA, LISTADO FIJO CON STOCK, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, MAESTRA MARCAS PRODUCCION, SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, PRODUCTO NO CAT EN DARKSTORE |
| 4 | `= 0` | `> 0` | **`PUBLICAR`** | `25,011 (2.04%)` | `0 (0.0%)` | `0 (0.0%)` | `25,011 (100.0%)` | PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE |
| 5 | `< 0` | `< 0` | **`NO PUBLICAR`** | `14,347 (1.17%)` | `14,228 (99.2%)` | `113 (0.8%)` | `6 (0.0%)` | PRODUCTO NUEVO, LISTADO FIJO CON STOCK, MAESTRA MARCAS PRODUCCION, REGLA DDS, JERARQUIA PRODUCCION, SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, JERARQUIA DESHABILITADA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, LISTADO BAJA |
| 6 | `= 0` | `= 0` | **`PUBLICAR`** | `3,844 (0.31%)` | `3,844 (100.0%)` | `0 (0.0%)` | `0 (0.0%)` | LISTADO FIJO SIN STOCK, PROD CON VENTA, CASO FECHA HORA, SET DE VENTA, PRODUCTO TIENDA MADRE, REGLA DDS |
| 7 | `= 0` | `> 0` | **`NO PUBLICAR`** | `3,511 (0.29%)` | `0 (0.0%)` | `0 (0.0%)` | `3,511 (100.0%)` | PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE, REGLA DDS |
| 8 | `< 0` | `< 0` | **`PUBLICAR`** | `2,813 (0.23%)` | `2,041 (72.6%)` | `745 (26.5%)` | `27 (1.0%)` | LISTADO FIJO SIN STOCK, CASO FECHA HORA, SET DE VENTA, PRODUCTO TIENDA MADRE, PROD CON VENTA |
| 9 | `> 0` | `= 0` | **`PUBLICAR`** | `2,305 (0.19%)` | `0 (0.0%)` | `2,305 (100.0%)` | `0 (0.0%)` | MAESTRA MARCAS MMPP-IMPO, SET DE VENTA, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, PROD CON VENTA, PRODUCTO NUEVO, REGLA DDS, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, MAESTRA MARCAS PRODUCCION |
| 10 | `< 0` | `> 0` | **`PUBLICAR`** | `2,150 (0.18%)` | `0 (0.0%)` | `0 (0.0%)` | `2,150 (100.0%)` | PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE |
| 11 | `NULL` | `= 0` | **`NO PUBLICAR`** | `1,437 (0.12%)` | `0 (0.0%)` | `0 (0.0%)` | `1,437 (100.0%)` | PRODUCTO NO CAT EN DARKSTORE, JERARQUIA DESHABILITADA, PRODUCTO TIENDA MADRE, LISTADO BAJA, SET DE VENTA, JERARQUIA PRODUCCION |
| 12 | `= 0` | `< 0` | **`NO PUBLICAR`** | `747 (0.06%)` | `0 (0.0%)` | `747 (100.0%)` | `0 (0.0%)` | PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, REGLA DDS, LISTADO FIJO CON STOCK, PRODUCTO NO CAT EN DARKSTORE, LISTADO BAJA, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION, MAESTRA MARCAS PRODUCCION, PRODUCTO NUEVO |
| 13 | `= 0` | `< 0` | **`PUBLICAR`** | `504 (0.04%)` | `0 (0.0%)` | `504 (100.0%)` | `0 (0.0%)` | LISTADO FIJO SIN STOCK, CASO FECHA HORA, REGLA DDS, PRODUCTO TIENDA MADRE, PROD CON VENTA |
| 14 | `> 0` | `= 0` | **`NO PUBLICAR`** | `492 (0.04%)` | `0 (0.0%)` | `492 (100.0%)` | `0 (0.0%)` | MAESTRA MARCAS MMPP-IMPO, PRODUCTO NO CAT EN DARKSTORE, MAESTRA MARCAS PRODUCCION, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, PRODUCTO NUEVO, REGLA DDS, JERARQUIA PRODUCCION, LISTADO BAJA, JERARQUIA DESHABILITADA |
| 15 | `NULL` | `> 0` | **`NO PUBLICAR`** | `372 (0.03%)` | `0 (0.0%)` | `0 (0.0%)` | `372 (100.0%)` | PRODUCTO NO CAT EN DARKSTORE |
| 16 | `> 0` | `< 0` | **`NO PUBLICAR`** | `329 (0.03%)` | `0 (0.0%)` | `329 (100.0%)` | `0 (0.0%)` | REGLA DDS, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, PRODUCTO NUEVO, MAESTRA MARCAS PRODUCCION, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION |
| 17 | `> 0` | `< 0` | **`PUBLICAR`** | `207 (0.02%)` | `0 (0.0%)` | `207 (100.0%)` | `0 (0.0%)` | PROD CON VENTA, LISTADO FIJO SIN STOCK, PRODUCTO TIENDA MADRE |
| 18 | `< 0` | `= 0` | **`PUBLICAR`** | `75 (0.01%)` | `0 (0.0%)` | `0 (0.0%)` | `75 (100.0%)` | REGLA DDS, PRODUCTO TIENDA MADRE, PROD CON VENTA |
| 19 | `< 0` | `= 0` | **`NO PUBLICAR`** | `69 (0.01%)` | `0 (0.0%)` | `0 (0.0%)` | `69 (100.0%)` | SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, PRODUCTO TIENDA MADRE, JERARQUIA PRODUCCION, REGLA DDS |
| 20 | `< 0` | `> 0` | **`NO PUBLICAR`** | `66 (0.01%)` | `0 (0.0%)` | `0 (0.0%)` | `66 (100.0%)` | PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE, REGLA DDS |
| 21 | `NULL` | `< 0` | **`NO PUBLICAR`** | `9 (0.00%)` | `0 (0.0%)` | `0 (0.0%)` | `9 (100.0%)` | PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE |

---

## 3. Detailed Permutations & Drill-Down Analysis

### Permutation 1: `> 0` | `> 0` | `PUBLICAR`
* **Volume**: `799,781 rows` (65.28%)
* **Cases Involved**: SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, PRODUCTO NUEVO, CASO FECHA HORA, REGLA DDS, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, PROD CON VENTA, MAESTRA MARCAS PRODUCCION, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, LISTADO FIJO CON STOCK

#### Drill-down A: Exact Match (`HANA = PUB`) — 714,750 rows (89.37%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Pasta para Uno Carozzi Caracoqueso 70 g** (SAP: `1822185` | VTEX: `81462`) | Store: `J408` | Qty: `33.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Pepinillos Kühne Schlemmertöpfchen Feine Gürkchen Clásico 300 g** (SAP: `925249` | VTEX: `3204`) | Store: `J408` | Qty: `15.0` | Case: *MAESTRA MARCAS MMPP-IMPO*
  * **Vino Casillero del Diablo Reserva Especial Cabernet Sauvignon 750 cc** (SAP: `1755861` | VTEX: `63483`) | Store: `J408` | Qty: `12.0` | Case: *JERARQUIA PRODUCCION*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 78,785 rows (9.85%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Ajo Desgranado Malla 150 g** (SAP: `296634` | VTEX: `5815`) | Store: `J408` | HANA: `30.0` | PUB: `28.0` | Case: *REGLA DDS*
  * **Jabón Líquido para Bebés Johnson's Avena 400 ml** (SAP: `1798142` | VTEX: `75452`) | Store: `J408` | HANA: `16.0` | PUB: `13.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Vino Undurraga Aliwen Reserva Carmenere 750 cc** (SAP: `1127027` | VTEX: `31322`) | Store: `J408` | HANA: `13.0` | PUB: `12.0` | Case: *JERARQUIA PRODUCCION*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 6,246 rows (0.78%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Zapallo Diaguita Orgánico 1 un.** (SAP: `1650361` | VTEX: `39780`) | Store: `J414` | HANA: `6.6` | PUB: `138.4` | Case: *REGLA DDS*
  * **Harina de Quinoa Mi Tierra 400 g** (SAP: `613500` | VTEX: `4265`) | Store: `J414` | HANA: `3.0` | PUB: `8.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Hervidor eléctrico Ursus Trotter UT-DUNKEL 17 P** (SAP: `1836687` | VTEX: `87389`) | Store: `J414` | HANA: `2.0` | PUB: `16.0` | Case: *REGLA DDS*

---

### Permutation 2: `= 0` | `= 0` | `NO PUBLICAR`
* **Volume**: `245,589 rows` (20.04%)
* **Cases Involved**: PRODUCTO NUEVO, LISTADO FIJO CON STOCK, MAESTRA MARCAS PRODUCCION, REGLA DDS, JERARQUIA PRODUCCION, SET DE VENTA, PRODUCTO NO CAT EN DARKSTORE, MAESTRA MARCAS MMPP-IMPO, JERARQUIA DESHABILITADA, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, LISTADO BAJA

#### Drill-down A: Exact Match (`HANA = PUB`) — 245,589 rows (100.00%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Helado Sammontana Croccantino 800 ml** (SAP: `2017489` | VTEX: `147512`) | Store: `J408` | Qty: `0.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Carbón Quincho Premium 2.5 kg** (SAP: `1721162` | VTEX: `51116`) | Store: `J408` | Qty: `0.0` | Case: *REGLA DDS*
  * **Tarjeta Micro SD C10 Kingston 128 GB Con Adaptador** (SAP: `1901687` | VTEX: `116083`) | Store: `J408` | Qty: `0.0` | Case: *REGLA DDS*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is already 0, so it cannot exceed positive or zero display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 3: `> 0` | `> 0` | `NO PUBLICAR`
* **Volume**: `121,570 rows` (9.92%)
* **Cases Involved**: JERARQUIA DESHABILITADA, JERARQUIA PRODUCCION, REGLA DDS, LISTADO BAJA, LISTADO FIJO CON STOCK, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, MAESTRA MARCAS PRODUCCION, SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, PRODUCTO NO CAT EN DARKSTORE

#### Drill-down A: Exact Match (`HANA = PUB`) — 120,705 rows (99.29%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Bebida Vegetal Vilay Arroz Sabor Original 1 L** (SAP: `1713115` | VTEX: `48011`) | Store: `J501` | Qty: `1.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Set Basurero Símil Rattan Pedal 5.5 L** (SAP: `1904260` | VTEX: `116513`) | Store: `J501` | Qty: `1.0` | Case: *JERARQUIA DESHABILITADA*
  * **Set 6 Vasos Altos Cristar Whisky Mikonos** (SAP: `1580780` | VTEX: `22555`) | Store: `J501` | Qty: `11.0` | Case: *JERARQUIA DESHABILITADA*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 756 rows (0.62%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Yogurt Batido Colun Mora 125 g** (SAP: `407611006` | VTEX: `7062`) | Store: `J504` | HANA: `14.0` | PUB: `2.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Cebolla Malla 1 kg** (SAP: `507056` | VTEX: `5838`) | Store: `J504` | HANA: `34.0` | PUB: `31.0` | Case: *REGLA DDS*
  * **Calzón Nocturno Nosotras Buenas Noches Talla m 3 un.** (SAP: `1892663` | VTEX: `105753`) | Store: `J504` | HANA: `5.0` | PUB: `3.0` | Case: *JERARQUIA CON STOCK REGULAR*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 109 rows (0.09%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Blíster Krons 3 cucharas de té Luna** (SAP: `1142644` | VTEX: `31353`) | Store: `J414` | HANA: `2.0` | PUB: `4.0` | Case: *REGLA DDS*
  * **Unknown Product** (SAP: `1895123` | VTEX: `N/A`) | Store: `J414` | HANA: `0.026` | PUB: `0.71` | Case: *PRODUCTO TIENDA MADRE*
  * **Zapallo Butternut Orgánico 1 un.** (SAP: `1441298` | VTEX: `18902`) | Store: `J414` | HANA: `6.0` | PUB: `13.0` | Case: *REGLA DDS*

---

### Permutation 4: `= 0` | `> 0` | `PUBLICAR`
* **Volume**: `25,011 rows` (2.04%)
* **Cases Involved**: PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Since physical inventory is zero, the display inventory must be inherited from a mother store (making PUB > 0), so it can never be equal.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is already 0, so it cannot exceed positive or zero display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 25,011 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Cepillo de Pelo It Cuadrado Grande Rubber Azul** (SAP: `2017978` | VTEX: `145718`) | Store: `J408` | HANA: `0.0` | PUB: `25.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Crema Corporal Umai Humectación 385 ml** (SAP: `1844763` | VTEX: `89635`) | Store: `J408` | HANA: `0.0` | PUB: `5.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `1984300` | VTEX: `N/A`) | Store: `J408` | HANA: `0.0` | PUB: `30.0` | Case: *REGLA DDS*

---

### Permutation 5: `< 0` | `< 0` | `NO PUBLICAR`
* **Volume**: `14,347 rows` (1.17%)
* **Cases Involved**: PRODUCTO NUEVO, LISTADO FIJO CON STOCK, MAESTRA MARCAS PRODUCCION, REGLA DDS, JERARQUIA PRODUCCION, SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, JERARQUIA DESHABILITADA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, LISTADO BAJA

#### Drill-down A: Exact Match (`HANA = PUB`) — 14,228 rows (99.17%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Jurel Al Natural San José Lata 300 g drenado** (SAP: `266192` | VTEX: `3295`) | Store: `J504` | Qty: `-24.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Máquina de Afeitar Gillette Prestobarba3 3 un.** (SAP: `1788979` | VTEX: `73732`) | Store: `J504` | Qty: `-23.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Palta Fuerte Malla 1 kg** (SAP: `260991` | VTEX: `103323`) | Store: `J504` | Qty: `-3.0` | Case: *REGLA DDS*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 113 rows (0.79%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Abrillantador SMF Piedra Pizarra 1 L** (SAP: `1057820` | VTEX: `22201`) | Store: `J624` | HANA: `-8.0` | PUB: `-10.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Huevos Cintazul Grandes Blancos 30 un.** (SAP: `302675` | VTEX: `6660`) | Store: `J624` | HANA: `-4.0` | PUB: `-5.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Repollo Blanco Dole 300 g** (SAP: `260698` | VTEX: `5683`) | Store: `J501` | HANA: `-2.0` | PUB: `-3.0` | Case: *REGLA DDS*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 6 rows (0.04%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Leche Colun Desayunow! Manzana Avena 330 ml** (SAP: `2007944` | VTEX: `143289`) | Store: `J408` | HANA: `-5.0` | PUB: `-1.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `2039273` | VTEX: `N/A`) | Store: `J408` | HANA: `-3.0` | PUB: `-1.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Ajo Granulado La Sazoneria 300 g** (SAP: `2019305` | VTEX: `144806`) | Store: `J408` | HANA: `-2.0` | PUB: `-1.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 6: `= 0` | `= 0` | `PUBLICAR`
* **Volume**: `3,844 rows` (0.31%)
* **Cases Involved**: LISTADO FIJO SIN STOCK, PROD CON VENTA, CASO FECHA HORA, SET DE VENTA, PRODUCTO TIENDA MADRE, REGLA DDS

#### Drill-down A: Exact Match (`HANA = PUB`) — 3,844 rows (100.00%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Prieta Tradicional 500 g** (SAP: `2007044` | VTEX: `141065`) | Store: `J513` | Qty: `0.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Ceviche salmón tipo peruano 290 g** (SAP: `1493620` | VTEX: `121954`) | Store: `J513` | Qty: `0.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Cheesecake Mousse Cookies And Cream** (SAP: `1583526` | VTEX: `8962`) | Store: `J513` | Qty: `0.0` | Case: *LISTADO FIJO SIN STOCK*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is already 0, so it cannot exceed positive or zero display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 7: `= 0` | `> 0` | `NO PUBLICAR`
* **Volume**: `3,511 rows` (0.29%)
* **Cases Involved**: PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE, REGLA DDS

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Since physical inventory is zero, the display inventory must be inherited from a mother store (making PUB > 0), so it can never be equal.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is already 0, so it cannot exceed positive or zero display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 3,511 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Unknown Product** (SAP: `1615787` | VTEX: `N/A`) | Store: `J403` | HANA: `0.0` | PUB: `9.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Porta Torta Rectangular** (SAP: `1953401` | VTEX: `129763`) | Store: `J403` | HANA: `0.0` | PUB: `13.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `1128829` | VTEX: `N/A`) | Store: `J403` | HANA: `0.0` | PUB: `2.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 8: `< 0` | `< 0` | `PUBLICAR`
* **Volume**: `2,813 rows` (0.23%)
* **Cases Involved**: LISTADO FIJO SIN STOCK, CASO FECHA HORA, SET DE VENTA, PRODUCTO TIENDA MADRE, PROD CON VENTA

#### Drill-down A: Exact Match (`HANA = PUB`) — 2,041 rows (72.56%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Pasta Fettuccine N°88 Carozzi 400 g** (SAP: `267628` | VTEX: `2306`) | Store: `J748` | Qty: `-25.0` | Case: *PROD CON VENTA*
  * **Pasta Roasted Chicken Salad 300 g** (SAP: `1932457` | VTEX: `115679`) | Store: `J748` | Qty: `-1.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Picado Naturnes Pollo Zanahoria Arroz 215 g** (SAP: `498103005` | VTEX: `8721`) | Store: `J748` | Qty: `-3.0` | Case: *PROD CON VENTA*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 745 rows (26.48%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Queque Frankfurt con Azúcar Flor 8-10 Porciones** (SAP: `684141` | VTEX: `9223`) | Store: `J502` | HANA: `-1.0` | PUB: `-2.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Pan Molde Fibra Clean Label 780 g** (SAP: `1806579` | VTEX: `77393`) | Store: `J502` | HANA: `-1.0` | PUB: `-3.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Piña Pelada 1 un.** (SAP: `1498557` | VTEX: `71734`) | Store: `J502` | HANA: `-1.0` | PUB: `-2.0` | Case: *LISTADO FIJO SIN STOCK*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 27 rows (0.96%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Queque Frankfurt con Azúcar Flor 8-10 Porciones** (SAP: `684141` | VTEX: `9223`) | Store: `J414` | HANA: `-9.0` | PUB: `-1.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `255202` | VTEX: `N/A`) | Store: `J414` | HANA: `-2.419` | PUB: `-0.996` | Case: *PRODUCTO TIENDA MADRE*
  * **Ravioles Carne 580 g** (SAP: `255175` | VTEX: `9161`) | Store: `J414` | HANA: `-11.0` | PUB: `-3.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 9: `> 0` | `= 0` | `PUBLICAR`
* **Volume**: `2,305 rows` (0.19%)
* **Cases Involved**: MAESTRA MARCAS MMPP-IMPO, SET DE VENTA, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, PROD CON VENTA, PRODUCTO NUEVO, REGLA DDS, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, MAESTRA MARCAS PRODUCCION

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 2,305 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Agua Saborizada Frambuesa 190 ml** (SAP: `1696044` | VTEX: `57195`) | Store: `J796` | HANA: `1217.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*
  * **Bebida Energética Red Bull Sin Azúcar 250 ml** (SAP: `1052135` | VTEX: `444`) | Store: `J796` | HANA: `492.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*
  * **Néctar Livean Naranja 200 ml** (SAP: `981778001` | VTEX: `801`) | Store: `J796` | HANA: `383.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 10: `< 0` | `> 0` | `PUBLICAR`
* **Volume**: `2,150 rows` (0.18%)
* **Cases Involved**: PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No buffer deductions are applied in this specific state.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 2,150 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Unknown Product** (SAP: `2060848` | VTEX: `N/A`) | Store: `J408` | HANA: `-1.0` | PUB: `30.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Bandeja Rectangular Ultrafoil Aluminio 1 L 4 un.** (SAP: `298465` | VTEX: `6274`) | Store: `J408` | HANA: `-4.0` | PUB: `10.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `2062275` | VTEX: `N/A`) | Store: `J408` | HANA: `-10.0` | PUB: `32.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 11: `NULL` | `= 0` | `NO PUBLICAR`
* **Volume**: `1,437 rows` (0.12%)
* **Cases Involved**: PRODUCTO NO CAT EN DARKSTORE, JERARQUIA DESHABILITADA, PRODUCTO TIENDA MADRE, LISTADO BAJA, SET DE VENTA, JERARQUIA PRODUCCION

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Since HANA inventory records do not exist, they cannot be equal to computed values.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is NULL, so it cannot exceed display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 1,437 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Lanzador de Dardos Nerf Elite 2.0 Volt Sd-1 + 6 Dardos** (SAP: `1856551` | VTEX: `95791`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Set Princesas Con 3 Zapatos Accesorios** (SAP: `1952468` | VTEX: `131995`) | Store: `J408` | HANA: `nan` | PUB: `45.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Galletas Villa Baviera Miel 250 g** (SAP: `622008` | VTEX: `9261`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*

---

### Permutation 12: `= 0` | `< 0` | `NO PUBLICAR`
* **Volume**: `747 rows` (0.06%)
* **Cases Involved**: PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, REGLA DDS, LISTADO FIJO CON STOCK, PRODUCTO NO CAT EN DARKSTORE, LISTADO BAJA, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION, MAESTRA MARCAS PRODUCCION, PRODUCTO NUEVO

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 747 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Cerveza Quilmes Lager 4.9° 300 cc** (SAP: `1962096` | VTEX: `126596`) | Store: `J660` | HANA: `0.0` | PUB: `-2.0` | Case: *REGLA DDS*
  * **Lustramuebles Virginia Naranja Aerosol 360 cc** (SAP: `933579002` | VTEX: `8084`) | Store: `J660` | HANA: `0.0` | PUB: `-1.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Mantequilla San Ignacio de Campo 230 g** (SAP: `1734642` | VTEX: `68916`) | Store: `J660` | HANA: `0.0` | PUB: `-1.0` | Case: *JERARQUIA CON STOCK REGULAR*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 13: `= 0` | `< 0` | `PUBLICAR`
* **Volume**: `504 rows` (0.04%)
* **Cases Involved**: LISTADO FIJO SIN STOCK, CASO FECHA HORA, REGLA DDS, PRODUCTO TIENDA MADRE, PROD CON VENTA

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 504 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Pan Molde Proteína Clean Label 780 g** (SAP: `1806580` | VTEX: `77394`) | Store: `J624` | HANA: `0.0` | PUB: `-2.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Piña Pelada 1 un.** (SAP: `1498557` | VTEX: `71734`) | Store: `J624` | HANA: `0.0` | PUB: `-3.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Pan Kassler Cereal 750 g** (SAP: `295937` | VTEX: `9092`) | Store: `J624` | HANA: `0.0` | PUB: `-1.0` | Case: *LISTADO FIJO SIN STOCK*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 14: `> 0` | `= 0` | `NO PUBLICAR`
* **Volume**: `492 rows` (0.04%)
* **Cases Involved**: MAESTRA MARCAS MMPP-IMPO, PRODUCTO NO CAT EN DARKSTORE, MAESTRA MARCAS PRODUCCION, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR, PRODUCTO NUEVO, REGLA DDS, JERARQUIA PRODUCCION, LISTADO BAJA, JERARQUIA DESHABILITADA

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 492 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Pañales Huggies Natural Care Talla Rn 34 un.** (SAP: `1858794` | VTEX: `98218`) | Store: `J633` | HANA: `2.0` | PUB: `0.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Cepillo de Dientes Dento Niños Suave 2 un.** (SAP: `1413925` | VTEX: `9501`) | Store: `J633` | HANA: `2.0` | PUB: `0.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Agua Tónica Canada Dry Zero 1.5 L** (SAP: `1706558` | VTEX: `32665`) | Store: `J633` | HANA: `3.0` | PUB: `0.0` | Case: *REGLA DDS*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 15: `NULL` | `> 0` | `NO PUBLICAR`
* **Volume**: `372 rows` (0.03%)
* **Cases Involved**: PRODUCTO NO CAT EN DARKSTORE

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Since HANA inventory records do not exist, they cannot be equal to computed values.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is NULL, so it cannot exceed display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 372 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Pack Whisky Johnnie Walker Blonde 750 cc + 2 Vasos** (SAP: `1939713` | VTEX: `119458`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Barbie Fashion Nuevo Closet de Lujo** (SAP: `1928172` | VTEX: `126687`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Playset Bluey Juegos de Rol Mini** (SAP: `1995635` | VTEX: `143983`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*

---

### Permutation 16: `> 0` | `< 0` | `NO PUBLICAR`
* **Volume**: `329 rows` (0.03%)
* **Cases Involved**: REGLA DDS, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, PRODUCTO NUEVO, MAESTRA MARCAS PRODUCCION, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 329 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Lechuga Fresh Cut Hojas Roble Verde 150 g** (SAP: `1261602` | VTEX: `5810`) | Store: `J624` | HANA: `2.0` | PUB: `-2.0` | Case: *REGLA DDS*
  * **Pechuga de Pollo Deshuesada Sin Piel Al Natural Super Pollo 780 g** (SAP: `1561726` | VTEX: `24287`) | Store: `J507` | HANA: `1.0` | PUB: `-1.0` | Case: *JERARQUIA PRODUCCION*
  * **Manzana Fuji Orgánica 900 g** (SAP: `1839769` | VTEX: `93525`) | Store: `J507` | HANA: `2.0` | PUB: `-1.0` | Case: *REGLA DDS*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 17: `> 0` | `< 0` | `PUBLICAR`
* **Volume**: `207 rows` (0.02%)
* **Cases Involved**: PROD CON VENTA, LISTADO FIJO SIN STOCK, PRODUCTO TIENDA MADRE

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 207 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Huevos Santa Marta Blanco 60 un.** (SAP: `2017578` | VTEX: `144075`) | Store: `J748` | HANA: `2.0` | PUB: `-1.0` | Case: *PROD CON VENTA*
  * **Pan Molde Low Carb Clean Label 700 g** (SAP: `1856603` | VTEX: `100173`) | Store: `J414` | HANA: `2.0` | PUB: `-8.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Maní Tipo Japonés Marco Polo Bolsa 100 g** (SAP: `1125544` | VTEX: `2574`) | Store: `J414` | HANA: `24.0` | PUB: `-1.0` | Case: *PROD CON VENTA*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 18: `< 0` | `= 0` | `PUBLICAR`
* **Volume**: `75 rows` (0.01%)
* **Cases Involved**: REGLA DDS, PRODUCTO TIENDA MADRE, PROD CON VENTA

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No buffer deductions are applied in this specific state.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 75 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Bebida Coca-Cola Zero 220 ml** (SAP: `1739916` | VTEX: `63142`) | Store: `J762` | HANA: `-70.0` | PUB: `0.0` | Case: *PROD CON VENTA*
  * **Néctar Manzana 190 ml** (SAP: `1214554` | VTEX: `848`) | Store: `J503` | HANA: `-97.0` | PUB: `0.0` | Case: *PROD CON VENTA*
  * **Unknown Product** (SAP: `2069041` | VTEX: `N/A`) | Store: `J408` | HANA: `-1.0` | PUB: `0.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 19: `< 0` | `= 0` | `NO PUBLICAR`
* **Volume**: `69 rows` (0.01%)
* **Cases Involved**: SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, PRODUCTO TIENDA MADRE, JERARQUIA PRODUCCION, REGLA DDS

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No buffer deductions are applied in this specific state.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 69 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Agua Saborizada Uva 190 ml** (SAP: `1696045` | VTEX: `57196`) | Store: `J502` | HANA: `-36.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*
  * **Agua Saborizada Uva 190 ml** (SAP: `1696045` | VTEX: `57196`) | Store: `J503` | HANA: `-27.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*
  * **Corazón de Costina Duo Color** (SAP: `1998295` | VTEX: `138683`) | Store: `J414` | HANA: `-1.0` | PUB: `0.0` | Case: *REGLA DDS*

---

### Permutation 20: `< 0` | `> 0` | `NO PUBLICAR`
* **Volume**: `66 rows` (0.01%)
* **Cases Involved**: PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE, REGLA DDS

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No buffer deductions are applied in this specific state.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 66 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Unknown Product** (SAP: `2068197` | VTEX: `N/A`) | Store: `J408` | HANA: `-1.0` | PUB: `1.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `1556091` | VTEX: `N/A`) | Store: `J408` | HANA: `-2.846` | PUB: `0.506` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `2063878` | VTEX: `N/A`) | Store: `J408` | HANA: `-5.0` | PUB: `2.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 21: `NULL` | `< 0` | `NO PUBLICAR`
* **Volume**: `9 rows` (0.00%)
* **Cases Involved**: PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Since HANA inventory records do not exist, they cannot be equal to computed values.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is NULL, so it cannot exceed display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 9 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Juguete Didáctico Sonajero Mi Primer Animalito (surtido)** (SAP: `1744132` | VTEX: `56456`) | Store: `J408` | HANA: `nan` | PUB: `35.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Pack Whisky Johnnie Walker Blonde 750 cc + 2 Vasos** (SAP: `1939713` | VTEX: `119458`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Lanzador de Dardos Nerf Elite 2.0 Volt Sd-1 + 6 Dardos** (SAP: `1856551` | VTEX: `95791`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*

---

