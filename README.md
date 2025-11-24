# NINJA FROST Monitor Page

This repository contains a single HTML page that monitors the NINJA FROST DAQ status during the NINJA physics run c (E71c).

## Overview

The page shows:

- Current run number and last modified time of the latest `.dat` file (JST).
- A traffic-light style status lamp and banner (green/OK or red/warning).
- Automatically updated data quality plots for the **latest run**.
- Reference plots for the **previous valid run** (if available).
- An audible alarm when data have not been updated for more than a fixed time.

All updates happen via JavaScript using `fetch` and timers. The page itself does **not** reload.

## Status Logic

### Time Threshold

- The status threshold is defined in minutes:

  ```js
  const THRESHOLD_MINUTES = 15;
  ```

- The latest timestamp is obtained from:

  ```js
  const DATE_JS = "figs/latestdat_info/latestdat_date.js";
  ```

  This file must call:

  ```js
  document.write("YYYY/MM/DD HH:MM");
  ```

- The function `evaluateStatusAndAlarm()` compares the current browser time to this timestamp.
  - If the difference is **≤ 15 minutes**, the status is **OK** (green lamp, green banner, no alarm).
  - If the difference is **> 15 minutes**, the status is **warning** (red lamp, red blinking banner, alarm sound).

### Latest Run

- The latest run ID is read from:

  ```js
  const RUN_JS = "figs/latestdat_info/latestdat_run.js";
  ```

  which must write a string like:

  ```js
  document.write("run00005");
  ```

- `updateRunDependentPlots(runVal)` uses this run ID to build file paths for the latest plots:

  - `figs/dataquality/unixtime/<run>_evnum_vs_unixtime.png`
  - `figs/dataquality/spillnum/<run>_evnum_vs_spillnum.png`
  - `figs/dataquality/tdc/<run>_leading_hist.png`
  - `figs/dataquality/tdc/<run>_trailing_hist.png`
  - `figs/dataquality/tdc/<run>_leading_fromadc_hist.png`
  - `figs/dataquality/tdc/<run>_trailing_fromadc_hist.png`
  - `figs/dataquality/lightyield/<run>_chavg_lightyield_hist.png`
  - `figs/dataquality/xgyg/<run>_xgyg.png`

These plots are shown in the **Data quality check** section.

## Previous Run Reference Plots

The page also shows reference plots for the **previous valid run** at the bottom.

### How the Previous Run Is Chosen

1. Read the latest run string, e.g. `run00005`.
2. Parse the numeric part (`5`) and its width (`5` digits).
3. Search downwards: `run00004`, `run00003`, …, `run00001`.
4. For each candidate, check if the representative file exists:

   ```js
   figs/dataquality/lightyield/<candidate>_chavg_lightyield_hist.png
   ```

5. The first candidate that exists is used as the *previous run*.
6. If **no valid previous run is found**, the *Previous data quality (reference)* section stays hidden.

### Previous Run Plots

When a previous run is found, `updatePrevRunDependentPlots(prevRunVal)` loads the same kinds of plots as for the latest run, but into separate `<img>` tags in the “Previous data quality (reference)” section.

## Sound and Alarm Behavior

- The alarm sound is provided by:

  ```html
  <audio id="beep-sound" preload="auto">
    <source src="se_amb01.wav" type="audio/wav">
  </audio>
  ```

- Browsers require user interaction before autoplay with sound, so the page shows an **Enable sound** bar:

  - First click:
    - Plays a very short low-volume sound to unlock audio permission.
    - Enables real alarms.
  - Subsequent clicks:
    - Toggle a **test alarm** on/off (looped playback).

- When the status is warning (data stale), `startOrStopAlarm(true)` is called:
  - If sound is enabled, the alarm loops.
  - A watchdog periodically checks and restarts the alarm if needed.

## Automatic Updates

- On page load:

  ```js
  window.addEventListener('load', () => {
    evaluateStatusAndAlarm();
    setInterval(evaluateStatusAndAlarm, 30000); // every 30s
  });
  ```

- Every 30 seconds:
  - Re-reads `latestdat_run.js` and `latestdat_date.js`.
  - Updates the main plots and status.
  - (Only once) detects and shows the previous run reference plots.

- When the tab becomes visible again, the status is re-evaluated immediately.

## Required Files and Directory Structure

The page assumes the following structure (partial):

```text
figs/
  latestdat_info/
    latestdat_run.js        # document.write("run00005")
    latestdat_date.js       # document.write("YYYY/MM/DD HH:MM")
  dataquality/
    unixtime/runXXXXX_evnum_vs_unixtime.png
    spillnum/runXXXXX_evnum_vs_spillnum.png
    tdc/runXXXXX_leading_hist.png
    tdc/runXXXXX_trailing_hist.png
    tdc/runXXXXX_leading_fromadc_hist.png
    tdc/runXXXXX_trailing_fromadc_hist.png
    lightyield/runXXXXX_chavg_lightyield_hist.png
    xgyg/runXXXXX_xgyg.png
  dataquality_withBSD/
    pot_info/*.js
    pot_plot/accumulated_pot_withBSD.png
    eventrate_plot/eventrate_plot.png
se_amb01.wav                 # alarm sound
```

Make sure these files are generated and updated by your DAQ-side scripts.

## How to Use

1. Place the HTML file and the `figs/` directory on a web server.
2. Ensure that:
   - `latestdat_run.js` and `latestdat_date.js` are updated as new `.dat` files appear.
   - Plot PNGs for each run are generated under the expected paths.
   - `se_amb01.wav` is accessible.
3. Open the page in a browser.
4. Click **Enable sound (click once)** to allow alarm playback.
5. Monitor:
   - Green banner/lamp: DAQ is updating within the threshold.
   - Red blinking banner + alarm: DAQ has not updated for more than 15 minutes.
   - Compare the latest plots with the previous-run reference plots at the bottom.
