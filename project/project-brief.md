# Project brief: priorización de reposiciones en una red de almacenes

## 1. Resumen

Herramienta de apoyo a decisiones para el equipo de planificación de inventario de una distribuidora con cinco almacenes. Cada semana ordenará las combinaciones producto-almacén por riesgo y explicará cada aviso. No hará pedidos automáticamente.

El análisis usa el *High-Dimensional Supply Chain Inventory Dataset* de Kaggle (91.250 filas, 15 columnas, 2024).

## 2. Dataset y calidad de los datos

### Ficha del dataset

| Campo | Información |
|---|---|
| Nombre | High-Dimensional Supply Chain Inventory Dataset |
| Publicador y enlace | Ziya07, Kaggle: https://www.kaggle.com/datasets/ziya07/high-dimensional-supply-chain-inventory-dataset |
| Licencia | CC0 1.0 Universal (dominio público): https://creativecommons.org/publicdomain/zero/1.0/ |
| Fecha de descarga y versión | Descargado el 13/10/2026. Kaggle no muestra fecha de última actualización e indica una frecuencia anual. |
| Qué representa una fila | Un registro por fecha, producto (`SKU_ID`) y almacén (`Warehouse_ID`). |
| Periodo | 01/01/2024 – 30/12/2024. |
| Tamaño | 91.250 filas × 15 columnas; 6,55 MB. 50 SKU × 5 almacenes = 250 combinaciones, presentes todos los días (250 × 365 = 91.250). 10 proveedores y 4 regiones. |
| Columnas principales | Fecha: `Date`. Dimensiones: `SKU_ID`, `Warehouse_ID`, `Supplier_ID`, `Region`. Medidas: `Units_Sold`, `Inventory_Level`, `Supplier_Lead_Time_Days`, `Reorder_Point`, `Order_Quantity`, `Unit_Cost`, `Unit_Price`, `Demand_Forecast`. Indicadores: `Promotion_Flag`, `Stockout_Flag`. Objetivo candidato: `Units_Sold` (regresión). |
| Calidad observada | 0 % nulos; 0 duplicados; la clave fecha-SKU-almacén es única. Proveedor, plazo, `Reorder_Point` y coste son constantes por combinación. `Stockout_Flag` vale 0 en el 100 % de las filas. `Region` cambia de un día a otro dentro de la misma combinación (las 250 aparecen en las 4 regiones), así que no describe el almacén. El pronóstico (`Demand_Forecast`) de cada día tiene una correlación de 0,95 con las ventas de ese mismo día y de 0,61 con las del día anterior o siguiente. Unidades y moneda de `Unit_Cost` y `Unit_Price` sin documentar. No hay valores imposibles; los outliers por IQR (1,3 % de los días) y Mahalanobis (48 filas) se marcan, no se borran. |
| Otras fuentes | Ninguna. Se valoró el calendario de festivos de 2024 (BOE), pero se descartó: la venta media es la misma todos los días de la semana (20,0–20,1 unidades). |

### Limitaciones y efecto en la decisión

- **Datos generados, no históricos.** Los 50 SKU comparten la misma curva estacional (correlación ≥ 0,99) y la demanda se reparte casi por igual. Las cifras no se pueden extrapolar a una empresa real.
- **Un solo año.** No muestra si la estacionalidad se repite. La validación será temporal dentro de 2024.
- **Faltan datos operativos.** Sin pedidos pendientes, capacidad ni costes, la herramienta prioriza pero no calcula cantidades óptimas. Usarla en la operación real exigiría datos actuales de la empresa.
- **Posible fuga en `Demand_Forecast`.** La correlación no la demuestra, pero no se usará en el bloque 3 hasta descartarla.
- **`Region` no describe el almacén.** Se comparará por `Warehouse_ID`.

## 3. Problema empresarial

**Organización.** Distribuciones Sago Packaging es un mayorista que abastece a comercios desde 5 almacenes, con 50 referencias y 10 proveedores.

**Departamento.** Planificación de inventario: tres personas que deciden qué reponer en cada almacén.

**Stakeholder.** La responsable de planificación. Revisa el dashboard cada lunes, antes de la reunión de operaciones. Puede adelantar o aplazar pedidos, pedir transferencias entre almacenes y revisar con Ventas una promoción. No puede cambiar precios, proveedores ni capacidad.

**Decisión.** La responsable decidirá cada semana qué combinaciones producto-almacén revisar primero y qué reposiciones adelantar o aplazar, según demanda, inventario, plazos y promociones.

**Restricciones.** Proyecto individual, sin ERP ni costes.

**Alternativas sin datos.** (1) Un Excel que avisa cuando `Inventory_Level` < `Reorder_Point`: barato, pero ignora plazo y estacionalidad. (2) Recuentos físicos: no anticipan nada. La regla de Excel se usará como **línea base**: la herramienta solo vale la pena si prioriza casos que esa regla no ve (H2).

**Otros afectados.** Compras, Almacén, Finanzas, Ventas y clientes.

## 4. Hipótesis

**Cobertura (días)** = `Inventory_Level` / media de `Units_Sold` de esa misma combinación SKU-almacén en los 28 días anteriores, sin incluir el día evaluado (mínimo 7 días de historia). Es una estimación. Las combinaciones sin cobertura calculable quedan aparte y no cuentan en los porcentajes. Los umbrales son criterios propios del proyecto, fijados tras una exploración preliminar. Las pruebas formales se harán en las lecciones 03 a 05. Prioridad por valor: H2 > H1 > H3.

| | **H2 (prioridad alta)** | **H1 (prioridad alta)** | **H3 (prioridad media)** |
|---|---|---|---|
| Enunciado | El criterio cobertura < plazo prioriza casos distintos de la regla de `Reorder_Point`, que ignora el plazo (r = 0,03). | El riesgo se concentra en los meses de demanda alta: inventario y `Reorder_Point` no siguen la estacionalidad. | Las promociones aumentan las ventas de forma relevante. |
| Métrica | % de registros con cobertura < plazo que la regla de `Reorder_Point` no marca. | % de registros en riesgo que caen entre febrero y mayo. | Uplift (%) = (media con promoción − media sin promoción) / media sin promoción. |
| Dirección esperada | Más del 30 %, concentrado en combinaciones con plazo ≥ 8 días. | Más del 60 %. | Más del 15 %, estable entre meses. |
| Cómo se comprueba | Tabla cruzada de ambas reglas por proveedor y plazo. | Riesgo y demanda por mes, gráfico de líneas. | Uplift del mismo SKU-almacén con y sin promoción, en los mismos meses (250 pares): mediana y % de pares positivos. |
| Si es cierta | Las reglas no son equivalentes y el motor combina ambas. Sin roturas reales (`Stockout_Flag` = 0) no se puede afirmar cuál acierta más. | Margen de seguridad estacional: más stock en primavera, menos en otoño. | Pedir a Ventas el calendario de promociones (no está en el dataset) y revisar antes la reposición. |
| Si es falsa | La regla de Excel basta: dashboard descriptivo. | Un margen único para todo el año. | No ajustar la reposición por promoción. |

En H3, la diferencia no prueba que la cause la promoción.

## 5. Indicadores del dashboard

| KPI | Fórmula | Grano | Comparación | Objetivo | Hipótesis |
|---|---|---|---|---|---|
| Demanda media diaria | Σ `Units_Sold` / días (ventana de 28 días) | SKU-almacén, día | Almacén, mes | Sin objetivo; KPI de contexto | H1 |
| % de registros en riesgo | Registros con cobertura < plazo / registros evaluados | Día | Almacén, proveedor | ≤ 2 % (actual: 3,7 %) | H2 |
| % de registros en banda | Registros con plazo ≤ cobertura ≤ plazo + 14 días / registros evaluados | Día | Almacén, mes | ≥ 70 % (actual: 41 %) | H1, H2 |
| % de registros bajo punto de pedido | Registros con `Inventory_Level` < `Reorder_Point` / registros evaluados (para cruzarlo con H2) | Día | Almacén | Sin objetivo; línea base (actual: 5,3 %) | H2 |
| Uplift por promoción | Ver H3 | SKU-almacén, mes | Mes | Sin objetivo; informa la previsión | H3 |
| Valor estimado del inventario (provisional: moneda sin verificar) | Σ `Inventory_Level` × `Unit_Cost` | Almacén, semana | Mes | Exploratorio: bajar en meses de demanda baja sin subir el % en riesgo | H1 |

Son porcentajes de registros diarios, que el dashboard agrega por semana. Los valores «actual» se calculan sobre 89.500 registros (el primer día evaluado de cada combinación es el octavo) y se reproducen con `project/01_eda_inventario.ipynb`. Los objetivos son metas del proyecto: reducir el riesgo a la mitad y cubrir bien a la mayoría.

**Trade-offs.** Subir la cobertura reduce el % en riesgo, pero aumenta el valor del inventario. La exploración preliminar apunta a que conviven ambos problemas: falta cobertura en primavera y sobra de agosto a diciembre, cuando más del 80 % de los registros de cada mes supera el plazo + 14 días.

## 6. Solución y arquitectura

### Tipo de solución

Herramienta de apoyo a decisiones: un motor calcula las prioridades y el dashboard las muestra.

- **Entradas:** el CSV de Kaggle (ventas, inventario, plazos, punto de pedido, promociones).
- **Lógica:** para cada SKU-almacén se calcula un índice de riesgo = plazo / cobertura. Por encima de 1, la combinación se marca para revisión. El índice avisa, pero no predice roturas. En el bloque 3, una previsión estacional (sin `Demand_Forecast`) sustituirá a la media móvil.
- **Salidas:** un ranking semanal con las 20 combinaciones a revisar, cada una con su motivo («cobertura 4 días, plazo 9»), y la lista de sobrecubiertas.
- **Capa what-if:** un control para cambiar los días de margen de seguridad; al moverlo se recalculan el ranking y un stock objetivo experimental, igual a demanda media × (plazo + margen).

### Arquitectura

```mermaid
flowchart LR
    A[Fuente<br/>CSV de Kaggle] --> B[Ingesta<br/>script Python + pandas]
    B --> C[Limpieza y calidad<br/>esquema, tipos, rangos, duplicados]
    C --> D[Modelo de datos<br/>hechos fecha×SKU×almacén<br/>+ dimensiones, Parquet]
    D --> E[Motor<br/>KPI, índice de riesgo, previsión]
    E --> F[Dashboard<br/>Streamlit + Plotly]
    F --> G((Responsable de<br/>planificación))
```

| Capa | Qué hace en el proyecto | Herramienta | Actualización |
|---|---|---|---|
| Fuente | CSV original sin modificar, con fecha de descarga | Kaggle | Anual, según el publicador |
| Ingesta | Carga y registra versión, filas y columnas | Python, pandas | Con cada descarga |
| Limpieza y calidad | Valida esquema, tipos, rangos y duplicados; si falla, se detiene | pandas, pytest | En cada carga |
| Modelo de datos | Tabla de hechos (fecha × SKU × almacén) y dimensiones producto, almacén, proveedor y fecha | Parquet | Tras superar las validaciones |
| Analítica / motor | KPI, índice de riesgo y, en el bloque 3, la previsión | Python, scikit-learn | Al cambiar los datos o las reglas |
| Dashboard | KPI, ranking semanal, detalle por SKU y promociones | Streamlit, Plotly | Al regenerar el motor |

**Responsable y refresco.** Lo mantengo yo manualmente con cada nueva versión. Si cambia el formato, las validaciones detienen el proceso. El dashboard muestra la fecha de la versión.

**Dónde vive.** `data/`, `src/`, `tests/` y `project/` en mi repositorio.

**Riesgos principales.** (1) Datos simulados demasiado regulares: el modelo parecerá mejor que con datos reales. (2) Cambio de formato o licencia: se guarda la copia original. (3) Un aviso erróneo puede encarecer el inventario o causar roturas: cada aviso es explicable y lo revisa una persona.

**Por qué esta solución.** Un dashboard descriptivo no ordena prioridades, y la regla de Excel ignora el plazo y la estacionalidad (el motor se medirá contra ella). Los pedidos automáticos se descartan por falta de datos de costes.

**Aspectos legales y éticos.** No hay datos personales y la licencia CC0 permite publicar resultados. El dashboard indicará que los datos son simulados.

## 7. Fuentes

1. Ziya07, *High-Dimensional Supply Chain Inventory Dataset*, Kaggle: https://www.kaggle.com/datasets/ziya07/high-dimensional-supply-chain-inventory-dataset
2. Creative Commons, *CC0 1.0 Universal*: https://creativecommons.org/publicdomain/zero/1.0/

Las cifras de tamaño, fechas, calidad y la exploración preliminar proceden del CSV descargado el 13/10/2026.
