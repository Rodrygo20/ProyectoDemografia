# Proyecto Demografía de Sinaloa

Este repositorio documenta el proceso completo para analizar la dinámica demográfica de Sinaloa mediante estructura poblacional, tablas de vida, causa eliminada por homicidios y medidas sintéticas de fecundidad. La base del proyecto usa datos de CONAPO, INEGI y Naciones Unidas, integrados en un flujo de trabajo en R y Quarto [file:251].

## Objetivo

El proyecto busca describir y comparar la evolución reciente de la población de Sinaloa mediante cuatro componentes principales: pirámides poblacionales para 2010, 2019 y 2021; construcción de tablas de vida abreviadas por sexo; estimación de tablas de vida de causa eliminada por homicidios; y cálculo de tasas específicas y medidas sintéticas de fecundidad [file:251].

## Estructura sugerida del repositorio

```text
ProyectoDemografia/
├── data/
│   ├── raw/
│   └── processed/
├── graficas/
├── output/
├── reportes/
├── scripts/
├── Proyecto.qmd
├── Proyecto.Rproj
└── README.md
```

Una organización así permite separar archivos originales, resultados procesados, gráficas y documentos finales, evitando que todos los archivos queden sueltos en la raíz del repositorio [file:251].

## Fuentes de datos

### 1. Población

La información poblacional proviene de las **Proyecciones de la Población de México y de las Entidades Federativas, 1950-2070** de CONAPO. En el código se filtra la entidad de Sinaloa y se trabaja con edad desplegada, sexo y año [file:251].

Variables clave:
- `ENTIDAD`
- `ANIO`
- `SEXO`
- `EDAD`
- `POBLACION`

Archivo usado en el proyecto:
- `Proy_pob.csv` [file:251]

### 2. Mortalidad

Las defunciones provienen del apartado **Estadísticas de Defunciones Registradas** de INEGI. Se emplean defunciones ocurridas en Sinaloa, desagregadas por edad y sexo, y en el caso del ejercicio de causa eliminada también por causa de muerte [file:251].

Archivos usados o mencionados:
- bases de defunciones de Sinaloa
- archivo de homicidios o tabla causa eliminada
- tablas intermedias para mortalidad y esperanza de vida [file:251]

### 3. Fecundidad

Para fecundidad se usaron nacimientos por edad de la madre y población femenina en edades reproductivas. En la comparación internacional de tasas específicas se usó la base de Naciones Unidas por grupos quinquenales de edad [file:251].

Archivos usados o mencionados:
- `Nacimientos_2010.xlsx`
- `Nacimientos_2019.xlsx`
- `WPP2024_FERT_F02_FERTILITY_RATES_BY_5-YEAR_AGE_GROUPS_OF_MOTHER.xlsx` [file:251]

## Paquetes necesarios en R

El documento base carga los siguientes paquetes [file:251]:

```r
library(data.table)
library(dplyr)
library(tidyr)
library(readxl)
library(lubridate)
library(ggplot2)
library(knitr)
library(kableExtra)
```

Dependiendo de la versión final del proyecto, también pueden requerirse otros paquetes para exportación, tablas o limpieza adicional.

## Flujo general del proyecto

## 1. Preparación del entorno

Antes de iniciar, se limpia la sesión y se cargan los paquetes. Esto evita conflictos de objetos previos y asegura que todas las funciones necesarias estén disponibles [file:251].

```r
rm(list = ls())
```

## 2. Lectura de población

Se importa `Proy_pob.csv` con `fread()` y después se filtran los registros de Sinaloa según el año que se quiera analizar. En las pirámides se trabaja directamente con hombres y mujeres, y para el efecto espejo la población masculina se convierte a negativa [file:251].

Ejemplo:

```r
pob <- fread("Proy_pob.csv")
SINA <- pob[ENTIDAD == "Sinaloa" & ANIO == "2010", .(SEXO, EDAD, POBLACION)]
SINA[, POBLACION_GRAF := ifelse(SEXO == "Hombres", -POBLACION, POBLACION)]
```

## 3. Construcción de pirámides poblacionales

Se elaboran gráficas para 2010, 2019 y 2021 con `ggplot2`, usando barras horizontales y el efecto espejo entre hombres y mujeres. Luego se guardan con `ggsave()` en formato PNG [file:251].

Pasos:
- Filtrar Sinaloa y año.
- Crear `POBLACION_GRAF` con signo negativo para hombres.
- Construir la gráfica con `geom_bar()` y `coord_flip()`.
- Exportar la imagen.

Ejemplo de exportación:

```r
ggsave("piramides_poblacionales_sinaloa2010.png", width = 8, height = 5, dpi = 300)
```

## 4. Interpretación de estructura poblacional

El proyecto interpreta tres cambios estructurales: estrechamiento de la base, desplazamiento del centro hacia edades activas y crecimiento de la parte superior de la pirámide. Eso se relaciona con descenso de la fecundidad, bono demográfico y envejecimiento poblacional [file:251].

## 5. Población expuesta al riesgo

Para mortalidad y fecundidad no basta con usar una población al inicio del año; se requiere estimar la población a mitad de año. Para eso se aplica interpolación exponencial entre dos años consecutivos y se obtiene una aproximación de los años persona vividos (APV) [file:251].

La función usada es:

```r
expo <- function(K_0, K_T, t_0, t_T, t){
  dt <- decimal_date(as.Date(t_T)) - decimal_date(as.Date(t_0))
  r <- log(K_T / K_0) / dt
  h <- t - decimal_date(as.Date(t_0))
  K_h <- K_0 * exp(r * h)
  return(K_h)
}
```

Después se usa `get_apv()` para generar la población media de 2010 y 2019 [file:251].

## 6. Tasas centrales de mortalidad y tabla de vida

Con defunciones observadas y población expuesta al riesgo se calcula la tasa central de mortalidad: `m_x = D_x / N_x`. A partir de esa tasa se construyen las demás columnas de la tabla de vida abreviada: `q_x`, `p_x`, `l_x`, `d_x`, `L_x`, `T_x` y `e_x` [file:251].

En la redacción del documento se explica además el uso del parámetro `a_x`, con reglas de Coale-Demeny para 0 y 1 a 4 años, y distribución uniforme para el resto de edades [file:251].

## 7. Tabla de vida de causa eliminada por homicidios

Después de tener la tabla base, se construye un escenario contrafactual eliminando homicidios como causa de muerte. Para ello se usa la lógica de riesgos competitivos y se recalculan probabilidades de sobrevivencia, probabilidad de morir, años persona vividos y esperanza de vida sin esa causa [file:251].

El documento incluye fórmulas para:
- factor de reducción del riesgo
- nueva probabilidad de sobrevivir
- nueva probabilidad de morir
- ajuste de `a_x`
- nueva esperanza de vida
- ganancia potencial de vida [file:251]

Este bloque permite medir cuánto aumentaría la esperanza de vida si los homicidios desaparecieran del patrón de mortalidad observado [file:251].

## 8. Preparación del análisis de fecundidad

El análisis de fecundidad requiere tres insumos: nacimientos por edad de la madre, población femenina expuesta al riesgo y, para TNR, información de supervivencia femenina de la tabla de vida [file:251].

### 8.1 Lectura de nacimientos

Los nacimientos se leen desde archivos de Excel para 2010 y 2019. Se busca la fila de Sinaloa y se extraen los nacimientos por grupos quinquenales de edad de la madre [file:251].

La función principal es:

```r
read_births_row <- function(file, year_target){
  b <- read_excel(file, skip = 5, col_names = TRUE)
  names(b) <- trimws(names(b))
  fila <- b %>%
    filter(grepl("Sinaloa", as.character(.[[2]])) | grepl("^25$", as.character(.[[1]])))
  fila <- fila[1, ]
  tibble(
    year = year_target,
    age = c("15-19","20-24","25-29","30-34","35-39","40-44","45-49"),
    births = c(
      as.numeric(gsub(",", "", fila[["De 15 a 19 años"]])),
      as.numeric(gsub(",", "", fila[["De 20 a 24 años"]])),
      as.numeric(gsub(",", "", fila[["De 25 a 29 años"]])),
      as.numeric(gsub(",", "", fila[["De 30 a 34 años"]])),
      as.numeric(gsub(",", "", fila[["De 35 a 39 años"]])),
      as.numeric(gsub(",", "", fila[["De 40 a 44 años"]])),
      as.numeric(gsub(",", "", fila[["De 45 a 49 años"]]))
    )
  )
}
```

### 8.2 Agrupación de edades femeninas

La población femenina se agrupa en quinquenios de 15 a 49 años usando una función auxiliar [file:251].

```r
grupo_edad_fec <- function(x){
  x <- suppressWarnings(as.numeric(x))
  dplyr::case_when(
    is.na(x) ~ NA_character_,
    x >= 15 & x <= 19 ~ "15-19",
    x >= 20 & x <= 24 ~ "20-24",
    x >= 25 & x <= 29 ~ "25-29",
    x >= 30 & x <= 34 ~ "30-34",
    x >= 35 & x <= 39 ~ "35-39",
    x >= 40 & x <= 44 ~ "40-44",
    x >= 45 & x <= 49 ~ "45-49",
    TRUE ~ NA_character_
  )
}
```

### 8.3 Tabla intermedia de fecundidad

Se unen nacimientos y población femenina para calcular:
- `fx`: tasa específica de fecundidad.
- `fx5`: tasa quinquenal.
- `fx5_fem`: tasa ajustada a nacimientos femeninos [file:251].

## 9. Cálculo de TGF, TBR y TNR

Las medidas sintéticas se calculan así [file:251]:

- **TGF**: suma de las tasas específicas quinquenales.
- **TBR**: TGF ajustada por la proporción de nacimientos femeninos.
- **TNR**: TBR ajustada adicionalmente por la supervivencia femenina proveniente de la tabla de vida.

En el código, la constante usada para nacimientos femeninos es [file:251]:

```r
K <- 100 / (100 + 105)
```

Luego se resume por año:

```r
res_tfg_tbr <- fec %>%
  group_by(year) %>%
  summarise(
    TGF = 5 * sum(fx, na.rm = TRUE),
    TBR = 5 * sum(fx * K, na.rm = TRUE),
    .groups = "drop"
  )
```

Para la TNR se incorpora `lx` de la tabla de vida femenina [file:251].

## 10. Comparación gráfica de fecundidad

Los resultados se integran en una tabla resumen con TGF, TBR y TNR para 2010 y 2019, y luego se construyen gráficas de barras y líneas para compararlas visualmente [file:251]. También se realizaron gráficas de tasas específicas por edad para comparar Sinaloa con otros contextos como México e Italia usando la base de Naciones Unidas [file:251].

## 11. Redacción del documento en Quarto

El informe se arma en `Proyecto.qmd`. Ahí se integran:
- explicación de fuentes de datos
- metodología
- fórmulas
- bloques de código R
- interpretación de resultados
- gráficas y tablas [file:251]

Para las ecuaciones, si Quarto da errores de compilación en PDF, conviene usar una sola convención de escritura matemática y mantener todas las expresiones con subíndices y superíndices dentro de modo matemático [file:251].

## 12. Exportación y organización de archivos

Se recomienda guardar los productos en carpetas separadas:

- `graficas/` para PNG
- `data/raw/` para bases originales
- `data/processed/` para tablas limpias
- `reportes/` para PDF y HTML
- `scripts/` para código y QMD

Esto hace más fácil subir el proyecto a GitHub sin dejar archivos sueltos [file:251].

## Secuencia recomendada para reproducir el proyecto

1. Colocar todos los archivos fuente en la carpeta del proyecto.
2. Instalar y cargar los paquetes de R.
3. Leer `Proy_pob.csv` y generar las pirámides poblacionales.
4. Estimar la población a mitad de año con interpolación exponencial.
5. Integrar defunciones por edad y sexo.
6. Calcular tasas centrales de mortalidad y construir tablas de vida.
7. Generar el ejercicio de causa eliminada por homicidios.
8. Leer nacimientos por edad de la madre.
9. Agrupar población femenina de 15 a 49 años.
10. Calcular tasas específicas de fecundidad.
11. Obtener TGF, TBR y TNR.
12. Comparar resultados en tablas y gráficas.
13. Redactar y renderizar `Proyecto.qmd` a PDF o HTML [file:251].

## Problemas comunes

### Error de compilación en Quarto/LaTeX

Si aparece el mensaje `Missing $ inserted`, normalmente significa que hay subíndices o superíndices fuera de modo matemático, por ejemplo `_x` o `^i` sin delimitadores matemáticos. La solución es revisar que todas las fórmulas estén correctamente escritas con `$...$` o `$$...$$` en el documento [file:251].

### Archivos sueltos en GitHub

Si los archivos se suben todos en la raíz del repositorio, hay que reorganizarlos en carpetas y actualizar las rutas en el código. Por ejemplo, cambiar `ggsave("grafica.png")` por `ggsave("graficas/grafica.png")` [file:251].

### Coincidencia entre nacimientos y población

Las tasas de fecundidad requieren que los nacimientos por edad de la madre y la población femenina tengan exactamente la misma agrupación por edad. Si no coinciden los intervalos, los resultados serán incorrectos [file:251].

## Resultado esperado

Al final del proceso, el proyecto debe producir un informe en Quarto con texto, fórmulas, tablas y gráficas que describen la estructura poblacional de Sinaloa, su patrón de mortalidad, el efecto de eliminar homicidios sobre la esperanza de vida y la evolución reciente de la fecundidad [file:251].
