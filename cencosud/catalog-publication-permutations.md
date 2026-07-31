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
| 1 | `> 0` | `> 0` | **`PUBLICAR`** | `799,781 (65.28%)` | `714,750 (89.4%)` | `78,785 (9.9%)` | `6,246 (0.8%)` | JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, REGLA DDS, PROD CON VENTA, PRODUCTO NUEVO, CASO FECHA HORA, LISTADO FIJO CON STOCK, MAESTRA MARCAS PRODUCCION, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, SET DE VENTA |
| 2 | `= 0` | `= 0` | **`NO PUBLICAR`** | `245,589 (20.04%)` | `245,589 (100.0%)` | `0 (0.0%)` | `0 (0.0%)` | REGLA DDS, SET DE VENTA, LISTADO BAJA, MAESTRA MARCAS MMPP-IMPO, MAESTRA MARCAS PRODUCCION, JERARQUIA DESHABILITADA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA PRODUCCION, PRODUCTO NUEVO, LISTADO FIJO CON STOCK |
| 3 | `> 0` | `> 0` | **`NO PUBLICAR`** | `121,570 (9.92%)` | `120,705 (99.3%)` | `756 (0.6%)` | `109 (0.1%)` | MAESTRA MARCAS PRODUCCION, LISTADO FIJO CON STOCK, JERARQUIA PRODUCCION, JERARQUIA DESHABILITADA, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, LISTADO BAJA, REGLA DDS |
| 4 | `= 0` | `> 0` | **`PUBLICAR`** | `25,011 (2.04%)` | `0 (0.0%)` | `0 (0.0%)` | `25,011 (100.0%)` | PROD CON VENTA, PRODUCTO TIENDA MADRE, REGLA DDS |
| 5 | `< 0` | `< 0` | **`NO PUBLICAR`** | `14,347 (1.17%)` | `14,228 (99.2%)` | `113 (0.8%)` | `6 (0.0%)` | REGLA DDS, SET DE VENTA, LISTADO BAJA, MAESTRA MARCAS MMPP-IMPO, MAESTRA MARCAS PRODUCCION, JERARQUIA DESHABILITADA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, JERARQUIA PRODUCCION, PRODUCTO NUEVO, LISTADO FIJO CON STOCK |
| 6 | `= 0` | `= 0` | **`PUBLICAR`** | `3,844 (0.31%)` | `3,844 (100.0%)` | `0 (0.0%)` | `0 (0.0%)` | PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE, CASO FECHA HORA, SET DE VENTA, LISTADO FIJO SIN STOCK |
| 7 | `= 0` | `> 0` | **`NO PUBLICAR`** | `3,511 (0.29%)` | `0 (0.0%)` | `0 (0.0%)` | `3,511 (100.0%)` | REGLA DDS, PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE |
| 8 | `< 0` | `< 0` | **`PUBLICAR`** | `2,813 (0.23%)` | `2,041 (72.6%)` | `745 (26.5%)` | `27 (1.0%)` | SET DE VENTA, CASO FECHA HORA, PRODUCTO TIENDA MADRE, PROD CON VENTA, LISTADO FIJO SIN STOCK |
| 9 | `> 0` | `= 0` | **`PUBLICAR`** | `2,305 (0.19%)` | `0 (0.0%)` | `2,305 (100.0%)` | `0 (0.0%)` | JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, PROD CON VENTA, REGLA DDS, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, MAESTRA MARCAS PRODUCCION, SET DE VENTA, PRODUCTO NUEVO, MAESTRA MARCAS MMPP-IMPO |
| 10 | `< 0` | `> 0` | **`PUBLICAR`** | `2,150 (0.18%)` | `0 (0.0%)` | `0 (0.0%)` | `2,150 (100.0%)` | REGLA DDS, PRODUCTO TIENDA MADRE, PROD CON VENTA |
| 11 | `NULL` | `= 0` | **`NO PUBLICAR`** | `1,437 (0.12%)` | `0 (0.0%)` | `0 (0.0%)` | `1,437 (100.0%)` | SET DE VENTA, LISTADO BAJA, JERARQUIA PRODUCCION, PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA DESHABILITADA |
| 12 | `= 0` | `< 0` | **`NO PUBLICAR`** | `747 (0.06%)` | `0 (0.0%)` | `747 (100.0%)` | `0 (0.0%)` | LISTADO FIJO CON STOCK, LISTADO BAJA, MAESTRA MARCAS MMPP-IMPO, MAESTRA MARCAS PRODUCCION, PRODUCTO NUEVO, JERARQUIA PRODUCCION, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, REGLA DDS |
| 13 | `= 0` | `< 0` | **`PUBLICAR`** | `504 (0.04%)` | `0 (0.0%)` | `504 (100.0%)` | `0 (0.0%)` | CASO FECHA HORA, PRODUCTO TIENDA MADRE, PROD CON VENTA, REGLA DDS, LISTADO FIJO SIN STOCK |
| 14 | `> 0` | `= 0` | **`NO PUBLICAR`** | `492 (0.04%)` | `0 (0.0%)` | `492 (100.0%)` | `0 (0.0%)` | JERARQUIA PRODUCCION, MAESTRA MARCAS MMPP-IMPO, PRODUCTO NUEVO, MAESTRA MARCAS PRODUCCION, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA DESHABILITADA, REGLA DDS, LISTADO BAJA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE |
| 15 | `NULL` | `> 0` | **`NO PUBLICAR`** | `372 (0.03%)` | `0 (0.0%)` | `0 (0.0%)` | `372 (100.0%)` | PRODUCTO NO CAT EN DARKSTORE |
| 16 | `> 0` | `< 0` | **`NO PUBLICAR`** | `329 (0.03%)` | `0 (0.0%)` | `329 (100.0%)` | `0 (0.0%)` | PRODUCTO NUEVO, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION, MAESTRA MARCAS PRODUCCION, REGLA DDS, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR |
| 17 | `> 0` | `< 0` | **`PUBLICAR`** | `207 (0.02%)` | `0 (0.0%)` | `207 (100.0%)` | `0 (0.0%)` | LISTADO FIJO SIN STOCK, PRODUCTO TIENDA MADRE, PROD CON VENTA |
| 18 | `< 0` | `= 0` | **`PUBLICAR`** | `75 (0.01%)` | `0 (0.0%)` | `0 (0.0%)` | `75 (100.0%)` | PRODUCTO TIENDA MADRE, PROD CON VENTA, REGLA DDS |
| 19 | `< 0` | `= 0` | **`NO PUBLICAR`** | `69 (0.01%)` | `0 (0.0%)` | `0 (0.0%)` | `69 (100.0%)` | REGLA DDS, PRODUCTO TIENDA MADRE, JERARQUIA PRODUCCION, MAESTRA MARCAS MMPP-IMPO, SET DE VENTA |
| 20 | `< 0` | `> 0` | **`NO PUBLICAR`** | `66 (0.01%)` | `0 (0.0%)` | `0 (0.0%)` | `66 (100.0%)` | REGLA DDS, PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE |
| 21 | `NULL` | `< 0` | **`NO PUBLICAR`** | `9 (0.00%)` | `0 (0.0%)` | `0 (0.0%)` | `9 (100.0%)` | PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE |

---

## 3. Detailed Permutations & Drill-Down Analysis

### Permutation 1: `> 0` | `> 0` | `PUBLICAR`
* **Volume**: `799,781 rows` (65.28%)
* **Cases Involved**: JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, REGLA DDS, PROD CON VENTA, PRODUCTO NUEVO, CASO FECHA HORA, LISTADO FIJO CON STOCK, MAESTRA MARCAS PRODUCCION, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, SET DE VENTA

#### Drill-down A: Exact Match (`HANA = PUB`) — 714,750 rows (89.37%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Kuchen de Nuez Fontarella 10 un** (SAP: `2018661` | VTEX: `144660`) | Store: `J501` | Qty: `41.0` | Case: *JERARQUIA PRODUCCION*
  * **Granola en Línea Cranberries 320 g** (SAP: `1749552` | VTEX: `55835`) | Store: `J501` | Qty: `41.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Detergente Lavavajilla Somat All In One Gel 1.1 L** (SAP: `1925103` | VTEX: `112215`) | Store: `J501` | Qty: `98.0` | Case: *MAESTRA MARCAS MMPP-IMPO*

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
* **Cases Involved**: REGLA DDS, SET DE VENTA, LISTADO BAJA, MAESTRA MARCAS MMPP-IMPO, MAESTRA MARCAS PRODUCCION, JERARQUIA DESHABILITADA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA PRODUCCION, PRODUCTO NUEVO, LISTADO FIJO CON STOCK

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
* **Cases Involved**: MAESTRA MARCAS PRODUCCION, LISTADO FIJO CON STOCK, JERARQUIA PRODUCCION, JERARQUIA DESHABILITADA, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, SET DE VENTA, MAESTRA MARCAS MMPP-IMPO, LISTADO BAJA, REGLA DDS

#### Drill-down A: Exact Match (`HANA = PUB`) — 120,705 rows (99.29%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Basurero Plástico Tapa Vaivén Plateado 10 L** (SAP: `1905817` | VTEX: `116668`) | Store: `J502` | Qty: `2.0` | Case: *REGLA DDS*
  * **Faldón 1.5 Plazas Microfibra 120 gsm** (SAP: `1906161` | VTEX: `117256`) | Store: `J502` | Qty: `9.0` | Case: *JERARQUIA DESHABILITADA*
  * **Lentes de Buceo Bestway Clásico Colores 7-13 Años** (SAP: `1998057` | VTEX: `143057`) | Store: `J502` | Qty: `2.0` | Case: *REGLA DDS*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 756 rows (0.62%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Huevos del Granero Gallinas Libres Extra Color 20 un.** (SAP: `1960680` | VTEX: `126582`) | Store: `J660` | HANA: `2.0` | PUB: `1.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Alimento Húmedo Perro Cachorro Pedigree Carne Sobre 85 g** (SAP: `1026773` | VTEX: `8388`) | Store: `J660` | HANA: `17.0` | PUB: `4.0` | Case: *REGLA DDS*
  * **Bebida Limón Soda Zero 1.75 L** (SAP: `1716831` | VTEX: `62523`) | Store: `J660` | HANA: `9.0` | PUB: `5.0` | Case: *REGLA DDS*

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
* **Cases Involved**: PROD CON VENTA, PRODUCTO TIENDA MADRE, REGLA DDS

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
* **Cases Involved**: REGLA DDS, SET DE VENTA, LISTADO BAJA, MAESTRA MARCAS MMPP-IMPO, MAESTRA MARCAS PRODUCCION, JERARQUIA DESHABILITADA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, JERARQUIA PRODUCCION, PRODUCTO NUEVO, LISTADO FIJO CON STOCK

#### Drill-down A: Exact Match (`HANA = PUB`) — 14,228 rows (99.17%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Acondicionador Ballerina Fuerza y Cuidado 750 ml** (SAP: `2029574` | VTEX: `147918`) | Store: `J502` | Qty: `-5.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Ketchup Hela Sabor Curry Picante 300 ml** (SAP: `1749879` | VTEX: `62555`) | Store: `J502` | Qty: `-19.0` | Case: *MAESTRA MARCAS MMPP-IMPO*
  * **Endulzante líquido alulosa 180 ml** (SAP: `1840706` | VTEX: `88055`) | Store: `J502` | Qty: `-5.0` | Case: *JERARQUIA CON STOCK REGULAR*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 113 rows (0.79%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Mayonesa Mccormick Guacamole** (SAP: `2001393` | VTEX: `147964`) | Store: `J408` | HANA: `-1.0` | PUB: `-3.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Galletas Doble Chocolate 2 un.** (SAP: `2006248` | VTEX: `141058`) | Store: `J408` | HANA: `-1.0` | PUB: `-12.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Pepinillos Agridulces 360 g** (SAP: `1929254` | VTEX: `118919`) | Store: `J408` | HANA: `-1.0` | PUB: `-3.0` | Case: *PRODUCTO TIENDA MADRE*

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
* **Cases Involved**: PROD CON VENTA, REGLA DDS, PRODUCTO TIENDA MADRE, CASO FECHA HORA, SET DE VENTA, LISTADO FIJO SIN STOCK

#### Drill-down A: Exact Match (`HANA = PUB`) — 3,844 rows (100.00%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Fugazza Al Pesto con Aceitunas un.** (SAP: `1509589` | VTEX: `9073`) | Store: `J660` | Qty: `0.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Ceviche de Reineta Nogada 290 g** (SAP: `1496584` | VTEX: `122027`) | Store: `J660` | Qty: `0.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Palta Reina un.** (SAP: `1359541` | VTEX: `121987`) | Store: `J660` | Qty: `0.0` | Case: *LISTADO FIJO SIN STOCK*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is already 0, so it cannot exceed positive or zero display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 7: `= 0` | `> 0` | `NO PUBLICAR`
* **Volume**: `3,511 rows` (0.29%)
* **Cases Involved**: REGLA DDS, PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE

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
* **Cases Involved**: SET DE VENTA, CASO FECHA HORA, PRODUCTO TIENDA MADRE, PROD CON VENTA, LISTADO FIJO SIN STOCK

#### Drill-down A: Exact Match (`HANA = PUB`) — 2,041 rows (72.56%)
* **Why it matches**: Standard active stock propagation. No safety buffers are applied, or the applied buffer was 0.
* **Examples**:
  * **Ravioles Carne 580 g** (SAP: `255175` | VTEX: `9161`) | Store: `J502` | Qty: `-17.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Pasta de Aceitunas Negras Oliomio 200 g** (SAP: `1473645` | VTEX: `18703`) | Store: `J502` | Qty: `-4.0` | Case: *PROD CON VENTA*
  * **Hielo Fiesta 2 kg** (SAP: `302665` | VTEX: `34`) | Store: `J502` | Qty: `-134.0` | Case: *PROD CON VENTA*

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 745 rows (26.48%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Queque Frankfurt con Chocolate 8-10 Porciones** (SAP: `684140` | VTEX: `9224`) | Store: `J501` | HANA: `-1.0` | PUB: `-3.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Mix Mango Arándano Frutilla 400 g** (SAP: `1863035` | VTEX: `99569`) | Store: `J501` | HANA: `-1.0` | PUB: `-3.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Sandwich Ave Palta natural un.** (SAP: `1800316` | VTEX: `111526`) | Store: `J501` | HANA: `-41.0` | PUB: `-43.0` | Case: *LISTADO FIJO SIN STOCK*

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
* **Cases Involved**: JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, PROD CON VENTA, REGLA DDS, JERARQUIA PRODUCCION, LISTADO FIJO SIN STOCK, MAESTRA MARCAS PRODUCCION, SET DE VENTA, PRODUCTO NUEVO, MAESTRA MARCAS MMPP-IMPO

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 2,305 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Jugo en Polvo Livean Maracuyá 7 g** (SAP: `1237163` | VTEX: `633`) | Store: `J748` | HANA: `310.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*
  * **Néctar Livean frutifrutilla 200 cc** (SAP: `1560678` | VTEX: `814`) | Store: `J748` | HANA: `336.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*
  * **Cerveza Clausthaler Radler Limón Sin Alcohol 500 cc** (SAP: `1910545` | VTEX: `111695`) | Store: `J748` | HANA: `62.0` | PUB: `0.0` | Case: *REGLA DDS*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 10: `< 0` | `> 0` | `PUBLICAR`
* **Volume**: `2,150 rows` (0.18%)
* **Cases Involved**: REGLA DDS, PRODUCTO TIENDA MADRE, PROD CON VENTA

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
* **Cases Involved**: SET DE VENTA, LISTADO BAJA, JERARQUIA PRODUCCION, PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA DESHABILITADA

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Since HANA inventory records do not exist, they cannot be equal to computed values.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is NULL, so it cannot exceed display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 1,437 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Barbie Fashion Nuevo Closet de Lujo** (SAP: `1928172` | VTEX: `126687`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Playset Bluey Juegos de Rol Mini** (SAP: `1995635` | VTEX: `143983`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Pack Whisky Johnnie Walker Blonde 750 cc + 2 Vasos** (SAP: `1939713` | VTEX: `119458`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*

---

### Permutation 12: `= 0` | `< 0` | `NO PUBLICAR`
* **Volume**: `747 rows` (0.06%)
* **Cases Involved**: LISTADO FIJO CON STOCK, LISTADO BAJA, MAESTRA MARCAS MMPP-IMPO, MAESTRA MARCAS PRODUCCION, PRODUCTO NUEVO, JERARQUIA PRODUCCION, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE, REGLA DDS

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 747 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Alimento Húmedo Perro Purina One Multi Proteína Carne, Pollo y Cordero 85 g** (SAP: `1938730` | VTEX: `119612`) | Store: `J760` | HANA: `0.0` | PUB: `-6.0` | Case: *REGLA DDS*
  * **Ice Tea Lipton Black Raspberry Zero 600 ml** (SAP: `1961331` | VTEX: `127804`) | Store: `J512` | HANA: `0.0` | PUB: `-6.0` | Case: *JERARQUIA PRODUCCION*
  * **Pana de Pollo Ariztía 600 g** (SAP: `1938620` | VTEX: `118894`) | Store: `J512` | HANA: `0.0` | PUB: `-1.0` | Case: *JERARQUIA PRODUCCION*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 13: `= 0` | `< 0` | `PUBLICAR`
* **Volume**: `504 rows` (0.04%)
* **Cases Involved**: CASO FECHA HORA, PRODUCTO TIENDA MADRE, PROD CON VENTA, REGLA DDS, LISTADO FIJO SIN STOCK

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 504 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Pañuelos Facial Elite Bolsa 50 un.** (SAP: `2021742` | VTEX: `146416`) | Store: `J748` | HANA: `0.0` | PUB: `-3.0` | Case: *PROD CON VENTA*
  * **Piña Bastón Unidad** (SAP: `1872152` | VTEX: `101364`) | Store: `J748` | HANA: `0.0` | PUB: `-1.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Piña Pelada 1 un.** (SAP: `1498557` | VTEX: `71734`) | Store: `J748` | HANA: `0.0` | PUB: `-1.0` | Case: *LISTADO FIJO SIN STOCK*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 14: `> 0` | `= 0` | `NO PUBLICAR`
* **Volume**: `492 rows` (0.04%)
* **Cases Involved**: JERARQUIA PRODUCCION, MAESTRA MARCAS MMPP-IMPO, PRODUCTO NUEVO, MAESTRA MARCAS PRODUCCION, PRODUCTO NO CAT EN DARKSTORE, JERARQUIA DESHABILITADA, REGLA DDS, LISTADO BAJA, JERARQUIA CON STOCK REGULAR, PRODUCTO TIENDA MADRE

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 492 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Bebida Energética Red Bull Zero 250 ml** (SAP: `2029630` | VTEX: `147901`) | Store: `J796` | HANA: `17.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*
  * **Néctar Naranja 190 ml** (SAP: `1214553` | VTEX: `849`) | Store: `J796` | HANA: `6.0` | PUB: `0.0` | Case: *JERARQUIA PRODUCCION*
  * **Pañales Huggies Natural Care Talla Rn 34 un.** (SAP: `1858794` | VTEX: `98218`) | Store: `J633` | HANA: `2.0` | PUB: `0.0` | Case: *JERARQUIA CON STOCK REGULAR*

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
  * **Lanzador de Dardos Nerf Elite 2.0 Volt Sd-1 + 6 Dardos** (SAP: `1856551` | VTEX: `95791`) | Store: `J408` | HANA: `nan` | PUB: `0.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Set Princesas Con 3 Zapatos Accesorios** (SAP: `1952468` | VTEX: `131995`) | Store: `J408` | HANA: `nan` | PUB: `45.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Acelga 1 un.** (SAP: `1555313` | VTEX: `99500`) | Store: `J408` | HANA: `nan` | PUB: `-2.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*

---

### Permutation 16: `> 0` | `< 0` | `NO PUBLICAR`
* **Volume**: `329 rows` (0.03%)
* **Cases Involved**: PRODUCTO NUEVO, MAESTRA MARCAS MMPP-IMPO, JERARQUIA PRODUCCION, MAESTRA MARCAS PRODUCCION, REGLA DDS, PRODUCTO TIENDA MADRE, JERARQUIA CON STOCK REGULAR

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 329 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Estufa Convección Sindelen EEP-2200BL Blanca 2000W** (SAP: `1789085` | VTEX: `74620`) | Store: `J532` | HANA: `1.0` | PUB: `-1.0` | Case: *REGLA DDS*
  * **Huevos Cintazul Grandes Blancos 30 un.** (SAP: `302675` | VTEX: `6660`) | Store: `J506` | HANA: `1.0` | PUB: `-1.0` | Case: *JERARQUIA CON STOCK REGULAR*
  * **Toallas Húmedas Simond's Gloss 2 Paquetes de 60 un.** (SAP: `1930854` | VTEX: `114020`) | Store: `J506` | HANA: `7.0` | PUB: `-3.0` | Case: *JERARQUIA CON STOCK REGULAR*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 17: `> 0` | `< 0` | `PUBLICAR`
* **Volume**: `207 rows` (0.02%)
* **Cases Involved**: LISTADO FIJO SIN STOCK, PRODUCTO TIENDA MADRE, PROD CON VENTA

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 207 rows (100.00%)
* **Why it happens**: Safety reserve deduction. The system subtracted a safety buffer (usually between `-1` and `-5` units) to prevent overselling of low shelf stock.
* **Examples**:
  * **Piña 1 un.** (SAP: `1502634` | VTEX: `15312`) | Store: `J624` | HANA: `2.0` | PUB: `-1.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Lechuga Escarola Feria 1 un.** (SAP: `1555305` | VTEX: `99588`) | Store: `J624` | HANA: `2.0` | PUB: `-1.0` | Case: *LISTADO FIJO SIN STOCK*
  * **Tortellini Jamón Serrano y Queso 250 g** (SAP: `1611087` | VTEX: `9178`) | Store: `J624` | HANA: `1.0` | PUB: `-1.0` | Case: *PROD CON VENTA*

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No stock overrides or darkstore inheritances apply in this specific state.

---

### Permutation 18: `< 0` | `= 0` | `PUBLICAR`
* **Volume**: `75 rows` (0.01%)
* **Cases Involved**: PRODUCTO TIENDA MADRE, PROD CON VENTA, REGLA DDS

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
  * **Torta Bizcocho, Manjar, Salsa Frambuesa y Frutos Secos 15 Porciones** (SAP: `1774606` | VTEX: `69951`) | Store: `J414` | HANA: `-1.0` | PUB: `0.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 19: `< 0` | `= 0` | `NO PUBLICAR`
* **Volume**: `69 rows` (0.01%)
* **Cases Involved**: REGLA DDS, PRODUCTO TIENDA MADRE, JERARQUIA PRODUCCION, MAESTRA MARCAS MMPP-IMPO, SET DE VENTA

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No buffer deductions are applied in this specific state.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 69 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Contenedor Pote 1.6 L** (SAP: `1931768` | VTEX: `123185`) | Store: `J408` | HANA: `-2.0` | PUB: `0.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `2070614` | VTEX: `N/A`) | Store: `J408` | HANA: `-1.0` | PUB: `0.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Sidra Outcider Eva 330 cc** (SAP: `1843493` | VTEX: `93778`) | Store: `J408` | HANA: `-3.0` | PUB: `0.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 20: `< 0` | `> 0` | `NO PUBLICAR`
* **Volume**: `66 rows` (0.01%)
* **Cases Involved**: REGLA DDS, PRODUCTO NO CAT EN DARKSTORE, PRODUCTO TIENDA MADRE

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Calculated display stocks are always altered by buffers in this state.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. No buffer deductions are applied in this specific state.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 66 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Queso Cabra Ciboulette Chavroux Envasado Pote 150 g** (SAP: `1627571` | VTEX: `24428`) | Store: `J414` | HANA: `-2.0` | PUB: `3.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Salchicha Premium Schwencke 500** (SAP: `1617362` | VTEX: `134764`) | Store: `J414` | HANA: `-2.0` | PUB: `4.0` | Case: *PRODUCTO TIENDA MADRE*
  * **Unknown Product** (SAP: `2021134` | VTEX: `N/A`) | Store: `J414` | HANA: `-3.0` | PUB: `66.0` | Case: *PRODUCTO TIENDA MADRE*

---

### Permutation 21: `NULL` | `< 0` | `NO PUBLICAR`
* **Volume**: `9 rows` (0.00%)
* **Cases Involved**: PRODUCTO TIENDA MADRE, PRODUCTO NO CAT EN DARKSTORE

#### Drill-down A: Exact Match (`HANA = PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Since HANA inventory records do not exist, they cannot be equal to computed values.

#### Drill-down B: Physical Stock Exceeds Display Stock (`HANA > PUB`) — 0 rows (0.00%)
* *Explanation*: This subset is empty. Physical stock is NULL, so it cannot exceed display stock.

#### Drill-down C: Display Stock Exceeds Physical Stock (`PUB > HANA`) — 9 rows (100.00%)
* **Why it happens**: Two primary reasons:
  1. **Darkstore Inheritance**: Local stock is 0 (or negative), but inherits positive stock from its assigned mother store.
  2. **Staple Rescue**: Inventory is negative or zero, but force-published because the item is a grocery staple or recently sold.
* **Examples**:
  * **Alga Luche Bolsa 30 g** (SAP: `1773410` | VTEX: `98268`) | Store: `J408` | HANA: `nan` | PUB: `10.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Espinaca Feria 1 un.** (SAP: `1555310` | VTEX: `99618`) | Store: `J408` | HANA: `nan` | PUB: `105.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*
  * **Auto Radio Controlado 1:20 Rally (surtido)** (SAP: `1839658` | VTEX: `88784`) | Store: `J408` | HANA: `nan` | PUB: `35.0` | Case: *PRODUCTO NO CAT EN DARKSTORE*

---

## 4. Financial Valuation of Blocked Stock
This section calculates the financial value of products that physically have stock on shelf but are marked as `NO PUBLICAR`. We use the **Weighted Average Cost (CPP Cost)** from SAP HANA to value this stock in Chilean Pesos (CLP) and US Dollars (USD) at an exchange rate of `1 USD = 920 CLP`.

### Summary of Blocked Inventory Value
* **Total Blocked SKU-Store Pairs**: **122,213 pairs** (Deduplicated)
* **Total Blocked Physical Units**: **2,239,455 units**
* **Total Tied-Up Inventory Value**: **6,858,883,171 CLP** (Approx. **$7,455,308 USD**)

---

### Financial Loss Breakdown by Permutation
The following table shows how the blocked inventory cost is split across the three unpublished permutations:

| Permutation | Description | Unique Rows | Blocked Value (CLP) | Blocked Value (USD) | % of Block |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Permutation 3** | `HANA > 0` \| `PUB > 0` \| `NO PUBLICAR` | 121,570 | 6,848,550,837 CLP | $7,444,077 USD | 99.85% |
| **Permutation 14** | `HANA > 0` \| `PUB = 0` \| `NO PUBLICAR` | 492 | 9,023,880 CLP | $9,809 USD | 0.13% |
| **Permutation 16** | `HANA > 0` \| `PUB < 0` \| `NO PUBLICAR` | 329 | 5,068,129 CLP | $5,509 USD | 0.07% |
| **TOTAL** | | **122,391** | **6,862,642,846 CLP** | **$7,459,395 USD** | **100.00%** |

*Note: The raw sum is 6.86 Billion CLP. Deduplicating minor overlapping UOM records reduces the true asset cost slightly to 6.858 Billion CLP.*

---

### Financial Loss Breakdown by Motive Rule (Case)
The following table groups the blocked inventory cost by the specific business rule case:

| Motive Rule Case (`caso`) | Blocked Rows | Blocked Value (CLP) | Blocked Value (USD) | % of Block | Database Description & Business Context |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **`JERARQUIA DESHABILITADA`** | 105,003 | 5,805,498,615 CLP | $6,310,325 USD | 84.64% | The product belongs to a category/hierarchy node that is currently disabled in the e-commerce publication mappings. *(Note: This can represent either a configuration gap or an intentional decision to exclude certain categories from online sale).* |
| **`LISTADO BAJA`** | 5,497 | 897,161,934 CLP | $975,176 USD | 13.08% | Product is commercially delisted or discontinued. |
| **`REGLA DDS`** | 3,320 | 46,546,893 CLP | $50,594 USD | 0.68% | Blocked because days of supply is below safety levels. |
| **`JERARQUIA CON STOCK REGULAR`** | 5,143 | 44,365,533 CLP | $48,223 USD | 0.65% | Blocked because physical stock is below the minimum safety threshold. |
| *Other Minor Rules combined* | 1,250 | 65,310,201 CLP | $70,989 USD | 0.95% | Private label blocks, sets, and new product rules. |
| **TOTAL** | **122,213** | **6,858,883,171 CLP** | **$7,455,308 USD** | **100.00%** | |

---

### Top 10 Blocked Store-Items by Cost Value
Below are the top 10 individual store-SKU inventory blocks:

1. **Unknown Product** (SAP: `1923723` | Store: `J514` | Qty: `839.00` | Unit Cost: `17,446 CLP`) ➔ **14,637,194 CLP** (Case: *LISTADO BAJA*)
2. **Unknown Product** (SAP: `2052038` | Store: `J512` | Qty: `383.00` | Unit Cost: `24,661 CLP`) ➔ **9,445,163 CLP** (Case: *LISTADO BAJA*)
3. **Unknown Product** (SAP: `2065873` | Store: `J519` | Qty: `1286.12` | Unit Cost: `6,514 CLP`) ➔ **8,377,794 CLP** (Case: *JERARQUIA DESHABILITADA*)
4. **Unknown Product** (SAP: `1844024` | Store: `J514` | Qty: `355.00` | Unit Cost: `22,801 CLP`) ➔ **8,094,355 CLP** (Case: *LISTADO BAJA*)
5. **Unknown Product** (SAP: `2065873` | Store: `J514` | Qty: `1237.48` | Unit Cost: `6,469 CLP`) ➔ **8,005,055 CLP** (Case: *JERARQUIA DESHABILITADA*)
6. **Whisky Johnnie Walker Blue Label 750 cc** (SAP: `264264` | Store: `J514` | Qty: `32.00` | Unit Cost: `216,309 CLP`) ➔ **6,921,885 CLP** (Case: *LISTADO BAJA*)
7. **Unknown Product** (SAP: `1997500` | Store: `J614` | Qty: `212.72` | Unit Cost: `32,354 CLP`) ➔ **6,882,461 CLP** (Case: *LISTADO BAJA*)
8. **Unknown Product** (SAP: `2065873` | Store: `J512` | Qty: `924.16` | Unit Cost: `6,557 CLP`) ➔ **6,059,645 CLP** (Case: *JERARQUIA DESHABILITADA*)
9. **Unknown Product** (SAP: `1844028` | Store: `J534` | Qty: `258.00` | Unit Cost: `23,222 CLP`) ➔ **5,991,276 CLP** (Case: *LISTADO BAJA*)
10. **Unknown Product** (SAP: `1957167` | Store: `J514` | Qty: `193.00` | Unit Cost: `26,851 CLP`) ➔ **5,182,243 CLP** (Case: *LISTADO BAJA*)

*Note on Unknown Products: These items are unmapped in the e-commerce catalog integration database, meaning they do not have active VTEX SKU mappings or transaction orders, which is why their names cannot be resolved from the integration catalog. However, they represent real physical store inventory assets.*
