# EigenBand Architecture Plan

## Context

The EigenBand plugin scaffold is in place. The `run()` method in `plugin_main.py` is a stub. We need to implement multi-band PCA dimensionality reduction with both a Processing algorithm and a standalone dialog, using pure numpy.

## New Directory Structure

```
eigenband/
├── __init__.py              (update: lazy-load provider)
├── __about__.py             (no change)
├── metadata.txt             (update: hasProcessingProvider=True)
├── plugin_main.py           (update: register provider, add toolbar action for dialog)
├── gui/
│   ├── dlg_settings.py/.ui  (existing — no change)
│   ├── dlg_pca.py           (NEW — standalone PCA dialog)
│   └── dlg_pca.ui           (NEW — Qt Designer UI)
├── processing/
│   ├── __init__.py          (NEW)
│   ├── provider.py          (NEW — EigenbandProvider)
│   └── pca_algorithm.py     (NEW — PCA Processing algorithm)
├── core/
│   ├── __init__.py          (NEW)
│   └── pca.py               (NEW — pure numpy PCA, no QGIS imports)
├── toolbelt/                (existing — no change)
└── resources/               (existing — no change)
```

## Module Responsibilities

### `core/pca.py` — Pure computation

The heart of the plugin. Must be testable without QGIS.

```python
def pca(
    data: np.ndarray,          # shape: (n_pixels, n_bands)
    n_components: int = 3,
    standardize: bool = True,  # zero-mean, unit-variance per band
) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
    # Returns: (transformed, explained_variance_ratio, component_loadings)
```

- Uses `np.linalg.svd` for decomposition
- Standardizes input bands before PCA (critical for multi-spectral data with different scales)
- Returns explained variance ratio so the dialog can show it
- Handles NoData by masking before computation

### `processing/provider.py` — Processing provider

```python
class EigenbandProvider(QgsProcessingProvider):
    def id(self) -> str:           # "eigenband"
    def name(self) -> str:         # "EigenBand"
    def loadAlgorithms(self):      # registers PcaAlgorithm
```

### `processing/pca_algorithm.py` — Processing algorithm

Parameters:
- `INPUT` — QgsProcessingParameterRasterLayer
- `BANDS` — QgsProcessingParameterBand (multi-select, lets user pick which bands)
- `N_COMPONENTS` — QgsProcessingParameterNumber (default 3, min 1)
- `OUTPUT` — QgsProcessingParameterRasterDestination

Flow in `processAlgorithm()`:
1. Read selected bands from input raster via `QgsRasterDataProvider`
2. Flatten to 2D array, mask NoData
3. Call `core.pca.pca()`
4. Reshape result to raster dimensions
5. Write output as GeoTIFF via `QgsRasterFileWriter`
6. Return output path

### `gui/dlg_pca.py` — Standalone dialog

UI elements:
- `QgsMapLayerComboBox` — filtered to raster layers
- Checkable band list — populated when layer selection changes
- `QSpinBox` for n_components (default 3)
- Output section: radio buttons for temp layer vs. file path + `QgsFileWidget`
- Run button
- Progress bar + explained variance display

Execution:
- Run PCA in a `QgsTask` to avoid blocking the UI
- On completion: add output layer to canvas with auto RGB styling (2-98 percentile stretch)

### `plugin_main.py` — Updated

- Register `EigenbandProvider` in `initGui()`
- Add toolbar action launching `dlg_pca` dialog
- Unregister provider and clean up in `unload()`

### `metadata.txt` — Updated

- `hasProcessingProvider=True`

## Data Flow

```
User picks raster + bands + n_components
        │
        ▼
Read bands as numpy array (QgsRasterDataProvider.block())
        │
        ▼
Mask NoData pixels
        │
        ▼
Standardize per-band (zero mean, unit variance)
        │
        ▼
core.pca.pca()  ──►  np.linalg.svd  ──►  top N components
        │
        ▼
Reshape to (height, width, n_components)
        │
        ▼
Write GeoTIFF (QgsRasterFileWriter)
        │
        ▼
Add to canvas + apply RGB style (percentile stretch)
```

## Implementation Order

1. **`core/pca.py`** + unit tests (`tests/unit/test_pca.py`) — pure numpy, testable immediately with known datasets
2. **`processing/provider.py`** + **`processing/pca_algorithm.py`** — wire PCA into Processing framework
3. **`gui/dlg_pca.py`** + **`dlg_pca.ui`** — standalone dialog
4. **`plugin_main.py`** — register provider + add dialog toolbar action
5. **`metadata.txt`** — flip `hasProcessingProvider=True`
6. **QGIS integration tests** (`tests/qgis/test_pca_algorithm.py`)

## Test Plan

- `tests/unit/test_pca.py` — verify PCA math with synthetic data (known eigenvalues), NoData handling, standardization. Run with `pytest -p no:qgis tests/unit/`
- `tests/qgis/test_pca_algorithm.py` — test Processing algorithm end-to-end with a synthetic raster layer
- `tests/qgis/test_dlg_pca.py` — test dialog widget: band list populates, run button triggers task

## Key Design Decisions

- **Pure numpy, no sklearn**: numpy ships with QGIS. SVD-based PCA is mathematically identical to sklearn's. Zero external dependencies.
- **`core/` has no QGIS imports**: keeps PCA logic testable in `tests/unit/` without QGIS runtime.
- **Processing algorithm is the primary interface**: supports batch mode, modeler integration, history. Dialog is a convenience wrapper.
- **NoData handling**: mask NoData pixels before PCA, write NoData back to output. Don't let NoData corrupt the covariance matrix.
- **Output styling**: auto-apply 2-98 percentile stretch to RGB bands for good visual contrast.
