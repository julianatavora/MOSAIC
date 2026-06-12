# MOSAIC

**MOSAIC (Multiple Wavelength Algorithm for Estimates of Light Absorption and Backscattering Properties in Turbid Waters)** is a semi-analytical inversion algorithm designed to retrieve inherent optical properties (IOPs) and suspended particulate matter (SPM) from red and near-infrared water-leaving reflectance measurements.

The algorithm was developed for highly turbid estuarine, coastal, and inland waters where traditional blue-green ocean color inversion methods often fail. MOSAIC exploits the 650–850 nm spectral region and combines a Linear Matrix Inversion (LMI) framework with an ensemble-based uncertainty analysis.

## Scientific Background

MOSAIC simultaneously estimates:

* Non-algal particle absorption (*a<sub>NAP</sub>*)
* Colored dissolved organic matter absorption (*a<sub>CDOM</sub>*)
* Phytoplankton absorption (*a<sub>phy</sub>*)
* Particle backscattering (*b<sub>bp</sub>*)
* Suspended particulate matter concentration (SPM)
* Water temperature (optional optimization parameter)

The inversion is based on:

1. Conversion of water reflectance to subsurface remote sensing reflectance (*rrs*).
2. Transformation of *rrs* into the Gordon–Lee reflectance parameter *u*.
3. Linear Matrix Inversion using large ensembles of spectral eigenvectors.
4. Selection of physically realistic solutions.
5. Ensemble averaging and uncertainty estimation.
6. SPM retrieval through particle backscattering relationships.

## Reference

If you use this code, please cite:

> Pereira, J. T. B., et al. (2026).
> *MOSAIC: Multiple wavelength algorithm for estimates of light absorption and backscattering properties in turbid waters*.
> SSRN Preprint.
> https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6138756

## Features

* Works with hyperspectral and multispectral reflectance data
* Designed for turbid waters
* Simultaneous retrieval of multiple IOPs
* Ensemble inversion framework
* Explicit uncertainty quantification
* Supports image processing and point measurements
* Temperature-dependent pure-water absorption correction

## Inputs

### Required

| Variable        | Description                     |
| --------------- | ------------------------------- |
| `RW`            | Water-leaving reflectance (ρw)  |
| `RW_std`        | Reflectance uncertainty         |
| `wavelength`    | Wavelength vector (nm)          |
| `conv_criteria` | Inversion convergence threshold |
| `temp`          | Water temperature (°C)          |

### Spectral Range

MOSAIC uses wavelengths between:

```text
650–850 nm
```

with optional exclusion of atmospheric absorption regions around:

```text
745–773 nm
```

## Outputs

The inversion returns:

| Variable            | Description                        |
| ------------------- | ---------------------------------- |
| `anap_model_mean`   | NAP absorption at 443 nm           |
| `acdom_model_mean`  | CDOM absorption at 440 nm          |
| `bbp_model_mean`    | Particle backscattering at 700 nm  |
| `aphyt_model_mean`  | Phytoplankton absorption magnitude |
| `Sanap_model_mean`  | NAP spectral slope                 |
| `Sacdom_model_mean` | CDOM spectral slope                |
| `Ybbp_model_mean`   | Backscattering spectral exponent   |
| `SPM`               | Suspended particulate matter       |
| `temp`              | Estimated water temperature        |

Associated uncertainty estimates are also provided for all retrieved quantities.

## Algorithm Workflow

```text
ρw
 │
 ▼
rrs conversion
 │
 ▼
u transformation
 │
 ▼
Generation of spectral eigenvectors
 │
 ├── NAP absorption
 ├── CDOM absorption
 ├── Particle backscattering
 └── Phytoplankton absorption
 │
 ▼
Linear Matrix Inversion
 │
 ▼
Physical filtering
 │
 ▼
Ensemble of acceptable solutions
 │
 ▼
Mean estimates + uncertainties
 │
 ▼
SPM retrieval
```

## Example

```matlab
results = IOP_inversion( ...
    RW, ...
    RW_std, ...
    wavelength, ...
    0.05, ...
    temperature);
```

Access retrieved SPM:

```matlab
spm = results.SPM;
spm_unc = results.SPM_unc;
```

Retrieve IOPs:

```matlab
anap  = results.anap_model_mean;
acdom = results.acdom_model_mean;
bbp   = results.bbp_model_mean;
```

## Repository Structure

```text
.
├── IOP_inversion.m
├── phyto_avg_field.m
├── weight_asses_field.m
├── README.md
└── examples/
```

## Computational Notes

For image inversions, the algorithm evaluates thousands of possible combinations of:

* NAP spectral slopes
* CDOM spectral slopes
* Backscattering exponents
* Temperature scenarios

Large scenes can therefore require substantial computation time and memory.

Checkpoint saving is implemented during image processing to prevent loss of intermediate results.

## Limitations

* Optimized for turbid waters.
* Retrieval performance depends on the quality of atmospheric correction.
* Accuracy decreases when reflectance uncertainty is high.
* Spectral coverage below 650 nm is not currently used.

## License

Please specify the license adopted by this repository (e.g., MIT, GPL-3.0, BSD-3-Clause).

## Contact

**Juliana Tavora Bertazo Pereira**

If you use MOSAIC in publications, please cite the associated preprint and consider opening an issue for questions, bug reports, or feature requests.
