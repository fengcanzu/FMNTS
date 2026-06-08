# HANTS-FMSF: Harmonic Analysis of Time Series with Fusion-based Multi-Source Filling

A Python pipeline for reconstructing dense, cloud-free NDVI time series by fusing **Landsat** and **MODIS** (or AVHRR) imagery using harmonic analysis (HANTS) and a Frequency-domain Multi-Scale Fusion (FMSF)method.

---
 
## Overview

Remote sensing NDVI time series are often incomplete due to cloud cover, especially for high-resolution Landsat data. This pipeline addresses that problem through two complementary strategies:

- **HANTS** — Applied to pixels with sufficient valid Landsat observations (> 14 valid dates per year). Fits a 3rd-order harmonic model directly to the Landsat time series with iterative outlier filtering.
- **FMSF** — Applied to data-sparse pixels. Transfers harmonic structure from a coarse-resolution reference (MODIS/AVHRR) to Landsat by computing amplitude ratios and phase differences against a multi-year reference dataset.

The two results are blended spatially by a **data-quality weight map**, yielding a seamless, gap-free NDVI reconstruction at the Landsat spatial resolution.

---

## Pipeline Architecture

```
Raw Inputs (Landsat + MODIS/AVHRR TIFs)
        │
        ▼
┌─────────────────────────────┐
│  Step 1: Data Preparation   │
│  - Max-value compositing    │
│  - Zarr array storage       │
└────────────┬────────────────┘
             │
      ┌──────┴──────┐
      ▼             ▼
 Predicted year   Reference year(s)
 (ltpr, mdpr)     (ltre, mdre)
      │             │
      ▼             ▼
┌─────────────────────────────┐
│  Step 2: Harmonic Fitting   │
│  Ridge regression (HANTS)   │
│  Dask parallel processing   │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  Step 3: Significance Test  │
│  Energy ratio pruning of    │
│  2nd and 3rd harmonics      │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  Step 4: FMSF Fusion        │
│  Amplitude ratio transfer   │
│  Phase difference transfer  │
│  Weighted blend with HANTS  │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  Step 5: Reconstruction     │
│  Evaluate harmonics on      │
│  dense date grid (16-day)   │
└────────────┬────────────────┘
             │
             ▼
    Output GeoTIFF (rendvi_<year>.tif)
```

---

## Requirements

```
numpy
rasterio
zarr
numcodecs
dask
scikit-learn
pandas
```

Install with:
```bash
pip install numpy rasterio zarr numcodecs dask scikit-learn pandas
```

---

## Directory Structure

All datasets are expected under a single `base` directory:

```
base/
├── collection_mdpr/        # MODIS NDVI, predicted year      (files: YYYY_MM_DD.tif)
├── collection_ltpr/        # Landsat NDVI, predicted year    (files: *_YYYYMMDD.tif)
├── collection_mdre/        # MODIS NDVI, reference year(s)   (files: YYYY_MM_DD.tif)
├── collection_ltre/        # Landsat NDVI, reference year(s) (files: *_YYYYMMDD.tif)
└── [outputs created by pipeline]
    ├── ltpr_lists.zarr
    ├── mdpr_lists.zarr
    ├── ltre_lists.zarr
    ├── mdre_lists.zarr
    ├── coefImg_ltpr_.zarr
    ├── coefImg_mdpr_.zarr
    ├── coefImg_ltre_.zarr
    ├── coefImg_mdre_.zarr
    ├── coefImg_hants_.zarr
    ├── weight_.zarr
    └── FMNTS/
        └── rendvi_<year>.tif
```

---

## Configuration

Edit the parameter block at the top of the notebook:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `year` | Predicted (target) year | `2019` |
| `interval` | Output time step in days | `16` |
| `base` | Root directory for all data | `E:\data\HANTS\...` |
| `H_chunk` / `W_chunk` | Spatial chunk size for Zarr | `64` |
| `n_workers` | Dask worker count | `4` |
| `memory_limit_per_worker` | Memory per Dask worker | `'6GB'` |
| `alpha` | CV sensitivity for weight calculation | `2.0` |

The variable `freq` is set automatically: `'md'` (MODIS) for years ≥ 2000, `'av'` (AVHRR) for earlier years.

---

## Step-by-Step Execution

### Step 1 — Data Preparation (Cells 1–2)

Reads raw TIF files, applies NDVI validity masks, performs maximum-value compositing (per date for Landsat, 8-day window for reference sequences), and stores results as compressed Zarr arrays.

- Landsat: grouped by acquisition date → max composite → `ltpr_lists.zarr`
- MODIS: read the composite data of the maximum value over 8 days → `mdpr_lists.zarr`
- Reference sequences are composited to standard 8-day intervals aligned to 2020

### Step 2 — Harmonic Coefficient Fitting (Cell 3–4)

Fits a Ridge-regression harmonic model to each pixel time series using Dask for parallel spatial block processing.

- Reference datasets (`ltre`, `mdre`): 3rd-order harmonics, fitted once
- Predicted Landsat (`ltpr`): 1st-order harmonics (sparse data)
- Predicted MODIS (`mdpr`): 3rd-order harmonics with 3× iterative outlier removal

Coefficients stored as `[constant, amp1, pha1, amp2, pha2, amp3, pha3]`.

### Step 3 — Harmonic Significance Pruning (Cell 5)

Removes insignificant higher-order harmonics based on energy ratio thresholds (97%):

- If zero-frequency + annual energy > 97% → mask 2nd and 3rd harmonics
- If adding semi-annual energy pushes ratio > 97% → mask 3rd harmonic only

### Step 4 — FMSF Fusion (Cell 6)

Constructs predicted-year harmonic coefficients by transferring the MODIS-to-Landsat ratio from the reference year:

```
amp_fmsf = amp_mdpr / amp_mdre × amp_ltre
pha_fmsf = pha_mdpr − pha_mdre + pha_ltre
```

A pixel-level **weight map** (based on Landsat observation count and temporal regularity) blends the HANTS and FMSF coefficient sets.If the number of observations per year exceeds 14, use the results from HANTS; if it is less than 14, use the results from FMSF.

### Step 5 — Reconstruction & Export (Cells 7–8)

Export multi-band GeoTIFF based on a custom time interval.

---

## Output

`FMNTS/rendvi_<year>.tif` — Multi-band GeoTIFF at Landsat resolution  
- Band count: determined by `interval` (e.g., 23 bands for 16-day steps over one year)  
- Projection: inherited from input Landsat files  
- NoData: `NaN`  
- Band names: `YYYY-MM-DD` strings

---

