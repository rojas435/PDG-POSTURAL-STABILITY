# Análisis de temblor con IMUs – Documentación del proyecto

Este repositorio contiene una serie de cuadernos Jupyter para cargar registros IMU (acelerómetro y giróscopo) en formato JSON, preprocesarlos y detectar temblor en distintas tareas (manos extendidas, tocarse la nariz derecha/izquierda y estabilidad en quietud). Todas las tareas comparten exactamente las mismas reglas de señal y de decisión para que los resultados sean comparables.

## Cuadernos incluidos

- `imu-manos-extendidas.ipynb` (baseline de análisis):
  - Pipeline completo de ingestión, preprocesado, estimación de PSD (Welch), filtrado de banda 3–8 Hz y detección de temblor por dispositivo.
  - Gráficos de tiempo (señal bruta + componente 3–8 Hz) y espectro (PSD) por dispositivo, más un resumen tabular y conclusiones.
- `imu-mano-derecha-nariz.ipynb`:
  - Misma lógica que el baseline, aplicada a la tarea de tocarse la nariz con la mano derecha. Se asume la otra mano como referencia en reposo para la interpretación clínica.
- `imu-mano-izquierda-nariz.ipynb`:
  - Réplica exacta de reglas/umbrales/figuras de la mano derecha, aplicada a la mano izquierda.
- `imu-estabilidad-quietud.ipynb`:
  - Misma lógica para la prueba de reposo con 5 IMUs (LEFT-HAND, RIGHT-HAND, LEFT-ANKLE, RIGHT-ANKLE, BASE-SPINE). Incluye un cargador JSON robusto que extrae señales aunque la estructura cambie (p. ej., `imuData` como string o anidamientos atípicos).

## Datos de entrada (JSON)

Campos esperados por registro (por dispositivo):
- `deviceId` (p. ej., LEFT-HAND, RIGHT-HAND, LEFT-ANKLE, RIGHT-ANKLE, BASE-SPINE)
- `timestamp` (ms)
- `accelerometer`: `{x, y, z}` en g (aprox.)
- `gyroscope`: `{x, y, z}` en deg/s

El cuaderno de quietud implementa extracción recursiva y tolerante a alias de clave (por ej., `device`/`placement`/`name` en lugar de `deviceId`; `time`/`ts` en lugar de `timestamp`).

## Flujo de análisis (común a todos los cuadernos)

1. Ingesta y aplanado por dispositivo; normalización de columnas.
2. Estimación de la tasa de muestreo por dispositivo (mediana de Δt); normalización temporal relativa `t`.
3. Cálculo de magnitudes:
   - Acelerómetro: `acc_mag = sqrt(ax^2 + ay^2 + az^2)`
   - Giróscopo: `gyro_mag = sqrt(gx^2 + gy^2 + gz^2)`
4. Selección de una ventana común de 10 s dentro del solapamiento entre dispositivos, centrada cuando es posible.
5. Detección de temblor por dispositivo y canal (acc/gyro):
   - PSD mediante Welch (SciPy si está disponible; si no, alternativa con NumPy)
   - Potencia en banda de temblor (3–8 Hz) frente a potencia total (0.5–15 Hz)
   - Filtrado pasa-banda 3–8 Hz y RMS de la componente temblor
6. Regla de decisión binaria por canal (temblor SÍ/NO) y fusión por dispositivo.
7. Gráficas de tiempo (bruta + 3–8 Hz) y PSD con sombreado de banda y pico anotado.
8. Tabla-resumen y conclusiones por dispositivo.

## Criterios de detección y reglas

- Banda de temblor: 3–8 Hz.
  - Justificación: el temblor en reposo típico de Parkinson se concentra principalmente en 4–6 Hz y puede extenderse hacia ~8 Hz según medicación o sujeto. Se escoge 3–8 Hz como banda inclusiva que cubre la variabilidad reportada en la literatura.
- Banda total para normalización: 0.5–15 Hz.
  - Cubre el contenido de baja y media frecuencia relevante para postura y movimientos lentos, evitando que altas frecuencias dominen la normalización.
- Métricas por canal (acc o gyro):
  - Potencia en banda (3–8 Hz) y potencia total (0.5–15 Hz) a partir de PSD (Welch). Se calcula el cociente: `ratio = band_power / total_power`.
  - RMS de la señal filtrada 3–8 Hz.
- Umbrales por canal (calibrables por dispositivo):
  - Acelerómetro: `ratio ≥ 0.30` y `RMS ≥ 0.02 g`
  - Giróscopo: `ratio ≥ 0.30` y `RMS ≥ 0.5 deg/s`
  - Razonamiento: el cociente de potencias estabiliza frente a ganancias absolutas distintas entre sensores; el umbral RMS evita falsos positivos por ruido de muy baja amplitud. Los valores propuestos son conservadores para temblores leves y se basan en rangos de amplitud reportados con sensores inerciales y en experiencia práctica; se recomienda ajustar con datos de referencia del propio hardware/población.
- Regla de fusión por dispositivo (`fusion_rule`):
  - Por defecto `gyro` (la decisión final sigue el canal giroscópico), dado que los giroscopios capturan con alta sensibilidad la componente rotacional del temblor y son menos sensibles a la gravedad/quias digitales del acelerómetro.
  - Alternativas implementadas: `acc` (sólo acelerómetro), `either` (acc o gyro), `both` (acc y gyro).

## Métodos de señal

- PSD de Welch: segmentación con ventana y promediado para reducir varianza, ampliamente utilizado para caracterizar bandas de temblor.
- Filtrado 3–8 Hz: Butterworth 4º orden con `filtfilt` (cuando hay SciPy) para respuesta plana en banda y fase cero; alternativa por máscara FFT si SciPy no está disponible.
- Selección de ventana: 10 s balancea estabilidad espectral y rapidez; centrado dentro del solapamiento minimiza sesgos por transitorios.

## Notas específicas por cuaderno

- Manos extendidas:
  - Uso ilustrativo de todo el pipeline; útil como referencia base y para inspección visual de componentes 3–8 Hz en ambos canales.
- Mano derecha tocar nariz:
  - Mismas reglas/umbrales; la mano izquierda se considera en reposo para orientar la interpretación (no afecta a la decisión algorítmica).
- Mano izquierda tocar nariz:
  - Réplica exacta de la mano derecha, permitiendo comparaciones simétricas.
- Estabilidad en quietud:
  - Incluye 5 IMUs. El cargador JSON es más robusto (parsing recursivo, alias de claves, `imuData` como string). Las conclusiones marcan si aparece energía significativa 3–8 Hz en reposo.

## Ajustes y consideraciones prácticas

- Afinado de umbrales:
  - Si se desea mayor sensibilidad en reposo, se puede bajar ligeramente el RMS (p. ej., `acc` 0.015 g; `gyro` 0.4 deg/s). Si hay falsos positivos, subir RMS o el `ratio` a 0.35.
  - Conviene calibrar umbrales con un conjunto de referencia (sujetos sanos vs. pacientes) y el mismo hardware.
- Banda de temblor:
  - En temblor de reposo PD clásico, un estrecho 4–7 Hz puede ser más específico; se mantuvo 3–8 Hz para robustez inter-sujeto.
- Acelerómetro vs. giroscopio:
  - El acelerómetro incluye gravedad y vibraciones lineales; el giroscopio a menudo ofrece SNR superior para temblor rotacional. La fusión es configurable según el contexto clínico.
- Tasa de muestreo:
  - La estimación se hace por mediana de Δt. Si hay aliasing o saturaciones, revisar el hardware o filtrar/limpiar antes de la PSD.
- Duración de la ventana:
  - 10 s es práctico y suficiente para estimar picos 3–8 Hz con buena resolución; si se desea mejor resolución frecuencial, ampliar la ventana.

## Limitaciones

- Umbrales absolutos dependen del sensor, su rango y ruido de fondo; se recomienda recalibrar por modelo de IMU.
- Movimientos voluntarios pueden introducir energía en 3–8 Hz; la normalización por potencia total y el doble criterio con RMS ayudan, pero no lo eliminan por completo.
- El análisis agrupa magnitudes (normas) y no separa ejes; en algunos casos, considerar ejes por separado puede aportar especificidad.

## Referencias (selección)

1. Welch, P. D. (1967). The use of fast Fourier transform for the estimation of power spectra: A method based on time averaging over short, modified periodograms. IEEE Transactions on Audio and Electroacoustics, 15(2), 70–73. doi:10.1109/TAU.1967.1161901
2. Bhatia, K. P., Bain, P., Bajaj, N., et al. (2018). Consensus Statement on the Classification of Tremors, from the Task Force on Tremor of the International Parkinson and Movement Disorder Society. Movement Disorders, 33(1), 75–87. doi:10.1002/mds.27121
3. Salarian, A., Russmann, H., Vingerhoets, F. J. G., Burkhard, P. R., & Aminian, K. (2007). Quantification of tremor and bradykinesia in Parkinson’s disease using a novel ambulatory monitoring system. IEEE Transactions on Biomedical Engineering, 54(2), 313–322. doi:10.1109/TBME.2006.886661
4. Patel, S., Park, H., Bonato, P., Chan, L., & Rodgers, M. (2012). A review of wearable sensors and systems with application in rehabilitation. IEEE Reviews in Biomedical Engineering, 5, 21–44. doi:10.1109/RBME.2012.2204071

Estas fuentes respaldan: (i) el uso de Welch para caracterizar bandas; (ii) rangos de frecuencia típicos del temblor en reposo PD; (iii) la cuantificación con sensores inerciales (acc/gyro) y la conveniencia de métricas normalizadas.

## Cómo ejecutar

1. Abra cada cuaderno en Jupyter (VS Code) y ejecute las celdas en orden.
2. Si usa otros archivos JSON, edite la ruta `json_path` en la celda de carga.
3. Los parámetros clave están visibles en las celdas:
   - Banda de temblor (por defecto `(3, 8)` Hz) y banda total `(0.5, 15)` Hz
   - Regla de fusión `fusion_rule = 'gyro'`
   - Umbrales RMS y de `ratio`
4. Revise las tablas y gráficos para interpretar los resultados por dispositivo.