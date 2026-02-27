# Zabbix Baseline — Algoritmos Nativos de Deteccion de Anomalias

[![GitHub Pages](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-0d1520?style=flat-square&logo=github&logoColor=white&labelColor=1a2740&color=e07b39)](https://simonlexrs.github.io/zabbix-anomaly-detection-baseline/)
[![HTML](https://img.shields.io/badge/HTML-100%25-e07b39?style=flat-square&logo=html5&logoColor=white)](https://github.com/SimonLexRS/zabbix-anomaly-detection-baseline/blob/main/index.html)
[![Chart.js](https://img.shields.io/badge/Chart.js-4.4.1-e07b39?style=flat-square&logo=chartdotjs&logoColor=white)](https://www.chartjs.org/)
[![Views](https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fsimonlexrs.github.io%2Fzabbix-anomaly-detection-baseline%2F&count_bg=%231a2740&title_bg=%23e07b39&icon=eye&icon_color=%23ffffff&title=views&edge_flat=true)](https://hits.seeyoufarm.com)

> Dashboard interactivo que explica y simula el mecanismo nativo de deteccion de anomalias de Zabbix usando **baselinewma (WMA)**, **baselinedev (stddevpop)** y **trendstl (STL + MAD)** — los 3 algoritmos nativos reales de Zabbix 6.0+.

---

## Demo en vivo

**[https://simonlexrs.github.io/zabbix-anomaly-detection-baseline/](https://simonlexrs.github.io/zabbix-anomaly-detection-baseline/)**

---

## Que hace el dashboard

El dashboard es una pagina HTML estatica completamente autocontenida (sin backend) que simula como Zabbix 6.0+ construye un **baseline predictivo** para una metrica de red (CPU Utilization de un BTS — LA PAZ NORTE) usando sus 3 funciones nativas de deteccion de anomalias, permitiendo comparar su comportamiento en tiempo real.

---

## Algoritmos implementados

### 1. baselinewma() — Weighted Moving Average

Calcula el baseline promediando los datos del **mismo periodo en N estaciones anteriores**, con pesos que dan mas importancia a las estaciones mas recientes.

**Sintaxis en Zabbix:**
```
baselinewma(/host/key, 1h:now/h, "w", 4)
```

- `Semana mas reciente` → mayor peso
- `Semana mas antigua` → menor peso
- Ideal para metricas con **patron semanal estable** (trafico de datos BTS, CPU de core network)

**Ejemplo de trigger:**
```
trendavg(/host/key,1h:now/h) > baselinewma(/host/key,1h,"w",4) * 1.5
```
El trafico de esta hora supera 1.5x el promedio de las ultimas 4 semanas.

---

### 2. baselinedev() — Desviacion estandar poblacional (stddevpop)

Retorna **cuantas desviaciones estandar** se aleja el periodo actual respecto a los mismos periodos historicos. Devuelve un numero: `0` = normal, `3` = muy anomalo.

**Sintaxis en Zabbix:**
```
baselinedev(/host/key, 1h:now/h, "d", 10)
```

- Rapido de implementar en dispositivos masivos con bajo overhead
- Ideal para **latencia PING, disponibilidad de servicios**

**Ejemplo de trigger:**
```
baselinedev(/host/key,1h:now/h,"d",10) > 3
```
El valor actual esta a mas de 3sigma del historial de los ultimos 10 dias.

---

### 3. trendstl() — STL Decomposition + MAD

El mas avanzado. Descompone la serie temporal en **Tendencia + Estacionalidad + Residuo** (LOESS), luego aplica MAD (Median Absolute Deviation) al residuo para detectar anomalias. Retorna una `tasa de anomalias` (0.0 a 1.0).

**Sintaxis en Zabbix:**
```
trendstl(/host/key, 28d:now/d, 1d, 7d)
```
Analiza 28 dias, detecta en el ultimo dia, estacionalidad semanal.

- Requiere minimo **28 dias de historial**
- Detecta anomalias sutiles que los otros metodos no ven
- Ideal para **tasa de errores CRC, retransmisiones TCP**

**Ejemplo de trigger:**
```
trendstl(/host/key,28d:now/d,1d,7d) > 0.3
```
Mas del 30% de los valores del ultimo dia son anomalos vs. las ultimas 4 semanas con patron semanal.

---

## Graficos incluidos

### Grafico principal — CPU Utilization (24 h, resolucion 1 min)

Muestra 5 capas de informacion simultaneamente:

| Capa | Color | Descripcion |
|---|---|---|
| **Valor real** | Azul (`#60a5fa`) | CPU % medido por Zabbix cada minuto desde el agente |
| **Baseline** | Naranja (`#e07b39`) punteado | Prediccion del comportamiento normal segun el algoritmo activo |
| **Banda superior** | Relleno naranja (10% opacidad) | `Baseline + sigma x desviacion_historica` |
| **Banda inferior** | Transparente | `Baseline - sigma x desviacion_historica` |
| **Anomalias** | Rojo (`#ef4444`) puntos | Valores que salieron de la banda de confianza |

### Grafico secundario — Comparacion por algoritmo

| Algoritmo activo | Que muestra el grafico secundario |
|---|---|
| **baselinewma** | Baseline WMA vs. N semanas historicas con pesos diferenciados |
| **baselinedev** | Numero de desviaciones estandar (sigma) en cada punto del tiempo |
| **trendstl** | Residuo de la descomposicion STL con umbral MAD |

---

## Panel de control interactivo

Dos sliders permiten modificar los parametros en tiempo real y regenerar ambos graficos:

- **Desviaciones (sigma)**: rango 0.5 a 4.0 — controla la sensibilidad de la deteccion
- **Estaciones / Dias de historial / Umbral MAD**: rango variable segun el algoritmo activo
  - `baselinewma`: Estaciones (semanas), rango 1–8, default 4
  - `baselinedev`: Dias de historial, rango 5–30, default 10
  - `trendstl`: Umbral MAD (sigma equiv.), rango 1–5, default 2

Cada cambio recalcula las anomalias, actualiza los marcadores estadisticos y regenera el registro de eventos.

---

## Estadisticas en tiempo real

- Total de anomalias detectadas
- Tasa de anomalia (% sobre total de puntos)
- Promedio del valor real
- Promedio del baseline

---

## Registro de eventos

Log scrolleable que muestra cada anomalia con timestamp, tipo (`SPIKE↑` / `DROP↓`) y valor en porcentaje. Si sigma es suficientemente alto para cubrir todos los picos, muestra `✓ Sin anomalias con sigma = N`.

**Zonas de anomalia simuladas en el dataset:**

| Timestamp | Tipo | CPU (%) |
|---|---|---|
| 05:30 | SPIKE↑ | ~72% |
| 10:28 | SPIKE↑ | ~68% |
| 15:05 | DROP↓ | ~18% |
| 21:27 | SPIKE↑ | ~79% |

---

## Seccion de conceptos

Tres tarjetas explicativas, una por algoritmo:

| Indice | Algoritmo | Descripcion tecnica |
|---|---|---|
| **W** (naranja) | `baselinewma()` — Weighted Moving Average | Promedio ponderado por estacion, mas peso a semanas recientes |
| **sigma** (verde) | `baselinedev()` — stddevpop | Numero de desviaciones estandar del valor actual vs. historial |
| **STL** (purpura) | `trendstl()` — STL + MAD | Descomposicion LOESS + Median Absolute Deviation sobre el residuo |

---

## Seccion de comparacion y uso en produccion

### Cuado usar cada algoritmo en produccion

| Algoritmo | Mejor uso | Ejemplo real |
|---|---|---|
| `baselinewma` | Metricas con patron semanal estable | Trafico de datos BTS, CPU de core network |
| `baselinedev` | Alertas simples y rapidas de configurar en muchos dispositivos | Latencia PING, disponibilidad de servicios |
| `trendstl` | Metricas complejas con anomalias sutiles | Tasa de errores CRC, retransmisiones TCP |

> ⚠ Los 3 son **univariados** — no correlacionan CPU + RAM + BW simultaneamente. Para correlacion multivariada se necesita un sistema ML externo.

---

## Generacion de datos simulados

El dataset simula 1440 puntos (24 h x 1 min) con:

- **Base estacional**: funcion gaussiana con picos a las ~09:07 h y ~14:53 h (~42% CPU base)
- **Tendencia lineal**: `+0.003 x t` (crecimiento gradual)
- **Ruido aleatorio**: ±3% (distribucion uniforme)
- **Anomalias inyectadas**: 4 zonas con magnitudes de +25 a +30 puntos (spikes) y -22 puntos (drop)

---

## Tecnologias utilizadas

| Tecnologia | Uso |
|---|---|
| HTML5 / CSS3 | Estructura y estilos del dashboard |
| [Chart.js 4.4.1](https://www.chartjs.org/) | Visualizacion de graficos interactivos |
| JavaScript (vanilla) | Logica de simulacion de los 3 algoritmos y controles |
| Google Fonts (Inter + JetBrains Mono) | Tipografia |
| GitHub Pages | Hosting estatico del dashboard |

---

## Estructura del repositorio

```
zabbix-anomaly-detection-baseline/
├── index.html   # Dashboard completo (autocontenido, sin dependencias locales)
└── README.md    # Documentacion
```

---

## Uso local

No requiere instalacion. Simplemente abre `index.html` en cualquier navegador moderno:

```bash
git clone https://github.com/SimonLexRS/zabbix-anomaly-detection-baseline.git
cd zabbix-anomaly-detection-baseline
open index.html      # macOS
# xdg-open index.html  # Linux
```

---

## Autor

**SimonLexRS** — [github.com/SimonLexRS](https://github.com/SimonLexRS)
