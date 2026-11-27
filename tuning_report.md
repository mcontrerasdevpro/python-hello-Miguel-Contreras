# StreamLoop: informe de tuning

## Metodología

- Se cargó el dataset directamente desde la URL del reto.
- Se reemplazaron cadenas vacías o compuestas solo por espacios por `NaN`.
- `TotalCharges` se convirtió a numérico; los valores no convertibles quedan como
  faltantes.
- Se eliminó `customerID`, que es un identificador y no una variable predictiva.
- El dataset se dividió una sola vez, de forma estratificada, en entrenamiento
  (80 %) y test (20 %) antes de ajustar cualquier modelo.
- El preprocesamiento vive dentro del `Pipeline`: imputación mediana para
  numéricas, imputación de la moda y one-hot encoding para categóricas, y
  escalado de numéricas.

El test se utilizó exactamente dos veces: para la línea base y para la
evaluación final. Tanto `RandomizedSearchCV` como `GridSearchCV` recibieron
únicamente `X_train` e `y_train`, con validación cruzada estratificada de cinco
folds y `n_jobs=-1`.

## Métrica y selección

La métrica principal fue `recall`. Para StreamLoop es prioritario detectar a
clientes que realmente cancelarán, aunque esto implique contactar a algunos
clientes que no cancelarán. `precision` y `ROC AUC` se reportan como métricas
complementarias para vigilar el coste de las alertas y la calidad del ranking.

La búsqueda aleatoria exploró `C`, `penalty` y `class_weight`. La búsqueda fina
se concentró alrededor del mejor `C` encontrado:

```text
model__C: 0.001952
model__penalty: l1
model__class_weight: {0: 1, 1: 2}
```

La configuración elegida fue la de mayor recall medio en la búsqueda fina:
`0.8087`, con desviación estándar `0.0237` entre folds. La desviación es
moderada frente al nivel de recall y no justificó sacrificar el promedio por
una opción ligeramente inferior. Como comprobación adicional, las métricas
medias entre folds fueron:

| Métrica | Media CV | Desviación estándar |
|---|---:|---:|
| Recall | 0.8087 | 0.0237 |
| Precision | 0.4142 | 0.0098 |
| ROC AUC | 0.7918 | 0.0092 |

`GridSearchCV` mantuvo `refit=True`, por lo que `best_estimator_` quedó
reentrenado automáticamente con todo el conjunto de entrenamiento. No se
volvió a llamar a `fit` manualmente sobre el mejor estimador.

## Comparación en test

| Modelo | Recall | Precision | ROC AUC |
|---|---:|---:|---:|
| Línea base (LogisticRegression por defecto) | 0.5588 | 0.6572 | 0.8419 |
| Modelo ajustado | 0.8102 | 0.4522 | 0.8084 |

El modelo ajustado aumenta el recall en aproximadamente 25 puntos
porcentuales, detectando muchos más churners. El trade-off es una menor
precision y un ROC AUC algo menor: genera más alertas falsas, pero es coherente
con la prioridad de negocio de no dejar escapar clientes que cancelan. La
decisión debe validarse con el coste económico real de una intervención de
retención antes de pasar a producción.
