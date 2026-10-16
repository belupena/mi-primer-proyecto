# Entrega práctica 5 — MLOps con MLflow

- **Nombre:** María Belén Peña
- **student_id:** belu_pena
- **Escala:** small

## Tabla de runs (notebook 03, sección 5)

Mis 15 mejores runs ordenados por PR-AUC de test (en total registré 40 runs de modelos en el experimento `bigdata_belu_pena_clase05`):

| run | modelo | features | class_weight | umbral | hiperparametros | train_pr_auc | pr_auc | accuracy | precision | recall | f1 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| features=completas | gradient_boosting | completas | balanced | 0.4 | learning_rate=0.1, max_depth=5 | 0.614 | 0.579 | 0.753 | 0.278 | 0.861 | 0.42 |
| run8 | gradient_boosting | completas | balanced | 0.4 | learning_rate=0.1, max_depth=5 | 0.614 | 0.579 | 0.753 | 0.278 | 0.861 | 0.42 |
| max_depth=8 | random_forest | completas | ninguno | 0.5 | n_estimators=100, max_depth=8 | 0.643 | 0.568 | 0.914 | 0.799 | 0.226 | 0.352 |
| run1 | random_forest | completas | ninguno | 0.5 | n_estimators=100, max_depth=8 | 0.643 | 0.568 | 0.914 | 0.799 | 0.226 | 0.352 |
| max_depth=8 | random_forest | completas | ninguno | 0.5 | n_estimators=200, max_depth=8 | 0.644 | 0.564 | 0.914 | 0.804 | 0.222 | 0.347 |
| max_depth=sin_limite | random_forest | completas | ninguno | 0.5 | n_estimators=200, max_depth=sin_limite | 1.0 | 0.544 | 0.914 | 0.673 | 0.328 | 0.441 |
| max_depth=5 | random_forest | completas | ninguno | 0.5 | n_estimators=100, max_depth=5 | 0.561 | 0.541 | 0.903 | 0.906 | 0.078 | 0.144 |
| features=completas | logistic_regression | completas | balanced | 0.4 | C=10.0 | 0.527 | 0.54 | 0.711 | 0.246 | 0.862 | 0.383 |
| run7 | logistic_regression | completas | balanced | 0.4 | C=10.0 | 0.527 | 0.54 | 0.711 | 0.246 | 0.862 | 0.383 |
| run7 | logistic_regression | completas | balanced | 0.4 | C=10.0 | 0.527 | 0.54 | 0.711 | 0.246 | 0.862 | 0.383 |
| C=10 | logistic_regression | completas | balanced | 0.4 | C=10.0 | 0.527 | 0.54 | 0.711 | 0.246 | 0.862 | 0.383 |
| run6 | logistic_regression | completas | balanced | 0.4 | C=10.0 | 0.527 | 0.54 | 0.711 | 0.246 | 0.862 | 0.383 |
| C=1 | logistic_regression | completas | balanced | 0.4 | C=1.0 | 0.527 | 0.54 | 0.711 | 0.246 | 0.862 | 0.383 |
| max_depth=5 | random_forest | completas | ninguno | 0.5 | n_estimators=200, max_depth=5 | 0.559 | 0.54 | 0.903 | 0.927 | 0.069 | 0.128 |
| run2 | random_forest | completas | ninguno | 0.5 | n_estimators=200, max_depth=5 | 0.559 | 0.54 | 0.903 | 0.927 | 0.069 | 0.128 |

## Datos y tracking

**1.** `train` tiene 35.654 filas (11,05 % de contracargos), `test` 7.129 (10,38 %) y `nuevas` 7.165 (11,05 %). Se separa por fecha porque en un sistema real el modelo se entrena con el pasado y se usa sobre lo que viene después: una partición al azar mezclaría días y filtraría información del futuro al entrenamiento, con lo que las métricas quedarían demasiado optimistas.

**2.** Un modelo que siempre dice "no" obtiene una accuracy de 89,6 % sin detectar ni un solo contracargo. Con clases desbalanceadas la accuracy engaña, por eso se usa el PR-AUC: mide qué tan bien el modelo ordena las transacciones de más a menos riesgosas, se centra en la clase minoritaria y no depende de un umbral (un modelo al azar da ≈ 0,104, la tasa de contracargos).

**3.** El run `01_manual_logistica` registró como parámetros `model_type`, `feature_set`, `C`, `class_weight`, `threshold`, `data_table` y `data_version`; como métricas `test_pr_auc` 0,339, `test_roc_auc` 0,794, `test_accuracy` 0,896, `test_precision` 0,493, `test_recall` 0,045, `test_f1` 0,082, `test_alertas` 67 y `train_pr_auc` 0,341; y como artefacto la matriz de confusión (`matriz_confusion.png`), más el modelo con su firma. Un experimento es la carpeta que agrupa todos los intentos de un mismo problema, y un run es una ejecución individual de entrenamiento dentro de ese experimento, con sus propios parámetros, métricas y artefactos.

**4.** Autolog registró 46 parámetros (todos los del Pipeline, no sólo los que elegimos) y 7 métricas, todas de entrenamiento: `training_accuracy_score` 0,91, `training_f1_score` 0,887, `training_log_loss` 0,229, `training_precision_score` 0,901, `training_recall_score` 0,91, `training_roc_auc` 0,899 y `training_score` 0,91. No registra las métricas sobre el conjunto de test, ni el PR-AUC, ni el umbral, ni la versión de los datos, porque no conoce nuestro problema.

**5.** Sí, `01_repro_a` y `01_repro_b` dieron métricas idénticas (el notebook imprimió `True`). Para que un run sea reproducible hay que fijar los mismos datos, el mismo código, la misma semilla (`random_state=42`) y el mismo entorno.

**6.** Al llamar al modelo sin `amount`, MLflow frenó la predicción con el error `Failed to enforce schema of data ...`. La firma (*signature*) es el contrato del modelo: declara qué columnas y tipos espera y qué devuelve, así que valida los datos de entrada antes de predecir y evita resultados incorrectos o silenciosos.

## Tus experimentos

**7.** Mi mejor run fue `run8`: un `gradient_boosting` con variables `completas`, `learning_rate=0,1`, `max_depth=5`, `class_weight=balanced` y umbral 0,4, con PR-AUC de test de 0,579. Usando sólo `rule_alert` (`solo_regla`, con un bosque) el PR-AUC fue 0,170, así que el modelo mejoró 0,409 puntos (≈ 3,4 veces).

**8.** Con el mismo modelo (`gradient_boosting`, `balanced`, umbral 0,4) el PR-AUC pasó de 0,328 con `basicas` a 0,514 con `cliente` (+0,186) y a 0,579 con `completas` (+0,065); la logística muestra lo mismo (0,338 → 0,444 → 0,540). Las variables más importantes en mi mejor run fueron `amount` (0,336) y `payment_channel` (0,180), seguidas de `category` (0,082), `hour` (0,081) y `account_age_days` (0,049).

**9.** Con `completas` anduvo mejor el boosting (PR-AUC 0,579), seguido por el bosque (0,568) y último la logística (0,540). Un modelo de árboles puede aprovechar combinaciones porque cada división se hace sobre el resultado de la anterior: puede aprender una regla como "de noche *y* por transferencia *y* importe alto" y asignarle riesgo, mientras que una logística suma el efecto de cada variable por separado (es lineal) y sólo capta interacciones si se las agregamos a mano.

**10.** Comparando `max_depth=5`, 100 árboles, sin balanceo (precision 0,906, recall 0,078, unas 64 alertas) con `run5` (`balanced`, mismo bosque), la precision bajó a 0,222, el recall subió a 0,904 y las alertas pasaron a 3.008, porque los contracargos pesan más al entrenar y el modelo alerta mucho más. El PR-AUC casi no cambió (0,541 → 0,539). Ojo: `run5` también usó umbral 0,4, así que parte del cambio en alertas, precision y recall viene del umbral y no sólo del balanceo.

**11.** Al bajar el umbral de 0,5 a 0,4 (en `run5`) el modelo genera muchas más alertas: sube el recall (detecta más contracargos) y baja la precision (más falsas alarmas). El PR-AUC no depende del umbral porque se calcula con los scores recorriendo todos los umbrales posibles, mientras que el F1 se calcula con las predicciones ya binarizadas en un umbral concreto, así que cambia cuando ese corte se mueve.

**12.** Barrí `max_depth` en un `random_forest` con `completas` y 100 árboles: PR-AUC de test 0,525 (3), 0,541 (5), **0,568 (8)** y 0,537 (sin_limite), con lo que el mejor valor fue `max_depth=8`. Sí: con `sin_limite` el `train_pr_auc` fue 1,0 contra 0,537 en test (con 8 fue 0,643 contra 0,568). Ese fenómeno se llama sobreajuste (*overfitting*): el modelo memoriza el entrenamiento en lugar de aprender patrones que se repitan.

## Comparar y elegir

**13.** No coincidieron: el mejor por PR-AUC fue `run8` (boosting, PR-AUC 0,579, accuracy 0,753) y el mejor por accuracy fue el bosque con `max_depth=sin_limite` (accuracy 0,916, PR-AUC 0,537). Pueden diferir porque la accuracy depende del umbral y está dominada por la clase mayoritaria (con 10 % de contracargos, no alertar casi nunca "acierta"): el boosting con umbral 0,4 y `balanced` alerta mucho y detecta el 86 % de los contracargos, pero su accuracy baja; el PR-AUC evalúa cómo se ordena el riesgo sin importar el umbral.

**14.** Quedan más lejos de la diagonal los bosques con `max_depth=sin_limite` (train 1,0 contra test ≈ 0,52–0,54) y, sobre todo, el de variables `basicas` sin límite de profundidad (train 0,997 contra test 0,199). Esa distancia indica sobreajuste: rinden muy bien con los datos que ya vieron y mucho peor con datos nuevos.

**15.** `mlflow.search_runs` permite filtrar, ordenar y procesar los runs desde código como si fueran una tabla, de forma reproducible y automatizable, algo que no se puede hacer mirándolos uno por uno en la interfaz cuando hay decenas de runs. El `filter_string` para traer sólo mis bosques con PR-AUC de test mayor a 0,55 sería:

```python
filter_string = "tags.student_id = 'belu_pena' AND params.model_type = 'random_forest' AND metrics.test_pr_auc > 0.55"
```

**16.** En fraude suele costar más un contracargo no detectado, porque la plataforma pierde el dinero, mientras que una alerta falsa cuesta tiempo de revisión y fricción con el cliente. Si el equipo de riesgo sólo puede revisar una cantidad fija de alertas por día, ajustaría el umbral para que la cantidad de alertas coincida con esa capacidad (revisar las N transacciones con mayor score), sin reentrenar. Por ejemplo, mi champion con umbral 0,4 alerta el 32 % de las transacciones: es mucho para revisar, así que habría que subir el umbral.

## Registro, uso y monitoreo

**17.** Sí, se registró en Unity Catalog como `workspace.bigdata_belu_pena.fraud_model`: primero la versión 1 (con el mejor run de ese momento, un bosque) y luego, al aparecer un run mejor, la **versión 2** (el boosting `run8`), que quedó con el alias `champion`. Cargar `models:/...@champion` permite que el proceso consumidor pida "el champion" sin depender de un `run_id`: se puede cambiar de modelo moviendo el alias a otra versión, sin tocar el código, y además se conserva el versionado y el linaje.

**18.** Se puntuaron 7.165 transacciones nuevas y 2.316 generaron alerta (32,3 %). Para auditar cada decisión, `ml_predictions` guarda `transaction_id`, `score`, `alerta`, `threshold`, `model_name`, `model_version`, `run_id` y `scored_at`. El umbral se guarda fuera del modelo porque es una decisión de negocio (depende de la capacidad de revisión y del costo de cada error) que puede cambiar sin reentrenar.

**19.** Sí, el PR-AUC sobre `nuevas` fue parecido al de test: 0,566 contra 0,579 (recall 0,866 contra 0,861 y precision 0,296 contra 0,278). En la campaña simulada el porcentaje de alertas subió de 32,3 % a 50,6 %, con un importe promedio que pasó de 989 a 1.218 y transferencias de 33 % a 61 %. El drift es el cambio en la distribución de los datos o del comportamiento respecto de los de entrenamiento; se puede detectar antes de tener las etiquetas porque alcanza con vigilar las distribuciones de las variables de entrada y de los scores, mientras que las métricas del modelo sólo se pueden calcular cuando llegan los contracargos confirmados.

## Cierre

**20.** Recorrí el ciclo así: datos (`ml_features` con split temporal), entrenamiento y tracking (los 40 runs, el barrido y la comparación en MLflow), registro (versión 2 con alias `champion` en Unity Catalog), uso (inferencia batch en `ml_predictions`) y monitoreo (comparación test contra nuevas y drift simulado). Para llevarlo a producción real faltarían, por ejemplo: automatizar el pipeline y el reentrenamiento (orquestación programada, CI/CD y pruebas), un monitoreo continuo con alertas de drift y de calidad de datos, un despliegue formal (serving en línea o batch programado con proceso de aprobación de versiones) y definir el umbral con el equipo de riesgo según su capacidad y el costo de los errores.
