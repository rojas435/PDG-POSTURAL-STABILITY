
# Análisis de temblor con IMUs – Documentación del proyecto

Este repositorio contiene cuadernos Jupyter y datos para el análisis de temblor con sensores inerciales (IMU) en sujetos con Parkinson y controles sanos. El flujo cubre desde la carga robusta de archivos JSON, preprocesamiento, análisis exploratorio avanzado y detección de temblor en distintas tareas.

## Estructura del repositorio

- **Datos/Control/**: Archivos JSON de sujetos control (sanos) para pruebas 3.15, 3.16 (LEFT/RIGHT), 3.17.
- **Datos/Parkinson/**: Archivos JSON de sujetos con Parkinson para pruebas 3.15, 3.16 (LEFT/RIGHT), 3.17.
- **Analisis Control/**: Cuadernos de análisis individuales para cada prueba/control.
- **Analisis Parkinson/**: Cuadernos de análisis individuales y EDA para cada prueba/Parkinson.
- **EDA-IMU-COMPLETO.ipynb**: Análisis exploratorio integral de todos los datos y sensores, con carga robusta, visualización, estadística, outliers, correlaciones y patrones de movimiento.

## Cuadernos incluidos

- `EDA-IMU.ipynb`: EDA profesional y robusto para todos los archivos y sensores, integrando controles y Parkinson.
- `Analisis Control/imu-3.15-Control.ipynb`, ...: Análisis por tarea para sujetos control.
- `Analisis Parkinson/imu-3.15-Parkinson.ipynb`, ...: Análisis por tarea para sujetos con Parkinson.
- Otros cuadernos: análisis específicos y pipelines para tareas individuales.

## Datos de entrada (JSON)

Cada archivo JSON contiene registros por dispositivo (sensor):
- `deviceId` (ej: LEFT-HAND, RIGHT-HAND, LEFT-ANKLE, RIGHT-ANKLE, BASE-SPINE)
- `timestamp` (ms)
- `accelerometer`: `{x, y, z}` en g
- `gyroscope`: `{x, y, z}` en deg/s

El EDA y los cuadernos usan un cargador recursivo y tolerante a alias de clave (`device`, `placement`, `name`, `time`, `ts`).

## Flujo de análisis

1. Carga y aplanado robusto de todos los archivos y sensores.
2. Análisis exploratorio: cantidad de datos, cobertura, distribución temporal, gaps.
3. Estadísticas descriptivas por sensor y prueba.
4. Análisis de outliers (boxplots), correlaciones (matriz heatmap) y visualizaciones de patrones (trayectorias ax vs ay).
5. Detección de temblor por dispositivo y canal (acc/gyro):
   - PSD (Welch), potencia en banda 3–8 Hz vs total 0.5–15 Hz, filtrado 3–8 Hz y RMS.
   - Decisión binaria por canal y fusión por dispositivo.
6. Gráficas de tiempo, espectro y tablas resumen.
7. Conclusiones y recomendaciones para análisis avanzados.

## Criterios y reglas de detección

- Banda de temblor: 3–8 Hz (cubre variabilidad típica de temblor en reposo).
- Banda total: 0.5–15 Hz.
- Métricas: potencia en banda, ratio, RMS filtrado.
- Umbrales sugeridos:
  - Acelerómetro: ratio ≥ 0.30 y RMS ≥ 0.02 g
  - Giroscopio: ratio ≥ 0.30 y RMS ≥ 0.5 deg/s
- Regla de fusión: por defecto `gyro` (puede configurarse a `acc`, `either`, `both`).

## Consideraciones prácticas

- Incluye análisis comparativo entre controles y Parkinson.
- El EDA permite identificar diferencias globales, outliers y correlaciones.
- Los pipelines individuales permiten análisis detallado por tarea y grupo.
- Los umbrales y reglas pueden calibrarse según el hardware y la población.

## Ejecución

1. Abre los cuadernos en Jupyter (VS Code) y ejecuta las celdas en orden.
2. Si usas otros archivos JSON, edita la ruta en la celda de carga.
3. Revisa los gráficos y tablas para interpretar resultados por dispositivo, tarea y grupo.

## Referencias

1. Welch, P. D. (1967). The use of fast Fourier transform for the estimation of power spectra: A method based on time averaging over short, modified periodograms. IEEE Transactions on Audio and Electroacoustics, 15(2), 70–73. doi:10.1109/TAU.1967.1161901
2. Bhatia, K. P., Bain, P., Bajaj, N., et al. (2018). Consensus Statement on the Classification of Tremors, from the Task Force on Tremor of the International Parkinson and Movement Disorder Society. Movement Disorders, 33(1), 75–87. doi:10.1002/mds.27121
3. Salarian, A., Russmann, H., Vingerhoets, F. J. G., Burkhard, P. R., & Aminian, K. (2007). Quantification of tremor and bradykinesia in Parkinson’s disease using a novel ambulatory monitoring system. IEEE Transactions on Biomedical Engineering, 54(2), 313–322. doi:10.1109/TBME.2006.886661
4. Patel, S., Park, H., Bonato, P., Chan, L., & Rodgers, M. (2012). A review of wearable sensors and systems with application in rehabilitation. IEEE Reviews in Biomedical Engineering, 5, 21–44. doi:10.1109/RBME.2012.2204071

Estas fuentes respaldan: (i) el uso de Welch para caracterizar bandas; (ii) rangos de frecuencia típicos del temblor en reposo PD; (iii) la cuantificación con sensores inerciales (acc/gyro) y la conveniencia de métricas normalizadas.