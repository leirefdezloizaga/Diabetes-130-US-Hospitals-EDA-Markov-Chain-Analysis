# Diabetes-130-US-Hospitals-EDA-Markov-Chain-Analysis
Análisis exploratorio y modelo de cadena de Markov sobre trayectorias de tratamiento antidiabético en pacientes hospitalizados (1999–2008).
## Dataset

**Diabetes 130-US Hospitals for years 1999–2008** — UCI Machine Learning Repository  
→ [https://archive.ics.uci.edu/ml/datasets/Diabetes+130-US+hospitals+for+years+1999-2008](https://archive.ics.uci.edu/ml/datasets/Diabetes+130-US+hospitals+for+years+1999-2008)

- **101.766 encuentros hospitalarios** en 130 hospitales de EEUU
- **50 variables**: datos del paciente, diagnósticos, medicación, y resultado del alta
- **Variable objetivo**: reingreso (`<30 días`, `>30 días`, `No`)
- Fuente original: Health Facts database (Cerner Corporation) — datos anonimizados

---

## Estructura del Proyecto

```
├── diabetic_data.csv               # Dataset original
├── diabetes_analysis.py           # EDA completo + 5 niveles de análisis
├── markov_diabetes.py              # Modelo de cadena de Markov
└── README.md
```

## Parte 1 — Análisis de Negocio

¿QUÉ ES UNA CADENA DE MARKOV?
 Un sistema donde el estado futuro depende SOLO del estado actual,
 no de la historia pasada. Perfecto para modelar cómo un paciente
 pasa de un tratamiento a otro entre hospitalizaciones sucesivas.

### ¿QUÉ MODELAMOS?
Cada hospitalización de un paciente diabético lo sitúa en uno de
 4 estados según su tratamiento y si terminó reingresando:
  · SIN_TTO   → no recibe ningún antidiabético
  · ORAL      → recibe antidiabético oral (metformina, glipizide…)
  · INSULINA  → recibe insulina (sola o combinada)
  · REINGRESO → fue readmitido en menos de 30 días

### PREGUNTAS QUE QUEREMOS RESPONDER:
 1. STEADY STATE: A largo plazo, ¿cuál es la cuota de mercado de
    cada terapia? ¿Hacia dónde converge el sistema?

 2. ABSORCIÓN: ¿Cuántas hospitalizaciones espera un paciente antes
   de reingresar? ¿El tratamiento oral "protege" más que la insulina?

 3. PROBABILIDAD DIRECTA: ¿Desde qué estado es más probable que
   ocurra un reingreso en el siguiente ingreso?

4. SIMULACIÓN: Si una cohorte de 1.000 pacientes empieza sin
   tratamiento, ¿dónde acaban después de N hospitalizaciones?

### HIPÓTESIS A DEMOSTRAR:
 H1: Los pacientes SIN tratamiento tienen mayor tasa directa de reingreso
 H2: La insulina domina el steady state (mercado crónico maduro)
 H3: Después de un reingreso, los médicos intensifican: el estado
   post-REINGRESO tiene más insulina que el promedio

### FUENTE DE TRANSICIONES:
 Usamos los pacientes con múltiples encuentros hospitalarios.
 Ordenamos sus encuentros por encounter_id (proxy temporal) y
 tomamos los pares consecutivos (encuentro t → encuentro t+1)..

---

## Parte 2 — Cadena de Markov

### ¿Por qué una cadena de Markov?

Queremos modelar cómo evoluciona el **estado de tratamiento** de un paciente entre hospitalizaciones sucesivas. Cada encuentro se asigna a uno de 4 estados:

| Estado | Definición |
|--------|------------|
| `SIN_TTO` | No recibe ningún antidiabético |
| `ORAL` | Recibe antidiabético oral (metformina, glipizide…) |
| `INSULINA` | Recibe insulina (sola o combinada) |
| `REINGRESO` | Fue readmitido en menos de 30 días |

Las transiciones se extraen de los **30.248 pares de encuentros consecutivos** disponibles en pacientes con más de una hospitalización.

---

### Resultados

#### Matriz de Transición

```
              SIN_TTO    ORAL    INSULINA  REINGRESO
SIN_TTO         ?         ?         ?         14.2%
ORAL            ?         ?         ?         14.5%
INSULINA        ?         ?         ?         15.9%
REINGRESO       ?         ?         ?          ?
```
*(ver heatmap completo en `markov_01_matriz.png`)*

#### Steady State — Cuota de mercado de largo plazo

| Estado | % largo plazo |
|--------|---------------|
| SIN_TTO | 17.1% |
| ORAL | 16.6% |
| **INSULINA** | **50.1%** |
| REINGRESO | 16.2% |

#### Tiempo esperado hasta el primer reingreso

| Estado inicial | Hospitalizaciones esperadas |
|----------------|----------------------------|
| SIN_TTO | 6.67 |
| ORAL | 6.64 |
| INSULINA | 6.48 |

#### Probabilidad directa de reingreso (siguiente ingreso)

| Estado | P(→ REINGRESO) |
|--------|---------------|
| SIN_TTO | 14.2% |
| ORAL | 14.5% |
| INSULINA | 15.9% |

#### Post-reingreso
Tras un reingreso, el uso de insulina **disminuye** 2 pp respecto a la distribución basal.  
→ El reingreso **no** actúa como trigger de intensificación hacia insulina.

---

### Interpretación Crítica

Los resultados son contraintuitivos a primera vista — y hay razones para eso.

**"La insulina tiene el mayor riesgo de reingreso (15.9%)"**  
No significa que la insulina sea peor. Es confusión por indicación (*confounding by indication*): la insulina se prescribe precisamente a los pacientes más graves y con mayor comorbilidad. Comparar tasas brutas de reingreso entre grupos farmacológicos sin ajustar por severidad es metodológicamente incorrecto. Para establecer cualquier relación causal haría falta al menos un propensity score matching o un modelo ajustado por case-mix.

**"SIN_TTO protege más (6.67 hospitalizaciones antes de reingresar)"**  
Los pacientes sin antidiabético activo no necesariamente son diabéticos "mejor controlados". En muchos casos se hospitalizan por diagnóstico primario distinto (cirugía cardiovascular, fractura, etc.) donde la diabetes es comorbilidad secundaria y el umbral de reingreso temprano responde a otras lógicas clínicas. El modelo no distingue esto.

**"La insulina domina el steady state (50.1%)"**  
El steady state de Markov refleja la *frecuencia de aparición en el pool de transiciones*, no la prevalencia real de prescripción en la población. Los pacientes insulinodependientes son los que más se hospitalizan (revolving door), por lo que están sobrerepresentados en los pares de transición. El steady state es un indicador de intensidad de uso hospitalario, no de cuota de mercado en sentido estricto.

**"El reingreso no dispara el switch a insulina"**  
Esto sí es un hallazgo con valor clínico real. En un sistema bien calibrado, un reingreso en <30 días debería activar una revisión de la terapia. El hecho de que el patrón post-reingreso sea prácticamente igual al basal sugiere inercia prescriptora: se da el alta sin intensificar, y el paciente vuelve. Desde una perspectiva de health economics, ahí es donde un programa de stewardship o una intervención farmacéutica tiene más margen de impacto.

**Limitación del modelo Markov en medicina**  
La propiedad de Markov (memorylessness — el futuro solo depende del estado presente) es una simplificación fuerte. En la realidad clínica, la historia del paciente importa: cuántas veces ha sido hospitalizado, cuánto tiempo lleva en insulina, qué otras comorbilidades tiene. Un modelo de estados ocultos (HMM) o un modelo multiestado de supervivencia capturarían mejor esa complejidad.

---

## Cómo Ejecutar

```bash
# En Google Colab o localmente con Python 3.8+

# 1. Instalar dependencias
pip install pandas numpy matplotlib seaborn scikit-learn scipy networkx

# 2. Subir diabetic_data.csv al entorno

# 3. Ejecutar el EDA

# 4. Ejecutar la cadena de Markov
python markov_diabetes.py
```

---

## Notas

- El dataset usa `?` como valor nulo — se trata en la carga con `na_values='?'`
- `encounter_id` se usa como proxy de orden temporal (no hay timestamp real)
- Las transiciones de Markov se construyen sobre los 16.773 pacientes con más de un encuentro
- La cadena de Markov es de primer orden — no captura dependencias temporales largas

---

*Análisis realizado con fines académicos y exploratorios.*  
*Datos: Virginia Commonwealth University / Cerner Corporation (de-identificados).*
