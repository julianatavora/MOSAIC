# MOSAIC

**MOSAIC (Multiple Wavelength Algorithm for Estimates of Light Absorption and Backscattering Properties in Turbid Waters)** is a semi-analytical inversion algorithm designed to retrieve inherent optical properties (IOPs) and suspended particulate matter (SPM) from red and near-infrared water-leaving reflectance measurements.

The algorithm was developed for highly turbid estuarine, coastal, and inland waters where traditional blue-green ocean color inversion methods often fail. MOSAIC exploits the 650–850 nm spectral region and combines a Linear Matrix Inversion (LMI) framework, built from a large ensemble of candidate spectral shapes, with a physically-constrained selection step and an ensemble-based uncertainty analysis.

## Scientific Background

MOSAIC simultaneously estimates:


* Particle backscattering (*b<sub>bp</sub>*) and its spectral exponent (*Y<sub>bbp</sub>*)
* Suspended particulate matter concentration (SPM)
* Water temperature (used as an optimization/fallback parameter when not supplied)
* Non-algal particle absorption (*a<sub>NAP</sub>*) and its spectral slope
* Colored dissolved organic matter absorption (*a<sub>CDOM</sub>*) and its spectral slope
* Phytoplankton absorption magnitude (*a<sub>phy</sub>*)

Retrieved quantities fall into two groups:

- **Primary outputs** — directly retrieved by the linear inversion and forward-model convergence check: particle backscattering (*b<sub>bp</sub>*) and its spectral exponent (*Y<sub>bbp</sub>*), SPM, and water temperature.
- **Secondary outputs** — derived afterward by fitting a parametric shape (via nonlinear least squares) to the ensemble-mean spectra: NAP absorption at 443 nm and its slope, CDOM absorption at 440 nm and its slope, and phytoplankton absorption magnitude.

The inversion is based on:

1. Conversion of water-leaving reflectance (ρw) to subsurface remote-sensing reflectance (*rrs*) via `rrs = (ρw/π) / (0.52 + 1.7·(ρw/π))`.
2. Per-pixel transformation of *rrs* into the Gordon/Lee reflectance parameter *u* (`bb/(a+bb)`) by solving the quadratic model of Wang, Boss & Roesler (2005), using fixed coefficients L3 = 0.0949 and L4 = 0.0794.
3. Generation of a large ensemble of candidate spectral eigenvectors: NAP absorption slopes, CDOM absorption slopes, backscattering spectral exponents, and phytoplankton absorption shapes.
4. Linear Matrix Inversion, solved independently for every combination of the above eigenvectors (and, when temperature is unknown, for every candidate temperature).
5. Selection of physically realistic solutions, followed by a forward-model convergence check against the measured reflectance.
6. Ensemble averaging of the accepted solutions and calculation of associated uncertainties.
7. SPM retrieval by testing a grid of candidate backscattering-to-SPM conversion factors against the retrieved *b<sub>bp</sub>* spectra, followed by a reflectance-uncertainty-weighted average.

## Inputs

### Required

| Variable        | Description                                                                 |
| --------------- | ---------------------------------------------------------------------------- |
| `RW`            | Water-leaving reflectance (ρw). A `[N × λ]` matrix for point spectra, or a `[rows × cols × λ]` cube for images. |
| `RW_std`        | Reflectance uncertainty, same shape as `RW`.                                  |
| `wavelength`    | Wavelength vector (nm), matching the spectral dimension of `RW`.             |
| `conv_criteria` | Convergence threshold: fraction of measured *rrs* that the forward-modeled *rrs* must match at every band (10 or 25%; see "Convergence check" below). |
| `temp`          | Water temperature (°C). Scalar or per-pixel vector/matrix. `NaN` entries fall back to a search over a range of temperatures (by default 5–34 °C; see below). |

### Temperature fallback behavior

If `temp` contains `NaN` values:
- For **image cubes**, MOSAIC first checks whether any valid (non-NaN) temperatures exist anywhere in the scene. If so, it computes the scene-wide mean ± standard deviation once and uses that range as the candidate temperature set for every NaN pixel. If no valid temperatures exist anywhere, it falls back to a fixed `5–34 °C` range.
- For **point spectra**, any `NaN` temperature is simply replaced by a fixed `5–34 °C` candidate range (no scene-statistics fallback is computed in this mode).

### Convergence check

A candidate solution is accepted only if, at every wavelength, the forward-modeled *rrs* falls within a tolerance of the measured *rrs*. That tolerance is `max(conv_criteria × rrs, 0.001)` for wavelengths ≥ 700 nm (i.e., a small absolute floor is enforced in addition to the relative criterion), and simply `conv_criteria × rrs` below 700 nm.

## Outputs

The inversion returns a `results` struct.

### Primary outputs

| Variable            | Description                                    |
| ------------------- | ----------------------------------------------- |
| `bbp_model_mean`    | Particle backscattering at 700 nm               |
| `Ybbp_model_mean`   | Backscattering spectral exponent                |
| `SPM`               | Suspended particulate matter concentration      |
| `temp`              | Estimated/used water temperature (mean of accepted candidates) |

### Secondary outputs

| Variable            | Description                                    |
| ------------------- | ----------------------------------------------- |
| `anap_model_mean`   | NAP absorption at 443 nm (fitted from ensemble mean spectrum) |
| `acdom_model_mean`  | CDOM absorption at 440 nm (fitted from ensemble mean spectrum) |
| `aphyt_model_mean`  | Phytoplankton absorption magnitude (ensemble mean) |
| `Sanap_model_mean`  | NAP spectral slope (fitted)                     |
| `Sacdom_model_mean` | CDOM spectral slope (fitted)                    |

### Diagnostics and uncertainty

| Variable             | Description                                                        |
| --------------------- | ------------------------------------------------------------------- |
| `nn`                  | Number of accepted ensemble members ("N") used to build the pixel's estimates |
| `anap_model_error`    | Uncertainty on `anap_model_mean`                                   |
| `acdom_model_error`   | Uncertainty on `acdom_model_mean`                                  |
| `bbp_model_error`     | Uncertainty on `bbp_model_mean`                                    |
| `aphyt_model_error`   | Uncertainty on `aphyt_model_mean`                                  |
| `Sanap_model_error`   | Uncertainty on `Sanap_model_mean`                                  |
| `Sacdom_model_error`  | Uncertainty on `Sacdom_model_mean`                                 |
| `Ybbp_model_error`    | Uncertainty on `Ybbp_model_mean`                                   |
| `SPM_unc`             | Uncertainty on `SPM` (derived from the 16th/84th percentile spread of the weighted ensemble, scaled by 1/√5) |
| `temp_unc`            | Uncertainty on `temp` (standard deviation across accepted temperature candidates) |

If a pixel has no valid reflectance data, or no candidate solution converges (`N = 0`), its outputs remain `NaN`.

## Algorithm Workflow

```text
ρw, ρw_std
 │
 ▼
rrs conversion  (rrs = (ρw/π) / (0.52 + 1.7·(ρw/π)))
 │
 ▼
Per-pixel / per-temperature-candidate loop
 │
 ├── Solve u = f(rrs) quadratic (Wang, Boss & Roesler, 2005)
 ├── Build eigenvector library:
 │     • NAP absorption slopes        (Snap:  0.001–0.012, step 0.002)
 │     • CDOM absorption slopes       (Scdom: 0.002–0.016, step 0.002)
 │     • Backscattering exponents     (Y:     0–1.6,       step 0.1)
 │     • Phytoplankton absorption shapes (5 cluster-derived shapes)
 ├── Solve K = nSnap × nScdom × nY × nSf linear systems (one per combination)
 ├── Keep solutions with all 4 coefficients > −0.002 (physical filter)
 ├── Forward-model rrs from candidates; keep those within conv_criteria of measured rrs
 └── Accumulate accepted solutions across all temperature candidates
 │
 ▼
Ensemble mean + std per IOP  →  anap, acdom, bbp, aphyt, temp (+ uncertainties)
 │
 ▼
Nonlinear curve fit (fminsearch) on ensemble-mean spectra
 →  NAP/CDOM slopes + 443/440 nm reference absorption
 │
 ▼
SPM retrieval: grid-search candidate bbp→SPM conversion factors,
reflectance-uncertainty-weighted average, percentile-based uncertainty
```

## Example

```matlab
[results] = IOP_inversion(RW, RW_std, wavelength, conv_criteria, sst);
```

Access retrieved SPM:

```matlab
spm     = results.SPM;
spm_unc = results.SPM_unc;
```

Retrieve primary IOPs:

```matlab
bbp  = results.bbp_model_mean;
ybbp = results.Ybbp_model_mean;
temp = results.temp;
```

Retrieve secondary IOPs:

```matlab
anap  = results.anap_model_mean;
acdom = results.acdom_model_mean;
aphyt = results.aphyt_model_mean;
```

## Repository / Code Structure

`IOP_inversion.m` is the single entry-point script. Internally it defines the following **local functions** (not separate files):

```text
IOP_inversion.m
│
├── IOP_inversion(...)            % main entry point / driver
├── get_AP_coeff_slope(...)       % fits NAP slope + 443 nm reference absorption
├── get_ACDOM_coeff_slope(...)    % fits CDOM slope + 440 nm reference absorption
├── get_BBP_coeff_slope(...)      % fits bbp exponent + 700 nm reference value
├── insider_inversion(...)        % per-pixel / per-temperature LMI + physical filtering
├── asw_corr(...)                 % temperature-corrected pure-water absorption
├── weight_asses_field(...)       % reflectance-uncertainty weighting for SPM
├── v(...)                        % solves for Gordon/Lee "u" parameter
├── phyto_avg_field(...)          % 5 cluster-derived phytoplankton absorption shapes
└── Array_h(...)                  % builds the known-term array for the LMI system
```

## Computational Notes

For image inversions, the algorithm evaluates every combination of:

* NAP spectral slopes
* CDOM spectral slopes
* Backscattering exponents
* Candidate temperatures (when `temp` contains `NaN`)

Large scenes can therefore require substantial computation time and memory.

## Limitations

* Optimized for turbid waters; not intended for clear open-ocean conditions. Typically struggles in low turbidity
* Retrieval performance depends on the quality of atmospheric correction.
* Accuracy decreases when reflectance uncertainty is high.
* Spectral coverage below 650 nm is not currently used.

## License

Please specify the license adopted by this repository (e.g., MIT, GPL-3.0, BSD-3-Clause).

## Reference

If you use this code, please cite:

> Tavora, J., et al. (2026).
> *An algorithm for the retrieval of particle backscattering, size, and suspended particulate matter in turbid coastal and estuarine waters (MOSAIC)*.
> Remote Sensing of Environment.
> (https://doi.org/10.1016/j.rse.2026.115629) 

## Contact

**Juliana Tavora Bertazo Pereira**

If you use MOSAIC in publications, please cite the associated preprint and consider opening an issue for questions, bug reports, or feature requests.
