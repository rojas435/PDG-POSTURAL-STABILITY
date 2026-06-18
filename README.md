
# Análisis de temblor con IMUs — Proyecto de Grado PDG

Este repositorio contiene cuadernos Jupyter y datos JSON para el análisis de temblor e inestabilidad postural mediante sensores inerciales (IMU) en sujetos con Parkinson y controles sanos. El flujo completo cubre: carga robusta de JSON heterogéneos, preprocesamiento de señales, análisis espectral (Welch PSD), detección de temblor en bandas de frecuencia, y clasificación MDS-UPDRS (items 3.14–3.18).

---

## Requisitos e instalación

**Python 3.10+** requerido. Proyecto verificado en Python 3.12.

### Instalación rápida (entorno virtual recomendado)

```powershell
python -m venv .venv
.venv\Scripts\Activate
pip install -r requirements.txt
```

### Dependencias

| Paquete | Obligatorio | Propósito |
|---------|-------------|-----------|
| `numpy` | ✅ | Cómputo numérico, FFT, álgebra lineal |
| `pandas` | ✅ | DataFrames, agregaciones, manipulación de datos |
| `matplotlib` | ✅ | Visualización de señales, PSD, gráficos de resultados |
| `seaborn` | ✅ | Estilos de gráficos (`seaborn-v0_8`), heatmaps |
| `scipy` | ❌ * | Welch PSD, filtro Butterworth (`scipy.signal`) |
| `plotly` | ❌ * | Gráficos interactivos alternativos |

\* *Scipy y plotly son opcionales. Si no están instalados, los notebooks utilizan implementaciones puras con NumPy (Welch, filtrado en frecuencia).*

### Verificación

```powershell
python -c "import numpy, pandas, matplotlib, seaborn; print('OK')"
```

---

## Estructura del repositorio

```
.
├── Datos/                          # Datos IMU en JSON
│   ├── Control/                    #   Sujetos sanos
│   │   ├── 3.15-Control.json
│   │   ├── 3.16-LEFT-Control.json
│   │   ├── 3.16-RIGHT-Control.json
│   │   └── 3.17-Control.json
│   └── Parkinson/                  #   Sujetos con Parkinson
│       ├── 3.15-Parkinson.json
│       ├── 3.16-LEFT-HAND-Parkinson.json
│       ├── 3.16-RIGHT-HAND-Parkinson.json
│       └── 3.17-Parkinson.json
├── Analisis Control/               # Notebooks por prueba — cohorte Control
│   ├── imu-3.14-Control.ipynb      #   Bradicinesia global (promedio 3.15-3.17)
│   ├── imu-3.15-Control.ipynb      #   Temblor postural brazos extendidos
│   ├── imu-3.16-LEFT-Control.ipynb #   Temblor cinético mano-izquierda-nariz
│   ├── imu-3.16-RIGHT-Control.ipynb#   Temblor cinético mano-derecha-nariz
│   ├── imu-3.17-Control.ipynb      #   Estabilidad en quietud (5 IMUs)
│   └── imu-3.18-Control.ipynb      #   Temblor cinético global (promedio 3.15-3.17)
├── Analisis Parkinson/             # Notebooks por prueba — cohorte Parkinson
│   ├── EDA-IMU.ipynb               #   Análisis exploratorio de datos
│   ├── imu-3.14-Parkinson.ipynb    #   Bradicinesia global (promedio 3.15-3.17)
│   ├── imu-3.15-Parkinson.ipynb    #   Temblor postural brazos extendidos
│   ├── imu-3.16-LEFT-Parkinson.ipynb#  Temblor cinético mano-izquierda-nariz
│   ├── imu-3.16-RIGHT-Parkinson.ipynb# Temblor cinético mano-derecha-nariz
│   ├── imu-3.17-Parkinson.ipynb    #   Estabilidad en quietud (5 IMUs)
│   └── imu-3.18-Parkinson.ipynb    #   Temblor cinético global (promedio 3.15-3.17)
├── Modelos Descartados/
│   └── comparacion-baselines-vs-modelo-analitico.ipynb  # Benchmark de enfoques
├── requirements.txt                # Dependencias Python
├── README.md                       # Este archivo
├── AGENTS.md                       # Instrucciones para asistentes de código
├── opencode.json                   # Configuración de OpenCode
└── .gitignore
```

---

## Datos de entrada (formato JSON)

Cada archivo JSON contiene registros de sensores IMU muestreados durante pruebas clínicas estandarizadas MDS-UPDRS (items 3.15, 3.16, 3.17).

### Columnas del DataFrame aplanado

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `deviceId` | str | Identificador del sensor: `BASE-SPINE`, `LEFT-HAND`, `RIGHT-HAND`, `LEFT-ANKLE`, `RIGHT-ANKLE` |
| `timestamp_ms` | float | Marca temporal en milisegundos |
| `ax`, `ay`, `az` | float | Aceleración en ejes X, Y, Z (en g) |
| `gx`, `gy`, `gz` | float | Velocidad angular en ejes X, Y, Z (en deg/s) |

### Estructuras JSON soportadas

El cargador robusto maneja dos formatos automáticamente:

1. **Plano:** Registros dentro de `imuData[]` en el nodo raíz del JSON.
2. **Anidado:** Registros dentro de `dgiResults[].imuData[]` (usado por algunos archivos de la prueba 3.17).

Además, normaliza alias de campo:
- `device`, `placement`, `name` → `deviceId`
- `time`, `ts` → `timestamp`

---

## Catálogo de notebooks

### Análisis exploratorio

| Notebook | Contenido |
|----------|-----------|
| `Analisis Parkinson/EDA-IMU.ipynb` | Carga multiarchivo, cobertura de sensores, gaps temporales, estadísticas descriptivas por prueba, boxplots, matriz de correlación, trayectorias ax vs ay. Solo EDA — no realiza detección de temblor. |

### Prueba 3.15 — Temblor postural (brazos extendidos)

Evalúa el temblor de acción con brazos extendidos al frente durante ~10 segundos. Cada notebook procesa un solo archivo JSON y calcula:

1. Carga y aplanado del JSON.
2. Normalización de `deviceId` y selección de tramo temporal común.
3. Cálculo por dispositivo:
   - Magnitudes `acc_mag`, `gyro_mag`.
   - PSD (Welch) con `nperseg=1024`.
   - Potencia en banda de temblor **3–8 Hz** y banda total **0.5–15 Hz**.
   - Ratio banda/total y RMS en señal filtrada pasa banda 3–8 Hz.
4. **Regla de decisión:** `(ratio >= 0.30) AND (RMS >= umbral)` donde umbral ACC=0.02 g, GYRO=0.5 deg/s.
5. **Fusión:** Por defecto `gyro` (configurable a `acc`, `either`, `both`).
6. Score MDS-UPDRS por dispositivo basado en RMS giroscópico y flag de temblor.
7. Tablas resumen y gráficos (señales en tiempo, PSD, métricas comparativas).

### Prueba 3.16 — Temblor cinético (dedo-nariz)

Evalúa el temblor de acción durante el movimiento alternante mano‑izquierda‑nariz / mano‑derecha‑nariz. El flujo de análisis es idéntico al de 3.15.

### Prueba 3.17 — Estabilidad en quietud

Evalúa el temblor en reposo con el paciente sentado y 5 IMUs colocadas en: BASE-SPINE, LEFT-HAND, RIGHT-HAND, LEFT-ANKLE, RIGHT-ANKLE. Utiliza el cargador recursivo (`_extract_imu_records`) para manejar formatos con datos anidados en `dgiResults[].imuData`.

### Pruebas 3.14 y 3.18 — Agregación multi-prueba

Estos notebooks **no cargan un solo JSON**, sino que integran los resultados de 3.15 + 3.16 + 3.17 para estimar los items 3.14 y 3.18 respectivamente.

**Metodología (acordada con tutor):**
1. No usar una sola toma aislada.
2. Estimar 3.14 y 3.18 como promedio de las pruebas 3.15, 3.16 y 3.17.
3. Reportar estadística descriptiva (media y desviación estándar entre pruebas).
4. Clasificar por rangos MDS-UPDRS en escala 0–4.

#### 3.14 — Bradicinesia global (espontaneidad del movimiento)

Por cada prueba y ejecución:
1. Normalización de `deviceId`.
2. Recorte por tramo temporal común por ejecución.
3. Métricas por IMU: `std_acc_mag`, `std_gyro_mag`, jerk medio absoluto, ratio de potencia de movimiento (0.5–3 Hz / 0.5–15 Hz), detección de temblor.
4. Promedio por prueba (3.15, 3.16, 3.17) y luego promedio global por IMU.
5. Clasificación según `positive_test_pct`:
   - 0: 0% | 1: >0–25% | 2: >25–50% | 3: >50–75% | 4: >75%
6. Score global: máximo score entre IMUs.

#### 3.18 — Temblor cinético (persistencia temporal)

Por cada prueba y ejecución:
1. Segmentación en ventanas de 2 s con 50% de solapamiento.
2. Por cada ventana se evalúa presencia de temblor.
3. `tremor_pct`: porcentaje de ventanas con temblor.
4. Promedio entre pruebas 3.15, 3.16, 3.17.
5. Clasificación MDS-UPDRS en escala 0–4 (mismos rangos que 3.14).
6. Score global: peor caso entre extremidades (max score).

### Modelos Descartados

| Notebook | Contenido |
|----------|-----------|
| `Modelos Descartados/comparacion-baselines-vs-modelo-analitico.ipynb` | Benchmark de 4 enfoques de detección: (1) umbral de varianza temporal, (2) pico de frecuencia FFT directo, (3) PSD + filtrado original, (4) PSD + filtrado calibrado. Compara accuracy, F1, precisión y recall contra todos los archivos del dataset. |

---

## Parámetros de detección de temblor

| Parámetro | Valor | Fundamento clínico |
|-----------|-------|-------------------|
| Banda de temblor | 3–8 Hz | Rango típico del temblor en reposo Parkinson (Bhatia et al., 2018) |
| Banda total | 0.5–15 Hz | Cubre movimiento voluntario + temblor |
| Banda de movimiento | 0.5–3 Hz | Movimiento voluntario (para ratio 3.14) |
| Umbral ACC | ratio ≥ 0.30 y RMS ≥ 0.02 g | Calibrado empíricamente sobre datos piloto |
| Umbral GYRO | ratio ≥ 0.30 y RMS ≥ 0.5 deg/s | Calibrado empíricamente sobre datos piloto |
| Regla de fusión | `gyro` (por defecto) | Giroscopio más sensible al temblor fino |

Los umbrales y reglas son configurables dentro de cada notebook modificando los parámetros de `detect_tremor()` y la regla de fusión.

---

## Flujo de ejecución recomendado

```powershell
# 1. Activar entorno virtual
.venv\Scripts\Activate

# 2. Iniciar Jupyter
jupyter notebook
# o en VS Code: abrir carpeta y ejecutar celdas
```

**Orden sugerido para entender el proyecto:**

1. `Analisis Parkinson/EDA-IMU.ipynb` — entender los datos.
2. `Analisis Control/imu-3.15-Control.ipynb` — detección básica en control.
3. `Analisis Parkinson/imu-3.15-Parkinson.ipynb` — comparar con Parkinson.
4. `Analisis Control/imu-3.14-Control.ipynb` — agregación multi-prueba.
5. `Analisis Control/imu-3.18-Control.ipynb` — persistencia temporal.
6. `Modelos Descartados/comparacion-baselines-vs-modelo-analitico.ipynb` — benchmark.

**Nota:** Todos los notebooks cargan datos desde rutas relativas (`../Datos/`). No es necesario configurar rutas adicionales.

---

## Referencias bibliográficas

1. Welch, P. D. (1967). *The use of fast Fourier transform for the estimation of power spectra: A method based on time averaging over short, modified periodograms.* IEEE Transactions on Audio and Electroacoustics, 15(2), 70–73. doi:10.1109/TAU.1967.1161901
2. Bhatia, K. P., Bain, P., Bajaj, N., et al. (2018). *Consensus Statement on the Classification of Tremors, from the Task Force on Tremor of the International Parkinson and Movement Disorder Society.* Movement Disorders, 33(1), 75–87. doi:10.1002/mds.27121
3. Salarian, A., Russmann, H., Vingerhoets, F. J. G., Burkhard, P. R., & Aminian, K. (2007). *Quantification of tremor and bradykinesia in Parkinson's disease using a novel ambulatory monitoring system.* IEEE Transactions on Biomedical Engineering, 54(2), 313–322. doi:10.1109/TBME.2006.886661
4. Patel, S., Park, H., Bonato, P., Chan, L., & Rodgers, M. (2012). *A review of wearable sensors and systems with application in rehabilitation.* IEEE Reviews in Biomedical Engineering, 5, 21–44. doi:10.1109/RBME.2012.2204071

Estas fuentes respaldan: (i) el uso del método de Welch para caracterizar bandas espectrales; (ii) los rangos de frecuencia típicos del temblor en reposo en Parkinson (3–8 Hz); (iii) la cuantificación con sensores inerciales (acelerómetro + giroscopio) y el uso de métricas normalizadas (ratio de potencia).

---

## Limitaciones y trabajo futuro

- **Enfoque proxy:** 3.14 y 3.18 se estiman a partir de pruebas 3.15–3.17, no de capturas nativas específicas. Esto es una limitación metodológica conocida.
- **Cohorte pequeña:** Datos de un solo paciente por cohorte. Se requiere validación en cohortes ampliadas.
- **Umbrales:** Los criterios ratio ≥ 0.30 y RMS ≥ umbral fueron calibrados sobre datos piloto. Deben recalibrarse con validación clínica formal.
- **Próximo paso recomendado:** Capturar datasets nativos de 3.14 y 3.18 bajo protocolo clínico estricto, comparar resultados proxy vs nativos, y ajustar umbrales finales.

---

## Declaración de uso de IAG (Inteligencia Artificial Generativa)

En cumplimiento del nivel 4 del programa del curso, se declara que este proyecto utilizó herramientas de IAG en las siguientes actividades:

| Actividad | Herramienta IAG | Propósito |
|-----------|----------------|-----------|
| Generación de código | GitHub Copilot, OpenCode (deepseek-v4-flash) | Implementación de carga de datos, preprocesamiento de señales, PSD (Welch), filtrado, detección de temblor, clasificación MDS-UPDRS |
| Documentación | OpenCode | Generación de README, AGENTS.md, documentación técnica |
| Depuración | OpenCode | Identificación y corrección de errores en notebooks |
| Estructuración | OpenCode | Organización de celdas, flujo de análisis, documentación IAG |

**Principio aplicado:** Todo código generado por IAG fue revisado, validado y ajustado por los autores para garantizar corrección técnica y alineación con los objetivos clínicos. Las decisiones metodológicas (umbrales, bandas de frecuencia, criterios MDS-UPDRS) fueron definidas por los autores basándose en la literatura clínica referenciada.

**Citación recomendada:**
> Herramientas de IAG (GitHub Copilot, OpenCode) utilizadas en el desarrollo del proyecto *"Análisis de temblor con sensores inerciales (IMU) para la detección de inestabilidad postural en pacientes con Parkinson"*, proyecto de grado PDG, 2025-2026.

Los comentarios `# IAG:` dentro del código fuente de los notebooks identifican las secciones específicas asistidas por estas herramientas.