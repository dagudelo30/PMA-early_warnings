# 🌧️ PMA Early Warnings — Recomendación Agroclimática Estacional

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/dagudelo30/PMA-early_warnings/blob/main/notebooks/resampling_PMA.ipynb)

**Proyecto:** Plataforma de Monitoreo Agroclimático (PMA)  
**Departamentos cubiertos:** Caquetá · Amazonas · Putumayo  
**Fuentes de datos:** IDEAM (pronóstico estacional) + CHIRPS / AgERA5 (histórico)

---

## 📋 Tabla de contenidos

- [¿Qué hace este flujo de trabajo?](#qué-hace-este-flujo-de-trabajo)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Cómo ejecutar en Google Colab (recomendado)](#cómo-ejecutar-en-google-colab)
- [Cómo ejecutar en local](#cómo-ejecutar-en-local)
- [Parámetros configurables](#parámetros-configurables)
- [Metodología detallada](#metodología-detallada)
- [Control de calidad a los pronósticos IDEAM](#control-de-calidad)
- [Archivos de salida](#archivos-de-salida)
- [Consumo externo de los GeoTIFFs](#consumo-externo-de-los-geotiffs)
- [Dependencias](#dependencias)
- [Referencias](#referencias)

---

## ¿Qué hace este flujo de trabajo?

Para un mes objetivo (p. ej. abril 2026), el notebook produce automáticamente:

| Producto | Formato | Descripción |
|---|---|---|
| Cambio porcentual de precipitación | `.tif` + `.png` | Anomalía del pronóstico respecto a la climatología 1982–2025, por píxel |
| Categoría SPEI modal | `.tif` + `.png` | Recomendación agroclimática (Muy seco / Normal / Muy húmedo) mediante análogos |
| Mapa de píxeles QC | `.png` | Diagnóstico de píxeles corregidos en el control de calidad |

Los productos se generan para cada uno de los departamentos configurados: **Caquetá, Amazonas y Putumayo**.

---

## Estructura del repositorio

```
PMA-early_warnings/
├── notebooks/
│   └── resampling_PMA.ipynb      ← Notebook principal
├── data/
│   └── precip_spei_mensual.nc    ← Datos históricos CHIRPS/AgERA5 (1982–2025)
├── outputs/                       ← GeoTIFFs y mapas generados (se crea automáticamente)
└── README.md
```

---

## Cómo ejecutar en Google Colab

La forma más sencilla. No requiere instalación local.

1. Haz clic en el botón **Open in Colab** al inicio de este README.
2. En la primera celda del notebook, ejecuta la instalación de dependencias (solo la primera vez).
3. Ve a la sección **"1. ⚙️ Parámetros"** y ajusta `MONTH`, `YEAR` y demás parámetros según el período que quieras procesar.
4. Ve al menú **Entorno de ejecución → Ejecutar todo** (`Ctrl+F9`).

> **Nota sobre los datos históricos en Colab:** El notebook descarga automáticamente `precip_spei_mensual.nc` desde este repositorio de GitHub. No necesitas subir ningún archivo manualmente.

---

## Cómo ejecutar en local

### 1. Clonar el repositorio

```bash
git clone https://github.com/dagudelo30/PMA-early_warnings.git
cd PMA-early_warnings
```

### 2. Crear entorno y dependencias

Con **conda** (recomendado):

```bash
conda create -n pma python=3.10
conda activate pma
pip install -r requirements.txt
```

Con **pip**:

```bash
pip install -r requirements.txt
```

### 3. Ejecutar el notebook

```bash
cd notebooks
jupyter notebook resampling_PMA.ipynb
```

---

## Parámetros configurables

Todos los parámetros están concentrados en la **celda "1. ⚙️ Parámetros"** del notebook. Al modificar únicamente esta celda y re-ejecutar el notebook completo, se obtienen los productos para el período y configuración deseados.

| Parámetro | Valor por defecto | Descripción |
|---|---|---|
| `MONTH` | `4` | Mes objetivo (1–12) |
| `YEAR` | `2026` | Año objetivo |
| `IDEAM_BASE_URL` | URL oficial IDEAM | Base de la URL del archivo NetCDF del pronóstico |
| `DEPARTMENTS` | `["Caquetá", "Amazonas", "Putumayo"]` | Departamentos a procesar |
| `QC_UPPER_FACTOR` | `2.0` | Si pred > factor × climatología → se trunca |
| `QC_CAP_FACTOR` | `1.1` | El pronóstico truncado se lleva a cap × climatología |
| `K` | `3` | Número de años análogos para el cálculo de la moda SPEI |
| `OUTPUT_DIR` | `"../outputs"` | Directorio donde se guardan los resultados |

### Ejemplo: procesar junio 2025

```python
MONTH = 6
YEAR  = 2025
K     = 5   # usar los 5 análogos más cercanos en lugar de 3
```

---

## Metodología detallada

### Contexto

La predicción climática estacional es un insumo esencial en las Mesas Técnicas Agroclimáticas de Colombia. El **IDEAM** pone a disposición pública pronósticos de precipitación mensual con cobertura nacional a través de su repositorio:  
[https://bart.ideam.gov.co/wrfideam/new_modelo/CPT/](https://bart.ideam.gov.co/wrfideam/new_modelo/CPT/)

Estos pronósticos integran ensambles multi-modelo con postproceso estadístico (CPT), coherente con la evidencia de que los ensambles multi-modelo mejoran la habilidad frente a modelos individuales (Barnston et al., 2003; DelSole et al., 2014).

### Climatología de referencia

Se construyó una climatología mensual con el archivo histórico `precip_spei_mensual.nc`, que cubre el período **febrero 1982 – febrero 2025** con datos de **CHIRPS** (precipitación) y **AgERA5** (evapotranspiración, para el SPEI). Esta climatología sirve como línea base para expresar la señal pronosticada en términos relativos, reduciendo la dependencia de magnitudes absolutas y facilitando la comparación regional.

### Cambio porcentual de precipitación

Con el pronóstico corregido por QC (P_f) y la climatología mensual (P_clim), se estima para cada píxel:

$$\Delta(\%) = 100 \times \frac{P_f - P_{\text{clim}}}{P_{\text{clim}}}$$

Este campo espacial sintetiza la señal prevista en términos comparables entre regiones.

### Método de años análogos

Para traducir el cambio porcentual pronosticado en una recomendación operativa:

1. **Histórico de anomalías:** Se calcula $\Delta_{hist}(t)$ para cada tiempo $t$ del archivo histórico con el mismo esquema (misma climatología de referencia), garantizando comparabilidad directa.

2. **Distancia por píxel:** Para cada píxel se cuantifica la similitud entre el pronóstico ($\Delta_f$) y cada año histórico:

$$d(t) = |\Delta_{hist}(t) - \Delta_f|$$

3. **Top-K análogos:** Se seleccionan los $K$ tiempos históricos de menor distancia. La búsqueda se implementa de forma eficiente con `np.argpartition` (O(T) por píxel, sin ordenar todo el vector).

4. **Moda del SPEI:** La recomendación agroclimática es la moda de las categorías SPEI entre los K análogos:

$$\text{SPEI}_f = \text{moda}\left\{\text{SPEI}(t_1), \text{SPEI}(t_2), \ldots, \text{SPEI}(t_K)\right\}$$

### Categorías SPEI

El SPEI se representa en **tres categorías operativas**, resultado del proceso de co-diseño con usuarios:

| Valor | Categoría | Descripción |
|---|---|---|
| `1` | 🔴 Verano (Muy seco) | Déficit hídrico significativo |
| `2` | 🟡 Normal | Condiciones dentro de lo esperado |
| `3` | 🔵 Invierno (Muy húmedo) | Exceso hídrico significativo |

> La simplificación a 3 categorías (desde 5 originales) fue resultado de talleres participativos con productores y técnicos, quienes identificaron mayor claridad interpretativa con esta clasificación.

---

## Control de calidad

Antes del cálculo del cambio porcentual, se aplica un **control de calidad (QC)** a los campos pronosticados por el IDEAM, debido a la presencia ocasional de valores físicamente inconsistentes o extremos.

### Reglas QC (por píxel)

| Regla | Condición | Acción |
|---|---|---|
| **R1 – Negativos** | pred < 0 | Reemplazar por 0 mm |
| **R2 – Extremos** | pred > `QC_UPPER_FACTOR` × clim | Truncar a `QC_CAP_FACTOR` × clim |

**Valores por defecto:**
- `QC_UPPER_FACTOR = 2.0`: pronósticos que superen el doble de la climatología son sospechosos.
- `QC_CAP_FACTOR = 1.1`: se permite hasta un 10% sobre la climatología como máximo post-QC.

### Justificación

Estas reglas buscan:
- **Preservar la señal espacial** del pronóstico, evitando que valores atípicos dominen el cálculo del porcentaje de cambio.
- **No imponer sesgo sistemático**: la regla solo actúa en los píxeles donde el valor supera el umbral, dejando intacto el resto.
- **Transparencia**: el notebook genera un mapa de los píxeles modificados, permitiendo auditar el alcance del QC en cada corrida.

### Diagnóstico post-QC

El notebook reporta automáticamente:
```
🔍 QC completado:
   Umbral superior : 2.0× climatología → cap 1.1× climatología
   Píxeles modificados: 47 / 12450 (0.38%)
```

Y genera un mapa PNG con la distribución espacial de los píxeles corregidos.

---

## Archivos de salida

Para cada departamento y período objetivo se generan:

```
outputs/
├── cambio_pct_04_2026_caqueta.tif       ← Cambio % precipitación (float32, EPSG:4326, LZW)
├── cambio_pct_04_2026_amazonas.tif
├── cambio_pct_04_2026_putumayo.tif
├── spei_moda_04_2026_caqueta.tif        ← Categoría SPEI modal (int16, nodata=-9999)
├── spei_moda_04_2026_amazonas.tif
├── spei_moda_04_2026_putumayo.tif
├── mapa_cambio_pct_04_2026_caqueta.png  ← Mapa listo para publicar (400 dpi)
├── mapa_cambio_pct_04_2026_amazonas.png
├── mapa_cambio_pct_04_2026_putumayo.png
├── mapa_spei_04_2026_caqueta.png
├── mapa_spei_04_2026_amazonas.png
├── mapa_spei_04_2026_putumayo.png
└── qc_pixels_04_2026.png               ← Diagnóstico QC
```

### Especificaciones técnicas de los GeoTIFFs

| Atributo | Cambio porcentual | SPEI modal |
|---|---|---|
| Tipo de dato | `float32` | `int16` |
| CRS | EPSG:4326 (WGS84) | EPSG:4326 (WGS84) |
| Compresión | LZW | LZW |
| NoData | `NaN` | `-9999` |
| Resolución | ~0.05° (~5 km) | ~0.05° (~5 km) |

---

## Consumo externo de los GeoTIFFs

Los archivos `.tif` son estándar **Cloud Optimized GeoTIFF** y pueden consumirse desde múltiples plataformas:

### Python (rioxarray)
```python
import rioxarray
da = rioxarray.open_rasterio("cambio_pct_04_2026_amazonas.tif", masked=True)
da.plot()
```

### R (terra)
```r
library(terra)
r <- rast("cambio_pct_04_2026_amazonas.tif")
plot(r)
```

### QGIS
Arrastra el archivo `.tif` directamente al lienzo de QGIS. El CRS se detecta automáticamente.

### Google Earth Engine (subida de assets)
```python
# Subir como asset de GEE con la herramienta ee upload
earthengine upload image --asset_id=users/tu_usuario/cambio_pct_04_2026_amazonas \
    cambio_pct_04_2026_amazonas.tif
```

---

## Dependencias

```
xarray>=2024.1
rioxarray>=0.15
rasterio>=1.3
geopandas>=0.14
netCDF4>=1.6
requests>=2.31
numpy>=1.26
matplotlib>=3.8
```

Instalar con:
```bash
pip install xarray rioxarray rasterio geopandas netCDF4 requests numpy matplotlib
```

---

## Referencias

- Barnston, A. G., et al. (2003). Multimodel ensembling in seasonal climate forecasting at IRI. *Bulletin of the American Meteorological Society*.
- DelSole, T., & Shukla, J. (2014). Artificial skill due to predictor screening. *Journal of Climate*.
- Poveda, G., & Mesa, O. J. (2005). Climate variability over Colombia. *International Journal of Climatology*.
- Vicente-Serrano, S. M., et al. (2010). A multiscalar drought index: The SPEI. *Journal of Climate*.
- IDEAM. Repositorio de predicción estacional: [https://bart.ideam.gov.co/wrfideam/new_modelo/CPT/](https://bart.ideam.gov.co/wrfideam/new_modelo/CPT/)

---

## Licencia

Este flujo de trabajo es de uso libre para instituciones y personas interesadas en servicios agroclimáticos. Si lo usas o adaptas, por favor cita el repositorio y las fuentes de datos originales (IDEAM, CHIRPS, AgERA5).
