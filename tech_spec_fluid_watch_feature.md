# Tech Spec: Fluid Simulation Watch Feature

**Date:** 2025-05-09
**Author:** Roo
**Version:** 1.0

## 1. Introduction

This document outlines the technical specifications for implementing a "watch feature" within the existing circular boundary FLIP fluid simulation. The watch feature displays the current time (HH:MM) as a dynamic, physical obstacle integrated into the simulation environment.

The specifications are derived from a comparative analysis of two versions of the simulation code:
*   `18-flip_circular_orig.html`: The baseline version without the watch feature.
*   `18-flip_circular_watch.html`: The enhanced version incorporating the watch feature.

## 2. Feature Overview

The core functionality of the watch feature is to render the current time (hours and minutes) as a set of obstacles within the fluid simulation. These obstacles are formed by designating specific grid cells as "time cells."

Key characteristics:
*   **Time Display:** Shows time in HH:MM format.
*   **Physical Obstacle:** The digits of the clock act as solid boundaries for the fluid particles. The colon separating hours and minutes is visually present but allows fluid to pass through.
*   **Dynamic Sizing:** The size and position of the clock display are dynamically calculated based on the simulation's main circular boundary and the current grid resolution.
*   **Grid-Based:** The clock is rendered by changing the type and properties of the underlying simulation grid cells.
*   **Periodic Updates:** The displayed time updates automatically every minute.
*   **User-Adjustable Resolution:** The grid resolution can be changed by the user, and the clock display will rescale and re-render accordingly.

## 3. Core Components and Logic

### 3.1. New Cell Types

Two new cell types are introduced to manage the clock display:

*   **`TIME_CELL` (value: 3):** Represents the solid segments of the clock digits (0-9). These cells behave as impenetrable obstacles for the fluid.
    *   In the `FlipFluid.s` array (solidity), these cells have a value of `0.0`.
*   **`VIRTUAL_TIME_CELL` (value: 4):** Represents the cells forming the colon (`:`) between hours and minutes. These cells are visually distinct but are treated as passable by the fluid (like `AIR_CELL`s for fluid dynamics).
    *   In the `FlipFluid.s` array, these cells have a value of `1.0`.

These constants are defined globally:
```javascript
var TIME_CELL = 3; // New cell type for clock digits
var VIRTUAL_TIME_CELL = 4; // Cell type for colon: visual wall, but passable
```

### 3.2. Clock Configuration (`scene.clockConfig`)

A new configuration object, [`scene.clockConfig`](18-flip_circular_watch.html:783), is added to the global `scene` object to manage the properties of the clock display.

```javascript
scene.clockConfig: {
    digitWidth: 5,    // Width of a single digit in grid cells (calculated)
    digitHeight: 7,   // Height of a single digit in grid cells (calculated)
    colonWidth: 3,    // Width of the colon in grid cells (calculated)
    spacing: 1,       // Spacing between characters in grid cells (calculated)
    offsetX: 0,       // Horizontal start position of the clock in grid cells (calculated)
    offsetY: 0        // Vertical start position of the clock in grid cells (calculated)
}
```
The dimensions (`digitWidth`, `digitHeight`, `colonWidth`, `spacing`) are dynamically calculated in [`setupScene()`](18-flip_circular_watch.html:931) based on the main circle's diameter and the current grid resolution (`f.h`). The goal is to make the clock characters occupy a significant portion of the circle (e.g., 35% of height, 80% of width for the full "HH:MM" string).

### 3.3. Digit Patterns (`scene.clockDigitsData`)

A data structure, [`scene.clockDigitsData`](18-flip_circular_watch.html:781), stores the visual patterns for each numeral (0-9) and the colon. These are defined as 2D arrays of 0s and 1s.

*   **Master Patterns:** Base patterns for digits (7x5) and colon (7x4) are defined in [`initClockDigitsData()`](18-flip_circular_watch.html:797).
    ```javascript
    // Example for '0' and ':' in masterDigitPatterns
    '0': [[0,1,1,1,0],[1,1,0,1,1],...], // 7 rows, 5 columns
    ':': [[0,0,0,0],[0,1,1,0],...]      // 7 rows, 4 columns
    ```
*   **Scaled Patterns:** The [`initClockDigitsData()`](18-flip_circular_watch.html:797) function scales these master patterns to the target dimensions specified in [`scene.clockConfig`](18-flip_circular_watch.html:783) and stores them in [`scene.clockDigitsData`](18-flip_circular_watch.html:781). This scaling uses nearest-neighbor-like sampling.

### 3.4. Time Display on Grid (`updateTimeDisplayOnGrid()`)

The [`updateTimeDisplayOnGrid()`](18-flip_circular_watch.html:855) function is responsible for rendering the current time onto the simulation grid.
1.  **Clear Previous Time:** It iterates through [`scene.timeCellCoords`](18-flip_circular_watch.html:782) (an array storing `[ix, iy]` of current time cells) and resets these cells to `AIR_CELL` with `s[idx] = 1.0`, provided they are within the main circular boundary.
2.  **Get Current Time:** Fetches the current hours and minutes and formats them as "HH:MM".
3.  **Draw New Time:**
    *   Iterates through each character of the time string.
    *   Retrieves the corresponding scaled pattern from [`scene.clockDigitsData`](18-flip_circular_watch.html:781).
    *   For each 'active' pixel (value 1) in the pattern:
        *   Calculates the corresponding grid cell coordinates (`ix`, `iy`) based on [`scene.clockConfig.offsetX/offsetY`](18-flip_circular_watch.html:788-789) and character dimensions.
        *   Checks if the cell is within a slightly inset main circular boundary.
        *   If so, sets `fluid.cellType[idx]` to `TIME_CELL` (for digits) or `VIRTUAL_TIME_CELL` (for colon).
        *   Sets `fluid.s[idx]` to `0.0` (solid) for `TIME_CELL` or `1.0` (passable) for `VIRTUAL_TIME_CELL`.
        *   Adds the `[ix, iy]` coordinates to [`scene.timeCellCoords`](18-flip_circular_watch.html:782).

### 3.5. Dynamic Sizing and Positioning in `setupScene()`

The [`setupScene()`](18-flip_circular_watch.html:931) function includes new logic to:
1.  Calculate `targetCharHeightInGridCells` (e.g., 35% of circle diameter in grid cells).
2.  Calculate `targetTotalTimeWidthInGridCells` (e.g., 80% of circle diameter for "HH:MM").
3.  Determine `idealDigitWidth`, `idealColonWidth`, and `idealSpacing` based on master pattern aspect ratios and the target height.
4.  Scale these ideal widths using a `widthScaleFactor` to fit within `targetTotalTimeWidthInGridCells`.
5.  Store these calculated dimensions in [`scene.clockConfig`](18-flip_circular_watch.html:783).
6.  Call [`initClockDigitsData()`](18-flip_circular_watch.html:797) to generate scaled patterns.
7.  Calculate `scene.clockConfig.offsetX` and `scene.clockConfig.offsetY` to center the clock display within the simulation grid, relative to the main circle's center.

## 4. Integration with Existing Simulation

The introduction of time cells requires modifications to several core simulation functions:

### 4.1. `isCellStaticWall()` Modification

The [`isCellStaticWall(ix, iy, fluidObj, sceneObj)`](18-flip_circular_watch.html:102) function is updated:
*   It now checks if `fluidObj.cellType[ix * N + iy]` is `TIME_CELL`. If so, it returns `true` (it's a static wall).
*   `VIRTUAL_TIME_CELL`s are *not* considered static walls by this function, allowing fluid to pass through them as if they were air cells for collision purposes.

### 4.2. `transferVelocities(toGrid, flipRatio)` Modification

The [`FlipFluid.transferVelocities()`](18-flip_circular_watch.html:418) method is updated:
*   **Cell Type Determination (when `toGrid` is true):**
    *   If `this.cellType[i]` is `TIME_CELL`, `this.s[i]` is set to `0.0`.
    *   If `this.cellType[i]` is `VIRTUAL_TIME_CELL`, `this.s[i]` is set to `1.0`.
    *   Otherwise, `this.cellType[i]` is determined based on `this.s[i]` (SOLID or AIR).
    *   When particles mark cells as `FLUID_CELL`, this only happens if the cell was previously `AIR_CELL`. `TIME_CELL` and `VIRTUAL_TIME_CELL` types are preserved.
*   **Restore Solid Cell Velocities (when `toGrid` is true):**
    *   The logic for restoring velocities of solid cells now considers `TIME_CELL`s as solid, in addition to `SOLID_CELL`s. `VIRTUAL_TIME_CELL`s are not treated as solid here.

### 4.3. `updateCellColors()` Modification

The [`FlipFluid.updateCellColors()`](18-flip_circular_watch.html:709) method is updated:
*   Assigns a distinct color (e.g., light gray: `[0.9, 0.9, 0.9]`) to both `TIME_CELL` and `VIRTUAL_TIME_CELL` for visual differentiation on the grid display.

### 4.4. `setObstacle(x, y, reset)` and `endDrag()` Modifications

The [`setObstacle()`](18-flip_circular_watch.html:1356) and [`endDrag()`](18-flip_circular_watch.html:1464) functions, which handle the draggable mouse obstacle, are updated:
*   Before processing a cell for the draggable obstacle or resetting it, they first check if the cell is a `TIME_CELL` or `VIRTUAL_TIME_CELL`.
*   If it is, its `s` value is set according to its type (`0.0` for `TIME_CELL`, `1.0` for `VIRTUAL_TIME_CELL`), and the function continues to the next cell, effectively preserving the clock display from being overwritten by the draggable obstacle logic.

### 4.5. Simulation Update Loop (`update()`)

The main [`update()`](18-flip_circular_watch.html:1666) function is modified:
*   It checks the current time (minute and second) in each frame.
*   If the current minute is different from `scene.lastMinuteDisplayed` and the current second is `0`, it calls [`updateTimeDisplayOnGrid()`](18-flip_circular_watch.html:855) to refresh the clock and updates `scene.lastMinuteDisplayed`.
*   `scene.lastMinuteDisplayed` is initialized in the main script body after [`setupScene()`](18-flip_circular_watch.html:931).

## 5. UI and Interaction

### 5.1. Grid Resolution Slider

A new HTML range slider is added to control the grid resolution:
```html
<input type="range" min="50" max="200" value="100" step="10" class="slider" id="gridResSlider" onchange="updateGridResolution(this.value)"> <span id="gridResValue">100</span>
```
*   The `onchange` event calls [`updateGridResolution(this.value)`](18-flip_circular_watch.html:1685).
*   [`updateGridResolution()`](18-flip_circular_watch.html:1685) parses the new resolution, updates `scene.currentRes`, updates the displayed value, and calls [`restartSimulation()`](18-flip_circular_watch.html:1691).

### 5.2. Simulation Restart (`restartSimulation()`)

The [`restartSimulation()`](18-flip_circular_watch.html:1691) function handles resetting the simulation when parameters like grid resolution change:
1.  Deletes and nullifies WebGL buffers that depend on grid size (`gridVertBuffer`, `gridColorBuffer`, `pointVertexBuffer`, `pointColorBuffer`).
2.  Sets `scene.fluid = null` to release the old fluid object.
3.  Resets `scene.frameNr = 0`.
4.  Calls [`setupScene()`](18-flip_circular_watch.html:931), which re-initializes the entire simulation using the new `scene.currentRes`. This includes recalculating clock dimensions and performing the initial time display.

## 6. Key Data Structures (Summary)

*   **`scene.clockConfig`**: Stores calculated dimensions (digitWidth, digitHeight, colonWidth, spacing) and positioning (offsetX, offsetY) for the clock display.
*   **`scene.clockDigitsData`**: An object mapping characters ('0'-'9', ':') to their scaled 2D grid patterns.
*   **`scene.timeCellCoords`**: An array storing `[ix, iy]` grid coordinates of cells currently occupied by the clock display, used for efficient clearing.
*   **`scene.lastMinuteDisplayed`**: Stores the last minute for which the clock was updated, to trigger updates only once per minute.
*   **`scene.currentRes`**: Stores the current grid resolution selected by the user.
*   **`FlipFluid.cellType[]`**: Integer array now includes `TIME_CELL` (3) and `VIRTUAL_TIME_CELL` (4).
*   **`FlipFluid.s[]`**: Float array where `s[idx] = 0.0` for `TIME_CELL`s and `1.0` for `VIRTUAL_TIME_CELL`s (among other cell types).

## 7. Summary of Changes (Diff Highlights from `orig` to `watch` version)

*   **HTML (`18-flip_circular_watch.html` vs `18-flip_circular_orig.html`):**
    *   Added Grid Resolution slider input and corresponding span for value display (around line 65-67).
    *   `showGrid` checkbox is checked by default in the watch version.
*   **JavaScript:**
    *   **Global Scope:**
        *   New constants: [`TIME_CELL`](18-flip_circular_watch.html:89), [`VIRTUAL_TIME_CELL`](18-flip_circular_watch.html:90).
    *   **`scene` Object:**
        *   Added: [`clockDigitsData`](18-flip_circular_watch.html:781), [`timeCellCoords`](18-flip_circular_watch.html:782), [`clockConfig`](18-flip_circular_watch.html:783), [`lastMinuteDisplayed`](18-flip_circular_watch.html:791), [`currentRes`](18-flip_circular_watch.html:792).
        *   `showGrid` default changed to `true`.
    *   **New Functions:**
        *   [`initClockDigitsData()`](18-flip_circular_watch.html:797)
        *   [`updateTimeDisplayOnGrid()`](18-flip_circular_watch.html:855)
        *   [`updateGridResolution()`](18-flip_circular_watch.html:1685)
        *   [`restartSimulation()`](18-flip_circular_watch.html:1691)
    *   **Modified Functions:**
        *   [`isCellStaticWall()`](18-flip_circular_watch.html:102): Added checks for `TIME_CELL`.
        *   [`FlipFluid.transferVelocities()`](18-flip_circular_watch.html:418): Handles `TIME_CELL` and `VIRTUAL_TIME_CELL` for `s` array and cell type preservation; considers `TIME_CELL` in restoring solid cell velocities.
        *   [`FlipFluid.updateCellColors()`](18-flip_circular_watch.html:709): Added coloring for `TIME_CELL` and `VIRTUAL_TIME_CELL`.
        *   [`setupScene()`](18-flip_circular_watch.html:931): Major additions for dynamic clock configuration, particle fill height adjusted, calls `updateTimeDisplayOnGrid()`.
        *   [`setObstacle()`](18-flip_circular_watch.html:1356): Added logic to preserve time cells.
        *   [`endDrag()`](18-flip_circular_watch.html:1464): Added logic to preserve time cells.
        *   [`update()`](18-flip_circular_watch.html:1666): Added logic to call `updateTimeDisplayOnGrid()` periodically.
        *   Initial script execution: Initializes `scene.lastMinuteDisplayed` and calls `updateGridResolution` related functions.
        *   Particle fill region in [`setupScene()`](18-flip_circular_watch.html:931) adjusted (e.g. `fillMaxY` changed from `circleCenterY - circleRadius + (2.0 * circleRadius / 3.0)` to `circleCenterY - (0.2 * circleRadius)`).