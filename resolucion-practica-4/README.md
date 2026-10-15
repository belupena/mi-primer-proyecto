# Entrega práctica 4 — Streaming: logs, ventanas y garantías

- **Nombre:** María Belén Peña
- **student_id:** belu_pena
- **Escala:** small

## Log y offsets (`01_log_y_offsets`)

**1.** Después de leer tres mensajes, en la cola quedaron 2 (`pago_4`, `pago_5`) y el log siguió teniendo los 5, con el offset del consumidor en 3. En una cola tradicional leer un mensaje lo borra; en un log como Kafka leer no modifica nada y cada consumidor solo avanza su propia posición (offset).

**2.** Después de la llegada 2, `antifraude` había leído 180 mensajes y `reportes` 0. El atraso de `reportes` no afectó a `antifraude`: cada consumidor tiene su propio checkpoint (como un consumer group distinto) y el topic conserva los mensajes, así que cada uno avanza a su ritmo.

**3.** Un offset es la posición hasta la que un consumidor leyó el log. En esta práctica lo guarda cada consulta en su checkpoint como la versión de la tabla Delta que le toca leer a continuación (al final de la prueba era 6 para ambos consumidores, con la última versión del topic en 5).

**4.** El replay leyó 280 mensajes, los mismos que tiene el topic. Kappa propone un único código de streaming: para reprocesar se lanza una consulta nueva (otro checkpoint) que lee el log desde el principio; Lambda, en cambio, mantiene dos sistemas (un batch preciso y una capa de streaming rápida) con la misma lógica escrita dos veces.

## Ingesta y enriquecimiento (`02_ingesta_y_enriquecimiento`)

**5.** Bronze procesó 60, 120, 40, 40 y 20 filas nuevas en las llegadas 1 a 5, y 0 al reejecutar sin archivos nuevos. Lo que evita leer dos veces el mismo archivo es el checkpoint de Auto Loader (`cloudFiles`), que recuerda qué archivos ya procesó.

**6.** Aparecieron `INVALID_AMOUNT` (20 registros) y `UNKNOWN_CUSTOMER` (20 registros), ambos en la llegada `stream_002`. Bronze guarda el texto original en lugar de descartar los registros con errores para no perder el dato crudo: si cambia el contrato o se corrige una regla, se puede reprocesar desde el original.

**7.** El join agregó `country` (desde `silver_customers`) y `category` (desde `silver_products`). Según la teoría es un join stream-tabla: cada evento del stream se enriquece con una tabla estática de referencia, y si la tabla cambia, los eventos siguientes ven la versión nueva sin recalcular los ya procesados.

## Tiempo y ventanas (`03_tiempo_y_ventanas`)

**8.** Event time es cuándo ocurrió la compra (`event_ts`) y processing time es cuándo llega al sistema. `late_ok` ocurrió a las 12:03 pero llegó en la llegada 2, después de que la llegada 1 ya había traído eventos de las 12:04.

**9.** El watermark al terminar cada llegada valió 11:54, 11:58, 12:16, 12:17 y 12:35. Se calcula como el máximo event time visto menos 10 minutos (por ejemplo, 12:26 − 10 min = 12:16).

**10.** Las primeras ventanas tumbling en modo append aparecieron en la llegada 3 (4 ventanas emitidas). Antes no, porque el watermark (11:54 y 11:58) todavía no había superado el fin de ninguna ventana (la primera termina a las 12:05); en la llegada 3 el watermark saltó a 12:16 y cerró las ventanas de 12:00–12:10.

**11.** `late_bad` (12:02) llegó en la llegada 4 con el watermark en 12:16: es más viejo que el watermark, así que se descartó y no sumó en ninguna ventana (20 eventos más viejos que el watermark). `late_ok` (12:03) se aceptó porque llegó en la llegada 2, cuando el watermark todavía estaba en 11:54, es decir, dentro del margen tolerado.

**12.** Tumbling son ventanas fijas de 5 minutos que no se solapan (cada evento cae en una sola): por ejemplo, ventas cada 5 minutos por canal de pago. Hopping son ventanas de 10 minutos que avanzan cada 5 y se solapan: por ejemplo, un promedio móvil del tráfico de la última media hora que se actualiza cada 5 minutos. Session agrupa por cliente y se cierra tras 5 minutos sin compras: por ejemplo, las compras de una misma visita a la tienda.

**13.** La ventana [12:00, 12:05) del canal card se emitió dos veces en modo update: 40 compras y 6.000,00 en la llegada 1, y 60 compras y 8.400,00 en la llegada 2, corregida cuando llegó `late_ok`. En modo append se emitió una sola vez, con 60 compras y 8.400,00, cuando el watermark la cerró (llegada 3). Update da resultados antes pero el consumidor debe tolerar la misma ventana dos veces; append es más simple y entrega el valor final, pero con más demora.

**14.** Con un watermark de 1 minuto las ventanas se cerrarían antes (menos latencia) y se guardaría menos estado en memoria. Se perdería completitud: eventos tardíos que hoy se aceptan, como `late_ok`, correrían riesgo de descartarse.

## Garantías y fallas (`04_garantias_y_fallas`)

**15.** Sin deduplicar quedaron 220 filas en el destino y deduplicando por `event_id` 200 (había 20 duplicados). El productor puede enviar el mismo evento dos veces porque, ante la duda (por ejemplo, si no recibe la confirmación), reintenta: es el comportamiento at-least-once del productor, como `e002`, que apareció en `stream_001` y `stream_002`.

**16.** Al reiniciar con el mismo checkpoint el destino siguió en 200 filas: el checkpoint sabe hasta dónde procesó y no escribe nada de nuevo (exactly-once dentro de Spark). Al perder el checkpoint la consulta empezó de cero y, como el destino era un `append` común, quedaron 400 filas: se observa at-least-once, con resultados duplicados.

**17.** Con `MERGE` por `event_id` (solo inserta si el evento no existe) el destino quedó en 200 filas aun con otro checkpoint, porque los eventos que ya estaban no se vuelven a insertar. Una escritura idempotente es aquella en la que aplicarla dos veces tiene el mismo efecto que aplicarla una (por ejemplo, `SET x = 5` en lugar de `INCREMENT x`); checkpoint más escritura idempotente da exactly-once de punta a punta.

## CDC (`05_cdc`)

**18.** El log generó 13 eventos de cambio: 6 `insert` (los 5 clientes iniciales y el alta del cliente 100), 3 `update_preimage`, 3 `update_postimage` y 1 `delete`. Un `UPDATE` genera dos eventos porque registra la fila antes del cambio (preimage) y después (postimage).

**19.** Sí: reconstruir la tabla quedándose con el último cambio por cliente y descartando los `delete` dio una tabla igual a la original. El log tiene lo que la tabla actual no tiene: la historia, por ejemplo que el cliente 2 pasó de UY a CL y luego a BR, y que existió el cliente 3 (BR, regular) antes de borrarse.

## Cierre

**20.** Spark Structured Streaming conviene cuando ya se trabaja con Spark o un lakehouse (como Delta en esta práctica) y alcanza una latencia de segundos: unifica batch y streaming con la misma API de DataFrames y SQL. Flink conviene cuando se necesita latencia muy baja (milisegundos), procesamiento evento por evento y un manejo avanzado del estado y del tiempo. Kafka Streams conviene para procesar datos que ya viven en Kafka con una simple librería dentro de la aplicación, sin armar un clúster aparte.
<!-- VERIFICAR la P20 contra la tabla de comparación de motores de la teoría de la clase -->
