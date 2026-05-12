# Predicción de ETA en Glovo París

**Trabajo Final — Master en Data Analytics for Business**

Sistema completo de predicción del tiempo de entrega para pedidos de Glovo París, basado en ~65.000 pedidos reales con marcas temporales detalladas.

---

## El problema

Cuando un cliente abre la app de Glovo y mira un restaurante, ve un número: el **tiempo estimado de entrega (ETA)**. Ese número condiciona si compra o no, qué espera, y cómo evalúa el servicio cuando llega el pedido.

El coste de equivocarse es **asimétrico**: prometer 30 min y entregar en 25 genera satisfacción; prometer 30 min y entregar en 45 genera quejas, refunds y churn.

---

## Dataset

- **Pedidos:** 63.646 pedidos completados de Glovo París (~3 meses)
- **Variables:** 21 columnas originales — temporales, geográficas, del courier, del restaurante y del pedido

### Variables target derivadas

| Variable | Cálculo | Significado |
|----------|---------|-------------|
| `total_time` | delivery − activation | **El que percibe el cliente — nuestro target** |
| `prep_time` | pickup − activation | Tiempo en el restaurante |
| `travel_time` | delivery − pickup | Tiempo del courier en la calle |
| `dispatch_delay` | courier_started − activation | Tiempo hasta que se asigna courier |

---

## Metodología — 5 fases

```
1. Auditoría de datos     → calidad, nulos, valores imposibles
2. Análisis exploratorio  → distribución del target, drivers, correlaciones
3. Ingeniería de variables → distancia haversine, geografía París, temporales
4. Comparación de modelos → 6 modelos en complejidad creciente
5. Interpretación final   → importancia de variables, limitaciones
```

### Validación: split temporal, no aleatorio

El último 20% del dataset (ordenado por fecha) se reserva como test, replicando el comportamiento en producción: entrenar con el pasado, predecir el futuro.

---

## Modelos comparados

| # | Modelo | MAE (min) | RMSE (min) | R² | Δ vs baseline |
|---|--------|-----------|------------|-----|---------------|
| 1 | Baseline (mediana) | 12.29 | 17.21 | -0.099 | — |
| 2 | Ridge | 9.17 | 12.59 | 0.412 | -3.12 min |
| 3 | Random Forest | 9.16 | 12.55 | 0.416 | -3.13 min |
| 4 | LightGBM básico | 9.02 | 12.35 | 0.434 | -3.26 min |
| 5 | LightGBM + categóricas | 8.88 | 12.20 | 0.448 | -3.41 min |
| **6** | **LightGBM + categóricas + saturation** | **6.43** | **8.21** | **0.750** | **-5.85 min** |

> **Modelo final:** LightGBM con manejo nativo de variables categóricas y variable `saturation`.
> Mejora de **~5.9 minutos** sobre el baseline.

---

## Hallazgos principales

### Contraintuitivos

1. **El problema operacional está en la mañana, no en lunch/dinner.** La mediana a las 8h es ~46 min; en lunch y dinner (~34 min) es de los valores más bajos del día. La flota se escala bien en horas pico; el desequilibrio aparece cuando hay poca demanda pero también poquísimos couriers activos.

2. **Viernes es el día más lento de la semana** (37.1 min), no el fin de semana. Sábado es el más rápido (34.0 min).

3. **Bicicletas y motos rinden prácticamente igual** (35.0 vs 35.2 min). En París, las bicis aprovechan carriles dedicados y atajos que compensan su menor velocidad punta.

4. **Lunch y dinner NO son lentos** — al contrario, son los momentos más rápidos. La intuición habitual sobre "horas pico de delivery" no se sostiene en estos datos.

### El caso `saturation`

`saturation` correlaciona **0.78** con `total_time` — un valor inusualmente alto que inicialmente sugería data leakage. Se aplicaron tres tests rigurosos:

| Test | Hipótesis si fuera leakage | Resultado |
|------|---------------------------|-----------|
| Heurístico (ciclo demanda) | Debería seguir picos lunch/dinner | Plano — *pero la heurística asume oferta constante, premisa falsa con flota escalable* |
| Within-hour correlation | Debería caer a ~0 al fijar hora | **0.73–0.85** — información residual real |
| Residualización | Correlación de residuos debería colapsar | **0.777 vs 0.778** — prácticamente idéntica |

**Veredicto:** `saturation` es legítima. Captura el ratio carga/capacidad de la flota, no la demanda absoluta. Excluirla habría costado 2.5 minutos de MAE injustificadamente.

### Importancia de variables (modelo final)

1. `saturation` — ~17% (palanca más fuerte; presión operacional en tiempo real)
2. `distance_km` — distancia física entre restaurante y cliente
3. `dist_to_center` — proximidad al Périphérique de París
4. `store_name` — identidad del restaurante (captura variabilidad de cocina)
5. `hour` / `dow` — hora y día de la semana

---

## Stack tecnológico

```
Python 3.x
├── pandas / numpy          — manipulación de datos
├── matplotlib / seaborn    — visualización
├── scipy / sklearn         — estadística y modelos base
└── lightgbm                — modelo final (gradient boosting)
```

---

## Archivos del repositorio

| Archivo | Descripción |
|---------|-------------|
| `glovo_eta_master.ipynb` | Notebook principal — análisis completo, código y visualizaciones |
| `glovo_eta_documento_tecnico.docx` | Documento técnico complementario — metodología y decisiones |
| `Presentacion_Final_de Glovo_ETA_ (1).pdf` | Presentación final del proyecto |

---

## Cómo ejecutar el notebook

1. Clona el repositorio
2. Instala las dependencias:
   ```bash
   pip install lightgbm pandas numpy matplotlib seaborn scikit-learn scipy
   ```
3. Coloca el dataset `glovo_ops_data_final.csv` en la raíz del proyecto
4. Abre y ejecuta `glovo_eta_master.ipynb`

> El notebook también puede ejecutarse en **Google Colab** — incluye carga de archivo automática al inicio.

---

## Limitaciones

- Cobertura temporal de **solo 3 meses** — no captura estacionalidad anual
- **Solo París** — los patrones geográficos no son extrapolables directamente
- **Distancia haversine**, no real por carretera (OSRM mejoraría la predicción)
- Sin datos de meteorología ni eventos externos (huelgas, festivos, partidos)
- Sin información del courier individual (experiencia, historial)

---

## Trabajo futuro

1. Confirmar pipeline de `saturation` con el equipo de datos de Glovo
2. Investigar el pico de mañana (8–11h) con datos adicionales
3. Integrar OSRM para distancia real por carretera
4. Añadir datos meteorológicos (OpenWeather API)
5. Reentrenamiento periódico (semanal/mensual) para capturar deriva temporal
6. Modelado segmentado por vertical (WALL Partner, NonPartner, COURIER tienen dinámicas distintas)

---

*Reto de Inteligencia Artificial — Master en Data Analytics for Business*
