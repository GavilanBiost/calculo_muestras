# Cálculo y Selección de Muestras - Estudio de Casos-Control Diabetes

## 📋 Descripción General

Este proyecto contiene el análisis completo para la selección de muestras en un **estudio de casos-control con diabéticos incidentes**. El estudio incluye:

- **600 casos**: Personas que desarrollaron diabetes durante el seguimiento
- **600 controles**: Personas sin diabetes, emparejadas según variables demográficas y clínicas

El propósito es identificar y organizar los participantes seleccionados, verificar disponibilidad de muestras biológicas y preparar la información para el biobanco.

---

## 🎯 Objetivo Principal

Realizar un emparejamiento **1:1 (case-control)** entre participantes con diabetes incidente y no diabéticos, ajustando por:
- Edad
- Sexo
- IMC (Índice de Masa Corporal)
- Glucosa basal
- Grupo de intervención
- Triglicéridos (añadido tras revisión el 14/10/2024)

---

## 📊 Estructura del Análisis

### 1. **Preparación de Bases de Datos**

El análisis comienza importando datos de múltiples fuentes:

- **BBDD**: Base de datos general con características basales (visita basal)
- **diab**: Base de datos con eventos de diabetes (ev_diab_2023-11-14)
- **Biobanc**: Registro de participantes con muestras biológicas disponibles
- **BBDD_2024_01_18**: Base de datos actualizada con información completa

**Librerías utilizadas:**
```r
library(haven)      # Para importar archivos .dta
library(readxl)     # Para leer Excel
library(dplyr)      # Para manipulación de datos
library(MatchIt)    # Para emparejamiento
```

### 2. **Generación de BBDD de Selección**

#### Paso 2.1: Selección de participantes sin diabetes previa
```r
BBDD_selec = subset(BBDD, diab_prev_s1 == 0)  # Excluir diabéticos previos
BBDD_selec = BBDD_selec[,c("paciente", "sexo_s1", "grupo_int_v00", 
                            "imc_v00", "edad_s1", "glucosa_v00", "trigli_v00")]
```

#### Paso 2.2: Integración de incidencia de diabetes
Se fusionan los datos basales con los eventos de diabetes:
```r
BBDD_selec_inci = merge(BBDD_selec, diab_selec, by = "paciente")
BBDD_selec_inci$diabetes[is.na(BBDD_selec_inci$diabetes)] = 0  # Imputar NAs
```

#### Paso 2.3: Filtrado por disponibilidad de muestras
Se mantienen solo los participantes con muestras en el biobanco:
- Se extraen IDs de pacientes del archivo biobanco
- Se eliminan duplicados
- Se mantienen 1,171 participantes (eliminados 29 con NAs)

### 3. **Emparejamiento (Matching)**

Se utiliza el método **nearest neighbor matching** (emparejamiento al vecino más cercano) con la librería `MatchIt`:

```r
set.seed(13)  # Para reproducibilidad
matched_data = matchit(
  diabetes ~ edad_s1 + sexo_s1 + glucosa_v00 + imc_v00 + grupo_int_v00 + trigli_v00,
  data = All_limpio,
  method = "nearest",
  ratio = 1  # 1 control por cada caso
)
```

**Características del matching:**
- Ratio 1:1 (un control por cada caso)
- Seed fija (13) para reproducibilidad
- Covariables ajustadas según las características basales de la población

### 4. **Selección Final de la Muestra**

```r
matched_sample = matched_df %>%
  group_by(diabetes) %>%
  sample_n(size = 600)  # 600 casos + 600 controles = 1,200 total
```

### 5. **Análisis Descriptivo**

Se generan dos tablas de características basales:

#### **Tabla 5.1: Variables Continuas**
Se calcula para cada variable:
- Media por grupo de diabetes
- Mediana por grupo
- Desviación estándar
- p-valor (prueba t de Student)

Variables incluidas:
- IMC
- Edad
- Score p17
- HDL, LDL (calculado)
- Glucosa basal
- Triglicéridos
- Energía total
- Consumo de alcohol

#### **Tabla 5.2: Variables Categóricas**
Se calcula para cada variable:
- Frecuencias (n) por grupo
- Porcentajes
- p-valor (prueba chi-cuadrado)

Variables incluidas:
- **Sexo**: Hombre/Mujer
- **Educación**: Título superior, Diplomatura, Secundaria, Primaria, Analfabeto
- **Grupo de intervención**: Control/Intervención

**Salida:**
- `20241011_caracteristicas_basales_cont.xlsx`: Tabla de variables continuas
- `20241011_caracteristicas_basales_cat.xlsx`: Tabla de variables categóricas

### 6. **Preparación de Datos para Biobanco**

#### Paso 6.1: Generación de IDs únicos
Se crea un nuevo formato de ID para las muestras:
```
Formato: P2/XXXXX/00
Ejemplo: P2/01234/00
```

#### Paso 6.2: Aleatorización en cajas
Se distribuyen aleatoriamente los 1,200 participantes en grupos de 96 por caja (12.5 cajas):

```r
set.seed(123)
grupos = data.frame(
  patient_ID = sample(unique(BBDD_id$patient_ID)),
  caja = rep(1:ceiling(length(unique(BBDD_id$patient_ID)) / 96), each = 96)
)
```

#### Paso 6.3: Asignación de pocillos (wells)
Se asignan posiciones en el formato estándar de placa (A-H × 1-12):

```r
aleatorizacion$"Well ID" = rep(rep(LETTERS[1:8], times = 12) %>%
  paste0(rep(1:12, each = 8)), 9)[1:1200]
```

#### Paso 6.4: Información de muestra
Se completan campos requeridos:
- **Plate ID**: ID de la caja/placa
- **Unique Sample ID**: Identificador único del participante
- **Well ID**: Posición en la placa
- **Sample volume**: 100 µL
- **Sample type**: EDTA plasma
- **Additional information**: Campo adicional (NA por defecto)

**Salida:** `20241024_SelecSamples.xlsx`

### 7. **Corrección de Identificadores (Biobanco)**

Se procesa el archivo de biobanco para corregir formatos de IDs:

```r
revisar <- read_excel("BIOR2300034 per revisar.xlsx")

# Extracción de patrón de 4 dígitos
matches <- corregir %>%
  regmatches(gregexpr("\\d{4}", .)) %>%
  unlist() %>%
  as.character() %>%
  data.frame(IDs = ., stringsAsFactors = FALSE) %>%
  mutate(patient_ID = paste0("P2/0", .[,1], "/00"))
```

**Salida:** `20241111_BIOR2300034_BBDD.xlsx`

---

## 📁 Archivos de Entrada Requeridos

| Archivo | Descripción |
|---------|------------|
| `BBDD.dta` | Base de datos general con características basales |
| `ev_diab_2023-11-14.dta` | Base de datos con eventos de diabetes |
| `Biobanc.xlsx` | Registro de participantes con muestras disponibles |
| `BBDD_2024_01_18.dta` | Base de datos actualizada (ruta: G:/trabajo/ARTICULOS/BBDD/BBDD/) |

---

## 📤 Archivos de Salida Generados

| Archivo | Contenido |
|---------|----------|
| `20241011_caracteristicas_basales_cont.xlsx` | Tabla descriptiva de variables continuas |
| `20241011_caracteristicas_basales_cat.xlsx` | Tabla descriptiva de variables categóricas |
| `20241024_SelecSamples.xlsx` | Selección final con IDs para biobanco |
| `20241111_BIOR2300034_BBDD.xlsx` | Archivo corregido del biobanco |
| `20241014_matched_sample.xlsx` | Muestra emparejada (opcional) |
| `estadisticos_seleccion.xlsx` | Estadísticos de selección (opcional) |
| `20241021_BBDD_1200_APOCIII_diab_matched_ids.csv` | IDs emparejados en formato CSV (opcional) |

---

## 🔧 Variables Clave Utilizadas

### Variables Demográficas
- `paciente`: Identificador único del participante
- `sexo_s1`: Sexo (Hombre/Mujer)
- `edad_s1`: Edad en visita basal
- `escola_v00`: Nivel educativo

### Variables Clínicas
- `imc_v00`: Índice de Masa Corporal (kg/m²)
- `glucosa_v00`: Glucosa basal (mg/dL)
- `trigli_v00`: Triglicéridos (mg/dL)
- `hdl_v00`: HDL colesterol (mg/dL)
- `ldl_calc_v00`: LDL colesterol calculado (mg/dL)
- `p17_total_v00`: Score P17
- `energiat_v00`: Energía total (kcal)
- `alcoholg_v00`: Consumo de alcohol (g)

### Variables de Clasificación
- `diabetes`: Indicador de incidencia de diabetes (0/1)
- `diab_prev_s1`: Diabetes previa (0/1)
- `grupo_int_v00`: Grupo de intervención (Control/Intervención)
- `fecha_ev`: Fecha del evento

---

## ⚠️ Notas Importantes

1. **Reproducibilidad**: Se utilizan seeds fijas (`seed=13` para matching y `seed=123` para aleatorización)

2. **Cambios realizados**:
   - *14/10/2024*: Se añadieron triglicéridos como variable de emparejamiento tras consulta con el IP

3. **Advertencia sobre BBDD_2024_01_18**: Esta base de datos fue generada para análisis de COVID y puede contener inconsistencias en vistas posteriores a la visita 3

4. **Pérdida de datos**: Se eliminan 29 participantes durante la limpieza de NAs

5. **Disponibilidad de muestras**: La selección final está limitada a participantes con muestras en el biobanco

---

## 🚀 Cómo Usar

1. **Preparar datos de entrada** en la ruta especificada (G:/trabajo/...)
2. **Ejecutar el notebook** `Sntx.Rmd` en RStudio
3. **Revisar outputs** generados (archivos xlsx)
4. **Usar `20241024_SelecSamples.xlsx`** para coordinación con biobanco

---

## 👤 Autor

**JFGG** (2024-09-02)

---

## 📝 Última Actualización

10 de mayo de 2026

---

---

# Sample Selection and Calculation - Incident Diabetes Case-Control Study

## 📋 General Description

This project contains the complete analysis for sample selection in a **case-control study with incident diabetics**. The study includes:

- **600 cases**: People who developed diabetes during follow-up
- **600 controls**: People without diabetes, matched according to demographic and clinical variables

The purpose is to identify and organize selected participants, verify availability of biological samples, and prepare information for the biobank.

---

## 🎯 Main Objective

Perform a **1:1 (case-control) matching** between participants with incident diabetes and non-diabetics, adjusting for:
- Age
- Sex
- BMI (Body Mass Index)
- Baseline glucose
- Intervention group
- Triglycerides (added after review on 14/10/2024)

---

## 📊 Analysis Structure

### 1. **Database Preparation**

The analysis begins by importing data from multiple sources:

- **BBDD**: General database with baseline characteristics (baseline visit)
- **diab**: Database with diabetes events (ev_diab_2023-11-14)
- **Biobanc**: Registry of participants with available biological samples
- **BBDD_2024_01_18**: Updated database with complete information

**Libraries used:**
```r
library(haven)      # To import .dta files
library(readxl)     # To read Excel
library(dplyr)      # For data manipulation
library(MatchIt)    # For matching
```

### 2. **Generation of Selection Database**

#### Step 2.1: Selection of participants without prior diabetes
```r
BBDD_selec = subset(BBDD, diab_prev_s1 == 0)  # Exclude previous diabetics
BBDD_selec = BBDD_selec[,c("paciente", "sexo_s1", "grupo_int_v00", 
                            "imc_v00", "edad_s1", "glucosa_v00", "trigli_v00")]
```

#### Step 2.2: Integration of diabetes incidence
Baseline data is merged with diabetes events:
```r
BBDD_selec_inci = merge(BBDD_selec, diab_selec, by = "paciente")
BBDD_selec_inci$diabetes[is.na(BBDD_selec_inci$diabetes)] = 0  # Impute NAs
```

#### Step 2.3: Filtering by sample availability
Only participants with samples in the biobank are retained:
- Extract patient IDs from biobank file
- Remove duplicates
- Retain 1,171 participants (29 with NAs removed)

### 3. **Matching**

The **nearest neighbor matching** method is used with the `MatchIt` library:

```r
set.seed(13)  # For reproducibility
matched_data = matchit(
  diabetes ~ edad_s1 + sexo_s1 + glucosa_v00 + imc_v00 + grupo_int_v00 + trigli_v00,
  data = All_limpio,
  method = "nearest",
  ratio = 1  # 1 control per case
)
```

**Matching characteristics:**
- 1:1 ratio (one control per case)
- Fixed seed (13) for reproducibility
- Covariates adjusted according to baseline population characteristics

### 4. **Final Sample Selection**

```r
matched_sample = matched_df %>%
  group_by(diabetes) %>%
  sample_n(size = 600)  # 600 cases + 600 controls = 1,200 total
```

### 5. **Descriptive Analysis**

Two baseline characteristics tables are generated:

#### **Table 5.1: Continuous Variables**
For each variable, the following is calculated:
- Mean by diabetes group
- Median by group
- Standard deviation
- p-value (Student's t-test)

Variables included:
- BMI
- Age
- p17 Score
- HDL, LDL (calculated)
- Baseline glucose
- Triglycerides
- Total energy
- Alcohol consumption

#### **Table 5.2: Categorical Variables**
For each variable, the following is calculated:
- Frequencies (n) by group
- Percentages
- p-value (chi-square test)

Variables included:
- **Sex**: Male/Female
- **Education**: Higher degree, University diploma, Secondary, Primary, Illiterate
- **Intervention group**: Control/Intervention

**Output:**
- `20241011_caracteristicas_basales_cont.xlsx`: Table of continuous variables
- `20241011_caracteristicas_basales_cat.xlsx`: Table of categorical variables

### 6. **Data Preparation for Biobank**

#### Step 6.1: Generation of unique IDs
A new ID format is created for samples:
```
Format: P2/XXXXX/00
Example: P2/01234/00
```

#### Step 6.2: Randomization in boxes
The 1,200 participants are randomly distributed in groups of 96 per box (12.5 boxes):

```r
set.seed(123)
grupos = data.frame(
  patient_ID = sample(unique(BBDD_id$patient_ID)),
  caja = rep(1:ceiling(length(unique(BBDD_id$patient_ID)) / 96), each = 96)
)
```

#### Step 6.3: Well assignment
Positions are assigned in standard plate format (A-H × 1-12):

```r
aleatorizacion$"Well ID" = rep(rep(LETTERS[1:8], times = 12) %>%
  paste0(rep(1:12, each = 8)), 9)[1:1200]
```

#### Step 6.4: Sample information
Required fields are completed:
- **Plate ID**: Box/plate ID
- **Unique Sample ID**: Participant's unique identifier
- **Well ID**: Position on the plate
- **Sample volume**: 100 µL
- **Sample type**: EDTA plasma
- **Additional information**: Additional field (NA by default)

**Output:** `20241024_SelecSamples.xlsx`

### 7. **Identifier Correction (Biobank)**

The biobank file is processed to correct ID formats:

```r
revisar <- read_excel("BIOR2300034 per revisar.xlsx")

# Extraction of 4-digit pattern
matches <- corregir %>%
  regmatches(gregexpr("\\d{4}", .)) %>%
  unlist() %>%
  as.character() %>%
  data.frame(IDs = ., stringsAsFactors = FALSE) %>%
  mutate(patient_ID = paste0("P2/0", .[,1], "/00"))
```

**Output:** `20241111_BIOR2300034_BBDD.xlsx`

---

## 📁 Required Input Files

| File | Description |
|------|-------------|
| `BBDD.dta` | General database with baseline characteristics |
| `ev_diab_2023-11-14.dta` | Database with diabetes events |
| `Biobanc.xlsx` | Registry of participants with available samples |
| `BBDD_2024_01_18.dta` | Updated database (path: G:/trabajo/ARTICULOS/BBDD/BBDD/) |

---

## 📤 Generated Output Files

| File | Content |
|------|---------|
| `20241011_caracteristicas_basales_cont.xlsx` | Descriptive table of continuous variables |
| `20241011_caracteristicas_basales_cat.xlsx` | Descriptive table of categorical variables |
| `20241024_SelecSamples.xlsx` | Final selection with IDs for biobank |
| `20241111_BIOR2300034_BBDD.xlsx` | Corrected biobank file |
| `20241014_matched_sample.xlsx` | Matched sample (optional) |
| `estadisticos_seleccion.xlsx` | Selection statistics (optional) |
| `20241021_BBDD_1200_APOCIII_diab_matched_ids.csv` | Matched IDs in CSV format (optional) |

---

## 🔧 Key Variables Used

### Demographic Variables
- `paciente`: Participant's unique identifier
- `sexo_s1`: Sex (Male/Female)
- `edad_s1`: Age at baseline visit
- `escola_v00`: Educational level

### Clinical Variables
- `imc_v00`: Body Mass Index (kg/m²)
- `glucosa_v00`: Baseline glucose (mg/dL)
- `trigli_v00`: Triglycerides (mg/dL)
- `hdl_v00`: HDL cholesterol (mg/dL)
- `ldl_calc_v00`: Calculated LDL cholesterol (mg/dL)
- `p17_total_v00`: P17 Score
- `energiat_v00`: Total energy (kcal)
- `alcoholg_v00`: Alcohol consumption (g)

### Classification Variables
- `diabetes`: Diabetes incidence indicator (0/1)
- `diab_prev_s1`: Prior diabetes (0/1)
- `grupo_int_v00`: Intervention group (Control/Intervention)
- `fecha_ev`: Event date

---

## ⚠️ Important Notes

1. **Reproducibility**: Fixed seeds are used (`seed=13` for matching and `seed=123` for randomization)

2. **Changes made**:
   - *14/10/2024*: Triglycerides were added as a matching variable after consultation with the IP

3. **Warning about BBDD_2024_01_18**: This database was generated for COVID analysis and may contain inconsistencies in visits after visit 3

4. **Data loss**: 29 participants are removed during NA cleaning

5. **Sample availability**: Final selection is limited to participants with samples in the biobank

---

## 🚀 How to Use

1. **Prepare input data** in the specified path (G:/trabajo/...)
2. **Run the notebook** `Sntx.Rmd` in RStudio
3. **Review generated outputs** (xlsx files)
4. **Use `20241024_SelecSamples.xlsx`** for biobank coordination

---

## 👤 Author

**JFGG** (2024-09-02)

---

## 📝 Last Update

May 10, 2026