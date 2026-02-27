# Zabbix Baseline — Holt-Winters Triple Exponential Smoothing

[![GitHub Pages](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-0d1520?style=flat-square&logo=github&logoColor=white&labelColor=1a2740&color=e07b39)](https://simonlexrs.github.io/zabbix-anomaly-detection-baseline/)
[![HTML](https://img.shields.io/badge/HTML-100%25-e07b39?style=flat-square&logo=html5&logoColor=white)](https://github.com/SimonLexRS/zabbix-anomaly-detection-baseline/blob/main/index.html)
[![Chart.js](https://img.shields.io/badge/Chart.js-4.4.1-ff6384?style=flat-square&logo=chartdotjs&logoColor=white)](https://www.chartjs.org/)
[![Views](https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fsimonlexrs.github.io%2Fzabbix-anomaly-detection-baseline%2F&count_bg=%231a2740&title_bg=%23e07b39&icon=eye&icon_color=%23ffffff&title=views&edge_flat=true)](https://hits.seeyoufarm.com)

> Dashboard interactivo que explica y simula el mecanismo nativo de deteccion de anomalias de Zabbix usando **Holt-Winters Triple Exponential Smoothing**.

---

## Demo en vivo

**[https://simonlexrs.github.io/zabbix-anomaly-detection-baseline/](https://simonlexrs.github.io/zabbix-anomaly-detection-baseline/)**

---

## Que hace el dashboard

El dashboard es una pagina HTML estatica completamente autocontenida (sin backend) que simula como Zabbix construye un **baseline predictivo** para una metrica de red (CPU Utilization de un BTS) y detecta desvios automaticamente.

### Graficos incluidos

#### 1. CPU Utilization — Grafico principal (24 h, resolucion 1 min)

Muestra 5 capas de informacion simultaneamente:

| Capa | Color | Descripcion |
|---|---|---|
| **Valor real** | Azul (`#60a5fa`) | Lo que Zabbix mide cada minuto |
| **Baseline (Holt-Winters)** | Naranja (`#e07b39`) punteado | Prediccion del comportamiento normal |
| **Banda superior** | Relleno naranja (10% opacidad) | `Baseline + σ × desviacion_historica` |
| **Banda inferior** | Transparente | `Baseline − σ × desviacion_historica` |
| **Anomalias** | Rojo (`#ef4444`) puntos | Valores que salieron de la banda de confianza |

El tooltip interactivo muestra el **Δ baseline** (diferencia entre el valor real y la prediccion) en cada punto del tiempo.

#### 2. Descomposicion del Baseline — Nivel, Tendencia y Estacionalidad

Grafico secundario con 3 series:

- **Nivel (α)** — naranja: valor promedio base constante (~42% CPU)
- **Tendencia (β)** — verde: incremento gradual acumulado (`0.003 × t`), eje Y secundario derecho
- **Estacionalidad (γ)** — purpura: patron ciclico diario con picos a las 09:07 h y 14:53 h

### Panel de control interactivo

Dos sliders permiten modificar los parametros en tiempo real y regenerar ambos graficos:

- **Ancho de banda (σ)**: rango 0.5 a 4.0 — controla la sensibilidad de la deteccion
- **Periodo de estacionalidad**: rango 60 a 1440 minutos — cambia el ciclo del patron

Cada cambio recalcula las anomalias, actualiza el marcador estadistico y regenera el registro de eventos.

### Estadisticas en tiempo real

- Total de anomalias detectadas
- Tasa de anomalia (% sobre total de puntos)
- Promedio del valor real
- Promedio del baseline

### Registro de eventos

Log scrolleable que muestra cada anomalia con timestamp, tipo (`SPIKE ↑` / `DROP ↓`) y valor en porcentaje. Si σ es suficientemente alto para cubrir todos los picos, muestra `✓ Sin anomalias`.

### Seccion de conceptos

Tres tarjetas explicativas con los parametros α, β y γ: que representa cada uno, como influye su valor alto/bajo y ejemplos de casos reales en infraestructura de red.

### Seccion de formulas y configuracion en Zabbix

Referencia tecnica con las expresiones exactas usadas en Zabbix 6.x (ver seccion siguiente).

---

## Como calcula el Baseline — Holt-Winters en Zabbix

### Concepto general

Zabbix implementa **Holt-Winters Triple Exponential Smoothing** como algoritmo nativo de prediccion de series temporales. El modelo descompone cualquier metrica en tres componentes independientes que se actualizan continuamente con cada nueva observacion.

### Los tres componentes

#### α — Nivel (Level smoothing)

Representa el **valor promedio suavizado** de la metrica en el momento actual. Se actualiza exponencialmente: las observaciones recientes tienen mayor peso que las antiguas.

- `α alto` (cercano a 1) → el baseline reacciona rapido a cambios, memoria corta
- `α bajo` (cercano a 0) → el baseline es estable, prioriza el historial largo

#### β — Tendencia (Trend smoothing)

Captura si la metrica tiene una **direccion sostenida** en el tiempo. Detecta crecimiento o decremento gradual aunque el ruido lo oculte en el corto plazo.

- Ejemplo: trafico que crece 2% mensual por aumento de usuarios activos en la red
- `β alto` → detecta nuevas tendencias rapidamente
- `β bajo` → tendencia suave, menos reactividad al ruido puntual

#### γ — Estacionalidad (Seasonal smoothing)

Captura **patrones ciclicos repetitivos**: el trafico sube a las 9 AM y baja de madrugada cada dia; el CPU sube cada lunes a las 8 AM, etc.

- El periodo define el ciclo: `1440` = diario (1440 min), `10080` = semanal
- `γ alto` → los factores estacionales se actualizan rapidamente
- `γ bajo` → patron estacional mas conservador y estable

### Formula de prediccion

```
Prediccion en t+1:
Y_hat(t+1) = (Nivel_t + Tendencia_t) x Factor_Estacional_t

Banda de confianza:
Limite_Superior = Y_hat(t+1) + sigma x Desviacion_historica
Limite_Inferior = Y_hat(t+1) - sigma x Desviacion_historica

Criterio de anomalia:
Valor_real > Limite_Superior  -->  SPIKE (pico hacia arriba)
Valor_real < Limite_Inferior  -->  DROP  (caida hacia abajo)
```

Donde `sigma` (σ) es el multiplicador de desviacion estandar que controla el ancho de la banda de confianza. A mayor σ, menos alertas pero mayor riesgo de omitir anomalias reales.

### Configuracion en Zabbix 6.x

#### Funcion de deteccion de anomalias (para triggers)

```
anomalydetection(/host/key, 1h:now/h)
```

Devuelve `1` si el valor actual esta fuera del rango predicho por el baseline. Se usa directamente en expresiones de trigger.

**Ejemplo de trigger completo:**
```
{host:system.cpu.util.anomalydetection(1h:now/h)}=1
```

#### Funcion de desviacion estandar del baseline (para dashboards)

```
baselinedev(/host/key, 1w:now/w, 2)
```

Devuelve cuantas desviaciones estandar se aleja el valor actual del baseline semanal. El tercer parametro `2` es el multiplicador σ.

#### Parametros del algoritmo en Zabbix GUI

Se configuran en `Configuration → Hosts → Items → [item] → Preprocessing` o en las opciones del item al activar el baseline:

| Parametro | Descripcion | Valor tipico |
|---|---|---|
| `Alpha (α)` | Suavizado del nivel | 0.1 – 0.3 |
| `Beta (β)` | Suavizado de tendencia | 0.1 – 0.2 |
| `Gamma (γ)` | Suavizado estacional | 0.1 – 0.3 |
| `Season` | Periodo del ciclo (en intervalos de datos) | 1440 (diario a 1 min) |

### Limitacion critica del metodo nativo

> ⚠️ **Univariado**: Zabbix analiza cada metrica de forma aislada. No puede correlacionar multiples metricas simultaneamente (CPU + RAM + Ancho de banda). Para correlacion multivariada se requiere integracion con herramientas externas (Grafana + ML, Elasticsearch, etc.).

---

## Tecnologias utilizadas

| Tecnologia | Uso |
|---|---|
| HTML5 / CSS3 | Estructura y estilos del dashboard |
| [Chart.js 4.4.1](https://www.chartjs.org/) | Visualizacion de graficos interactivos |
| JavaScript (vanilla) | Logica de simulacion Holt-Winters y controles |
| Google Fonts (Inter + JetBrains Mono) | Tipografia |
| GitHub Pages | Hosting estatico del dashboard |

---

## Estructura del repositorio

```
zabbix-anomaly-detection-baseline/
├── index.html     # Dashboard completo (autocontenido, sin dependencias locales)
└── README.md      # Documentacion
```

---

## Uso local

No requiere instalacion. Simplemente abre `index.html` en cualquier navegador moderno:

```bash
git clone https://github.com/SimonLexRS/zabbix-anomaly-detection-baseline.git
cd zabbix-anomaly-detection-baseline
open index.html   # macOS
# xdg-open index.html  # Linux
```

---

## Autor

**SimonLexRS** — [github.com/SimonLexRS](https://github.com/SimonLexRS)
