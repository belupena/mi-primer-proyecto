# Entrega práctica 3 — Bases de datos NoSQL

- **Nombre:** María Belén Peña
- **student_id:** belu_pena
- **Escala:** small

## CAP y PACELC

**1.** Con la red partida, la réplica C en modo CP respondió "ERROR: no disponible" al leer y "ERROR: sin quórum" al escribir; en modo AP respondió "devuelve 100" al leer y "OK" al escribir. CP sacrifica la disponibilidad (prefiere dar error antes que un dato dudoso) y AP sacrifica la consistencia (siempre responde, pero C devolvió 100 cuando A y B ya tenían 80).

**2.** En modo AP, después de la reparación las tres réplicas quedaron en saldo = 50. Se perdió la escritura `saldo = 80` hecha en A, porque la reconciliación *last-write-wins* se queda con la versión más nueva (la de C).

**3.** Según la tabla y el gráfico, la p99 es 9,6 ms con W=1 y 55,6 ms con W=3. A cambio de esperar más réplicas se gana consistencia y durabilidad (la escritura queda en más nodos y las lecturas ven el dato más reciente), pero sube la latencia: es el trade-off latencia/consistencia de PACELC.

**4.** Pudo seguir escribiendo el lado ABC, porque tiene 3 de las 5 réplicas, que es la mayoría (quórum). El lado DE tiene solo 2: sus escrituras dieron "ERROR: sin quórum" y su lectura "ERROR: no disponible".

## Clave-valor

**5.** El GET en memoria fue 1.850.566 veces más rápido que el GET con Spark (0,0003 ms contra 522 ms). Redis guarda los datos en memoria porque buscar una clave en una tabla hash en RAM es mucho más rápido que ir a disco o lanzar una consulta distribuida.

**6.** Para contar los clientes de AR hubo que leer y parsear los 5.000 valores, porque el valor es un JSON opaco para la base y no existe un `WHERE country = 'AR'`. Ventaja: acceso por clave muy rápido y fácil de repartir entre nodos. Desventaja: no se puede consultar por el contenido del valor sin recorrer todo.

**7.** Al pasar de 4 a 5 nodos, `hash mod N` movió el 79,6 % de las claves y el hashing consistente el 21,2 % (≈ 1/5). Con mod N casi todas las claves cambian de nodo y hay que mover la mayoría de los datos por la red; con hashing consistente nodos y claves están sobre un anillo y solo se mueven las del tramo que ocupa el nodo nuevo. Los nodos virtuales equilibran la carga (con 1 nodo virtual la carga máxima llegó a 56 %, con 100 bajó a 21,2 %), por eso lo usan Dynamo y Cassandra.

## Documental

**8.** Para el cliente 42, la consulta relacional devolvió 11 filas (con 2 joins) y la documental 1 único documento. Quedaron embebidos el perfil del cliente, sus estadísticas y la lista de sus transacciones, cada una con su producto (snapshot con `product_id`, categoría y precio).

**9.** Spark construyó un esquema que es la unión de todos los campos y completó con `null` los que faltaban en cada documento. Se dice que el esquema es flexible porque los documentos de una colección no necesitan tener los mismos campos: escribimos cuatro documentos con formas distintas y el esquema se infirió al leer (schema-on-read).

**10.** El documento más grande pesa 5,2 KB (≈ 225 bytes por transacción). Si un documento crece sin límite, cada lectura y cada actualización mueve más datos y se termina chocando con el tope de tamaño del motor (16 MB en MongoDB, unas 74.599 transacciones a este ritmo); por eso se acotan los arrays o se referencian los datos en otra colección.

**11.** El modelo documental resuelve bien las consultas que leen una entidad completa en una sola lectura (un cliente con su historial) y las que filtran dentro del documento con campos anidados y funciones sobre arrays. Le cuestan las consultas transversales entre documentos, como el monto por categoría, que obligan a desarmar los arrays con `explode`, y también los joins entre colecciones.

## Grafos

**12.** El grafo tiene 5.000 nodos `Customer` (738 marcados), 2.361 nodos `Device` y 5.400 aristas `USES`.

**13.** SQL necesitó 3 joins para el patrón de 2 saltos y 5 joins para el de 4 saltos (cada flecha del patrón agrega un join contra la tabla de aristas). Neo4j recorre las relaciones con adyacencia sin índice: cada nodo guarda punteros directos a sus vecinos, así que el costo depende de los nodos que se visitan y no del tamaño de las tablas, mientras que cada join relacional busca coincidencias entre tablas completas.

**14.** La propagación de etiquetas convergió en 14 iteraciones (en la 14.ª ningún nodo cambió de componente) y el componente más grande tiene 23 nodos (`c:122`).

**15.** Con `hash(id) mod 4`, el 74,4 % de las aristas quedó entre máquinas distintas (con un reparto al azar entre 4 máquinas se esperan ≈ 75 %), frente a 0 % si se reparte por componente. Distribuir un grafo es difícil porque los nodos se conectan de forma arbitraria: cada arista que cruza máquinas obliga a un salto por la red, y encontrar un reparto que minimice esos cruces y mantenga la carga pareja es un problema complejo.

## Vectorial

**16.** El vector de cada cliente tiene 10 números que resumen su comportamiento: la proporción del gasto en cada una de las 5 categorías, la proporción de pagos por cada uno de los 3 canales, el ticket promedio (dividido por 2.000) y la proporción de compras de más de 1.500. La similitud coseno mide el ángulo entre dos vectores: vale 1 si apuntan en la misma dirección (comportamientos parecidos) y se acerca a 0 si no se parecen, sin depender de la magnitud.

**17.** Entre los vecinos de clientes marcados hay un 31,6 % de marcados, frente a una tasa base del 14,8 % (y 13,2 % entre los vecinos de clientes no marcados). Buscar vecinos parecidos sirve para detectar casos sospechosos, y también para recomendar, segmentar o encontrar duplicados; como clasificador solo no alcanza (acertó 85,9 %, contra 86,3 % de predecir siempre "no marcado"), pero aporta señal.

**18.** Con `nprobe=4` el índice IVF obtiene un recall@10 de 87,2 % escaneando solo el 5,99 % de los vectores. Con la búsqueda aproximada (ANN) se gana velocidad y menor costo, porque no se compara contra todos los vectores, y se pierde exactitud: puede faltar algún vecino verdadero. Subir `nprobe` mejora el recall (con 16 llega a 100 %) pero escanea más (24 %).

## Columnar

**19.** Las 503.490 transacciones ocupan 50,0 MB en CSV, 121,1 MB en JSON y 5,3 MB en Parquet (9,4 veces menos que CSV y 22,8 veces menos que JSON). Para la consulta de `amount` sobre Parquet, Spark leyó solo esa columna (`ReadSchema: struct<amount:decimal(12,2)>`), y con los datos ordenados por `amount` se pueden saltear 7 de 8 archivos (con los datos al azar, 0 de 8).

**20.** El formato columnar es bueno para analítica porque lee solo las columnas necesarias, comprime muy bien (valores del mismo tipo juntos) y puede saltear archivos con las estadísticas min/max. Es poco conveniente para modificar una fila por vez porque cada fila está repartida entre las columnas y los archivos son inmutables, así que cambiar una fila obliga a reescribir bloques enteros; para eso conviene un formato por filas.
