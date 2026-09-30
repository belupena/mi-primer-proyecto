# Práctica 1 — Ingesta y capa Bronze

**Nombre:** María Belén Peña

**student_id:** belu_pena

## Tres observaciones sobre CSV/JSON, Parquet y Delta

1. **CSV/JSON:** no traen tipos definidos. El CSV se leyó todo como `string`. Con `inferSchema` hizo falta una pasada extra por los datos y `amount` siguió como texto por valores como `N/A`. El JSON trae estructura anidada (`context` con `platform` y `session_id`).
2. **Parquet:** es un formato columnar que guarda el esquema dentro del archivo (`product_id` long, `price` decimal(12,2)), sin necesidad de inferirlo. Al ser columnar, una consulta lee solo las columnas que necesita: en el plan de `EXPLAIN`, el `ReadSchema` incluye únicamente `payment_channel`.
3. **Delta:** son archivos Parquet más un log de transacciones. Con `DESCRIBE HISTORY` se ve la versión 0, la operación `CREATE OR REPLACE TABLE AS SELECT` y las 50.011 filas escritas. Con `DESCRIBE DETAIL` se ven metadatos como `numFiles` y `sizeInBytes`. Un directorio Parquet suelto no ofrece ese historial ni ese control de las escrituras.

## Las cinco V en este caso

- **Volumen:** hay 5.000 clientes, 500 productos, 50.011 transacciones y 200.000 eventos. La tabla de eventos es la más grande.
- **Velocidad:** transacciones y eventos tienen `event_ts`, por lo que en un caso real llegarían de forma continua. En la práctica se cargaron en lote y las cuatro tablas se escribieron en unos 23 segundos.
- **Variedad:** las fuentes vienen en CSV (clientes, transacciones), Parquet (productos) y JSON anidado (eventos), cada una con esquemas y tipos distintos.
- **Veracidad:** el diagnóstico de calidad encontró 50.011 filas para 50.000 `transaction_id` distintos (11 duplicados) y 52 importes que no se pueden convertir a número. Bronze conserva estos problemas y se corregirían en Silver.
- **Valor:** con los datos ya ingestados y trazables (`_source`, `_source_file`, `_ingested_at`) se pueden analizar ventas por canal de pago, comportamiento de los usuarios a partir de los eventos y detección de fraude con `is_fraud`.