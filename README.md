# Galveston Beachfront Water Temperature, 1995–present

A continuous daily record of Gulf water temperature at the Galveston seawall. It splices NOAA's discontinued **Flagship / Pleasure Pier** gauge (1995–2011) onto the **Galveston Bay Entrance North Jetty** gauge, after adjusting the jetty to read like the beach.

The goal is the water temperature right off the beach at 25th Street, not offshore and not in the bay.

---

## Contents

- [Data sources](#data-sources)
- [Processing pipeline](#processing-pipeline)
  - [1. Download](#1-download)
  - [2. Quality control](#2-quality-control)
  - [3. Daily averages](#3-daily-averages)
  - [4. Pier sensor drift, 2008–2010](#4-pier-sensor-drift-20082010)
  - [5. Fitting the jetty to the beach](#5-fitting-the-jetty-to-the-beach)
  - [6. Splicing the record](#6-splicing-the-record)
- [Derived statistics](#derived-statistics)
- [Output files](#output-files)
- [Limitations](#limitations)
- [Reproducing](#reproducing)

---

## Data sources

All data comes from NOAA CO-OPS (Center for Operational Oceanographic Products and Services) through the [Tides & Currents data API](https://api.tidesandcurrents.noaa.gov/api/prod/).

| Station | ID | Location | Water temp record | Role |
|---|---|---|---|---|
| Galveston Pleasure Pier (Flagship Pier) | `8771510` | SE corner of the pier at 25th St, Gulf side | Mar 1995 – Jul 19, 2011 (station removed) | **Target**: the beachfront reference |
| Galveston Bay Entrance, North Jetty | `8771341` | Mouth of the Houston Ship Channel, ~6 mi NE of the pier | Apr 2000 – present (gap late 2008 – 2010) | **Proxy** after the pier closed |
| Eagle Point | `8771013` | Upper Galveston Bay | 1990s – present | Independent check only, not used in the record |

Pier 21 (`8771450`) also has water temperature from 1995. It sits on the harbor side of the island and responds like the bay, so it was not used.

The pier gauge dates to 1957 for water level. Its electronic water temperature sensor arrived with NOAA's Next Generation Water Level Measurement System rollout, so temperature starts in 1995.

---

## Processing pipeline

### 1. Download

Hourly water temperature (`product=water_temperature`, `interval=h`, `units=english`, `time_zone=lst`) was requested one calendar year at a time, because the API limits hourly requests to about one year. Each hourly record has a value `v` and a quality flag string `f`.

### 2. Quality control

These rules were applied in order:

1. **NOAA flags.** Any hour with a nonzero flag (max or min exceeded, rate of change) is dropped.
2. **Daily completeness.** A day is kept only if at least **18 of 24 hours** survive.
3. **Physical range.** Daily averages outside **38–96 °F** are dropped. This removed 136 pier days in 1996 when the sensor reported about −21 °F.
4. **Spike filter.** A day is dropped if it differs by more than **8 °F** from the median of the surrounding ±7 days, with at least 4 neighbors required. This removed 5 pier days and 8 jetty days.

### 3. Daily averages

Each surviving day's temperature is the plain mean of its valid hours. Daily highs and lows in the dashboard are therefore the warmest and coolest **daily averages**, not single hourly readings.

### 4. Pier sensor drift, 2008–2010

Comparing the pier against the jetty and Eagle Point month by month shows a sustained warm bias in the pier record:

| Period | Pier − North Jetty | Pier − Eagle Point (same months, prior years) |
|---|---|---|
| 2004–2007 | about +0.5 °F, stable | about 0 °F in summer |
| Jan – Sep 2008 | **+1.7 to +3.2 °F** | +1.7 to +2.9 °F |
| Oct 2008 – Mar 2010 | jetty offline | **+2 to +5 °F**, up to +9 °F in winter |
| Apr 2010 – Jul 2011 | jetty offline | back to normal |

The bias starts before Hurricane Ike (September 2008) and ends suddenly in spring 2010, which suggests sensor drift followed by replacement or recalibration rather than a real change in the water. **Pier data from 2008‑01‑01 through 2010‑03‑31 is flagged `pier_suspect = yes` and left out of the combined record.** Without this step, 2009 shows up as the warmest year on record by a wide margin.

### 5. Fitting the jetty to the beach

**Training data:** days from 2001‑02‑21 to 2007‑12‑31 with good readings at both stations, **2,002 days** in all. The suspect 2008 data is left out.

**Candidate models**, predicting the pier from the jetty:

| Model | Form |
|---|---|
| Constant | `pier = jetty + c` |
| Monthly | `pier = jetty + c[month]` |
| Linear | `pier = a + b·jetty` |
| Linear + season | `pier = a + b·jetty + c·sin(ωt) + d·cos(ωt)` |
| Linear + season × jetty | adds `jetty·sin`, `jetty·cos` terms |
| **Binned by jetty temp** | `pier = jetty + median_offset[jetty bin]`, interpolated |

**Validation:** leave-one-year-out cross-validation. Each year is predicted from a model fit on all other years.

| Model | CV RMSE (°F) | CV MAE (°F) |
|---|---|---|
| No adjustment | 1.66 | — |
| Constant | 1.48 | 0.89 |
| Monthly | 1.52 | 0.92 |
| Linear (`a = 2.43`, `b = 0.977`) | 1.48 | 0.89 |
| Linear + season | 1.48 | 0.88 |
| Linear + season × jetty | 1.47 | 0.87 |
| **Binned by jetty temp** | **1.48** | **0.87** |

All models land within a few hundredths of a degree of each other. The remaining ~1.5 °F daily error is mostly real short-term noise: fronts, tides, and wind-driven mixing that hit the two sites on different days. The **binned model** was chosen because it ties for best and can be checked directly:

| Jetty temperature | Median jetty in bin | Beach warmer by | Days |
|---|---|---|---|
| Below 55 °F | 53.1 °F | **+1.1 °F** | 129 |
| 55–65 °F | 59.7 °F | +0.7 °F | 433 |
| 65–75 °F | 70.3 °F | +0.8 °F | 423 |
| 75–82 °F | 78.9 °F | +0.7 °F | 359 |
| 82–86 °F | 84.3 °F | +0.4 °F | 435 |
| Above 86 °F | 86.9 °F | **+0.2 °F** | 223 |

The adjustment is a piecewise-linear interpolation between bin medians, held constant beyond the end bins:

```
pier_equiv = jetty + interp(jetty, [53.1, 59.7, 70.3, 78.9, 84.3, 86.9],
                                   [ 1.1,  0.7,  0.8,  0.7,  0.4,  0.2])
```

**Reading the result:** the beach is not more extreme than the jetty. It runs slightly **warmer**, and the gap is largest in cold water. A likely reason is that the jetty sits in the tidal outflow of Galveston Bay, which cools quickly behind winter fronts. In the summer heat the two sites nearly match. Averaged by month, the adjusted jetty matches the pier with an RMSE of **0.6 °F**.

### 6. Splicing the record

For each day, the `combined_f` value is chosen as follows:

1. **Pier**, if the date is on or before 2011‑07‑19, the pier passed QC, and the date is outside the suspect window.
2. Otherwise, the **adjusted North Jetty**, if available.
3. Otherwise, missing.

| Source | Days |
|---|---|
| Pier | 4,850 |
| North Jetty (adjusted) | 5,791 |
| Missing | 903 |

The largest gap is **Oct 2008 – Mar 2010**: the pier is suspect and the jetty was out of service after Hurricane Ike.

---

## Derived statistics

All of these are computed from `combined_f`.

| Statistic | Definition |
|---|---|
| **Normal for a date** | 10th, 50th, and 90th percentiles of daily values from 1996–2025, pooled over a ±7-day window around the calendar date. Feb 29 is folded into Feb 28. |
| **Monthly average** | Mean of daily values, only for months with ≥ 20 days. |
| **Monthly departure** | Monthly average minus the mean of that calendar month across 1996–2025. |
| **Yearly average** | Mean of daily values, only for years with ≥ 330 days. 1996, 2008, 2009, and 2010 fall short. |
| **Trend** | Ordinary least-squares slope of yearly averages against year, reported per decade. |
| **Records** | Extremes of daily (or monthly) averages over the combined record. |

---

## Output files

### `galveston_water_temp_daily.csv`

One row per calendar day from 1995‑03‑15 onward.

| Column | Description |
|---|---|
| `date` | ISO date (local standard time, UTC−6) |
| `pier_f` | Pier daily mean after QC (°F), blank if missing |
| `pier_suspect` | `yes` for 2008‑01‑01 – 2010‑03‑31 pier values left out of the combined record |
| `north_jetty_f` | North Jetty daily mean after QC (°F) |
| `north_jetty_adjusted_f` | Jetty value with the beach offset applied |
| `combined_f` | Best estimate of beachfront temperature |
| `combined_source` | `Pier` or `North Jetty (adjusted)` |

### `offsets.csv`

The binned offset table from step 5.

---

## Limitations

- **Two different instruments.** The adjustment corrects the average difference, not every day. Expect ±1.5 °F on any single day and about ±0.6 °F for a monthly average.
- **Fixed fit period.** The offset comes from 2001–2007. If either site's surroundings changed later (jetty work, the pier rebuild, sensor replacements), the relationship may have shifted without any way to check it.
- **Trend uses spliced data.** About half of the 1995–2026 trend period comes from each station. The splice is calibrated, but treat the 0.6 °F/decade figure as approximate.
- **Daily means hide extremes.** Shallow surf water can swing several degrees within a day. Hourly peaks are not represented.
- **Gap.** Oct 2008 – Mar 2010 cannot be recovered from these stations. It could be estimated from a bay station like Eagle Point, but those relate less closely to the beach.
- **Pre-1995 data.** Older surface water temperatures exist as handwritten tide-observer logs published by the Coast and Geodetic Survey. They are not digitized here.

---

## Reproducing

1. Download hourly `water_temperature` for `8771510` (1995–2011) and `8771341` (2000–present), one year per request:

   ```
   https://api.tidesandcurrents.noaa.gov/api/prod/datagetter
     ?product=water_temperature&station=<ID>
     &begin_date=<YYYY>0101&end_date=<YYYY>1231
     &interval=h&units=english&time_zone=lst&format=json
   ```

2. Apply the QC in step 2 and the daily averaging in step 3.
3. Mask the pier from 2008‑01‑01 to 2010‑03‑31.
4. Fit the binned offsets on 2001‑02‑21 – 2007‑12‑31 overlap days.
5. Splice as in step 6.

---

*Data: NOAA CO-OPS. Processed October 2026.*
