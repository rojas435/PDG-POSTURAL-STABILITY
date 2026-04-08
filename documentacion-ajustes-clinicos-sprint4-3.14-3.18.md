# Ajustes clinico-tecnicos aplicados a los cuadernos 3.14 y 3.18 (Sprint 4)

## 1. Objetivo final del sprint

Este sprint deja implementado el enfoque final solicitado por tutor para los items MDS-UPDRS 3.14 y 3.18:

1. No usar una sola toma aislada.
2. Estimar 3.14 y 3.18 con el promedio de las pruebas 3.15, 3.16 y 3.17.
3. Reportar estadistica descriptiva (media y desviacion estandar entre pruebas).
4. Clasificar por rangos MDS-UPDRS en escala 0-4.

Ademas, se mantiene la continuidad metodologica con la logica validada previamente en 3.15-3.17.

---

## 2. Fuentes de datos integradas

En lugar de reutilizar un unico JSON por cuaderno, ahora cada cuaderno integra varias capturas:

### 2.1 Cohorte Control

- 3.15: `Datos/Control/3.15-Control.json`
- 3.16: `Datos/Control/3.16-LEFT-Control.json` y `Datos/Control/3.16-RIGHT-Control.json`
- 3.17: `Datos/Control/3.17-Control.json`

### 2.2 Cohorte Parkinson

- 3.15: `Datos/Parkinson/3.15-Parkinson.json`
- 3.16: `Datos/Parkinson/3.16-LEFT-HAND-Parkinson.json` y `Datos/Parkinson/3.16-RIGHT-HAND-Parkinson.json`
- 3.17: `Datos/Parkinson/3.17-Parkinson.json`

---

## 3. Normalizacion de estructura JSON

Se dejo un cargador robusto para cubrir ambos formatos detectados:

1. Registros en `imuData` de nivel superior.
2. Registros anidados en `dgiResults[].imuData`.

Con esto se evita perdida de datos en archivos donde `imuData` superior esta vacio pero existe informacion valida en niveles internos.

---

## 4. Cobertura de IMUs en el enfoque actual

Se valida y reporta cobertura de IMUs esperadas por protocolo:

- LEFT-HAND
- RIGHT-HAND
- LEFT-ANKLE
- RIGHT-ANKLE
- BASE-SPINE

Con la integracion 3.15-3.17 se observa cobertura completa de las 5 IMUs esperadas dentro del conjunto agregado.

---

## 5. Metodologia implementada para 3.14

## 5.1 Logica de analisis

Para cada prueba y ejecucion:

1. Normalizacion de `deviceId`.
2. Recorte por tramo temporal comun por ejecucion.
3. Calculo de metricas por IMU:
   - desviacion de aceleracion (`std_acc_mag`),
   - desviacion de giroscopio (`std_gyro_mag`),
   - jerk medio absoluto,
   - ratio de potencia de movimiento (0.5-3 Hz sobre 0.5-15 Hz),
   - deteccion de temblor con formula validada.

### 5.2 Formula validada reutilizada

Se conserva la deteccion usada en 3.15-3.17:

- Welch PSD,
- banda temblor 3-8 Hz,
- banda total 0.5-15 Hz,
- ratio banda/total,
- RMS en banda filtrada,
- criterio: ratio >= 0.3 y RMS >= umbral (0.02 g acc, 0.5 deg/s gyro).

### 5.3 Agregacion multi-prueba

- Primero se promedia por prueba (3.15, 3.16, 3.17).
- Luego se obtiene promedio global por IMU entre pruebas.
- Se reporta desviacion estandar entre pruebas para variables clave.

### 5.4 Clasificacion por rangos MDS-UPDRS

Se usa el porcentaje promedio de positividad (`positive_test_pct`) para clasificar:

- 0: 0%
- 1: >0% y <=25%
- 2: >25% y <=50%
- 3: >50% y <=75%
- 4: >75%

Campo final: `mds314_score_from_average`.

---

## 6. Metodologia implementada para 3.18

## 6.1 Logica de persistencia temporal

Para cada prueba y ejecucion:

1. Se segmenta en ventanas de 2 s con 50% de solapamiento.
2. Por ventana se evalua presencia de temblor.
3. Se calcula porcentaje temporal de ventanas con temblor (`tremor_pct`).

### 6.2 Agregacion multi-prueba

- Se promedia por prueba y por IMU.
- Luego se promedia entre 3.15, 3.16 y 3.17.
- Se reporta desviacion estandar (`tremor_pct_std`) entre pruebas.

### 6.3 Clasificacion por rangos MDS-UPDRS 3.18

La puntuacion se obtiene del porcentaje promedio de persistencia:

- 0: 0%
- 1: >0% y <=25%
- 2: >25% y <=50%
- 3: >50% y <=75%
- 4: >75%

La puntuacion global se reporta por peor caso entre extremidades disponibles (maximo score).

---

## 7. Salidas agregadas incorporadas

En los cuatro cuadernos quedaron salidas consistentes con el enfoque final:

1. Resumen por prueba (3.15, 3.16, 3.17).
2. Resumen global por IMU con media y desviacion estandar.
3. Porcentaje de positividad/persistencia y score 0-4.
4. Estado de cobertura de IMUs esperadas.

---

## 8. Verificacion tecnica realizada

Se reejecutaron celdas clave en los 4 notebooks:

- imports,
- carga y aplanado multi-fuente,
- preprocesamiento y ventana comun,
- analisis principal,
- resumen final.

Resultado:

1. Ejecucion correcta sin errores de runtime tras los ajustes.
2. Flujo agregado 3.15-3.17 funcionando en 3.14 y 3.18 para Control y Parkinson.
3. Clasificacion por rangos y estadistica descriptiva visible en salida final.

---

## 9. Limitaciones actuales y siguiente fase

Aunque la metodologia final ya esta implementada, se mantiene una limitacion de interpretacion clinica:

- 3.14 y 3.18 se estiman desde pruebas 3.15-3.17 (enfoque proxy), no desde capturas nativas especificas de cada item.

Siguiente fase recomendada:

1. Capturar datasets nativos de 3.14 y 3.18 bajo protocolo clinico estricto.
2. Comparar resultados proxy vs resultados nativos para calibracion.
3. Ajustar umbrales finales con validacion clinica en cohorte ampliada.

---

## 10. Archivos modificados en este sprint

- `Analisis Control/imu-3.14-Control.ipynb`
- `Analisis Parkinson/imu-3.14-Parkinson.ipynb`
- `Analisis Control/imu-3.18-Control.ipynb`
- `Analisis Parkinson/imu-3.18-Parkinson.ipynb`
- `documentacion-ajustes-clinicos-sprint4-3.14-3.18.md`
