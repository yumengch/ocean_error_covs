# Configuration Reference

The tool is driven by two levels of INI configuration files parsed with Python's
standard `configparser` library.

---

## Top-level `config.ini`

The top-level file controls which observation types to process, which
covariance matrices to estimate, and where to write the output.

```ini
[ObsTypes]
# Comma-separated list of observation config file names (stems or full paths).
# Each entry may be given with or without the .ini extension.
# Resolved first relative to the directory of config.ini, then relative to the
# current working directory.
obs_config_files = chlo, carbon

[Uncertainty]
# Comma-separated selection of covariance matrices to estimate.
# R       – observation error covariance
# HBH     – background error covariance in observation space
# R+HBH   – sum of both
estimate = R, HBH, R+HBH

[Output]
# Directory where output NetCDF files are written.
output_dir = output/all
# Stem of the intermediate binned-statistics file (no .nc extension).
binned_filename = binned
# Stem of the final covariance-estimates file (no .nc extension).
cov_filename = cov
```

---

## Per-observation-type `<obs>.ini`

Each observation type listed in `obs_config_files` has its own INI file with
four sections.

### `[General]`

```ini
[General]
# Short identifier used internally and in output variable names.
name = chlo
# Human-readable label written to output file attributes.
long_name = Phytoplankton Chlorophyll-a Concentration
```

### `[Input]`

```ini
[Input]
# Directory containing the input NetCDF innovation files.
input_dir = /path/to/innovation/files

# strftime-style filename pattern (% must be doubled because configparser
# treats % as an escape character).
# See https://docs.python.org/3/library/datetime.html#strftime-and-strptime-format-codes
filename_format = owt_chlo_%%Y%%m.nc

# Temporal frequency of the input files, using NumPy datetime units:
# e.g. '6h' (6-hourly), '1D' (daily), '1M' (monthly), '1Y' (yearly).
# Leave empty to load every NetCDF file found in input_dir.
data_freq = 1M

# Date range to process (NumPy datetime format).
data_start = 2015-02-01T00:00:00.00
data_end   = 2016-12-01T00:00:00.00

# Comma-separated list of variables to load from each file.
# Must include at least the innovation/residual variables and lon, lat.
# Include any additional fields needed by the predictor mask function.
variables = d_ob, d_ab, d_oa, lon, lat

# Mapping from required internal names to variable names in the NetCDF files.
d_ob_name = d_ob   # observation minus background
d_ab_name = d_ab   # analysis increment in observation space
d_oa_name = d_oa   # observation minus analysis
lon_name  = lon
lat_name  = lat

# Variable holding the integer predictor bin index for each observation.
# Omit if no predictor is used (all observations treated as one bin),
# or if `mask_func` is used.
pred_name = mask
```

### `[Predictor]`

```ini
[Predictor]
# Name of the predictor (used in output variable names).
name = OWT

# Total number of predictor bins (bin indices run from 0 to n_classes-1).
n_classes = 14

# Maximum number of observations to load per time step.
# A random subset is drawn when the file exceeds this limit.
max_obs_per_day = 10000

# minimum number of samples to compute covariance estimates
n_samples = 100

# window size for smoothing correlations, better if even
smooth_window = 6
```

### `[Grid]`

Defines the discrete radial distance bins used to compute spatial correlations.
If the distance grid is `[0, 200, 400, ...]` km, then the covariance at lag
200 km is estimated from all observation pairs whose separation falls in the
interval (0, 200] km, and so on.

```ini
[Grid]
# Upper bound of the distance grid in km.
max_distance = 10000

# Width of each distance bin in km.
distance_interval = 200
```

---

## Provided example configs

| File | Predictor | Notes |
|------|-----------|-------|
| `OWT.ini` | Optical Water Type (14 classes) | Daily ocean-colour files |
| `Bathy.ini` | Ocean bathymetry (1 class, i.e. no stratification) | Daily files |
| `chlo.ini` | OWT (monthly files) | Chlorophyll-a |
| `carbon.ini` | OWT (monthly files) | Particulate organic carbon |
| `chlo_all.ini` / `carbon_all.ini` | No predictor (global pooling) | Pooled over all bins |
| `chlo_lat_month.ini` / `carbon_lat_month.ini` | Latitude band × month | Seasonal/latitudinal stratification |
