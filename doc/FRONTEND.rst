======================================
FRONTEND DESIGN DOCUMENT
======================================

.. sectnum::
   :depth: 3

.. contents:: Table of Contents
   :depth: 3
   :local:

Overview
========

This document describes the architecture and implementation of ``script.js``,
which renders an interactive time-series chart for PostgreSQL performance
metrics. The chart allows users to filter by scale factor and branch, zoom
and pan the time axis, and inspect individual data points via tooltips.

Architecture
============

The application is organized into four layers that flow top-down:

.. code-block:: text

   ┌─────────────────────────────────────────┐
   │        Interaction Layer (UI Events)    │
   │  - Brushing, Tooltips, Keyboard, Clicks │
   └──────────────┬──────────────────────────┘
                  │
   ┌──────────────▼──────────────────────────┐
   │     Control & UI Layer (State Mgmt)     │
   │  - currentFilters state object          │
   │  - Event listeners & DOM manipulation   │
   └──────────────┬──────────────────────────┘
                  │
   ┌──────────────▼──────────────────────────┐
   │    Rendering Layer (D3 Visualization)   │
   │  - SVG generation, scales, axes         │
   │  - Data binding with D3 selections      │
   └──────────────┬──────────────────────────┘
                  │
   ┌──────────────▼──────────────────────────┐
   │        Data Layer (Transform)           │
   │  - Flatten nested RESULTS_DATA          │
   │  - Pre-calculate scale counts           │
   └─────────────────────────────────────────┘

Global State
============

The following module-level variables are shared across functions:

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - Variable
     - Type
     - Description
   * - ``allData``
     - ``Array<Object>``
     - Flattened dataset. Each entry is a single data point with fields
       ``branch``, ``scale``, ``revision``, ``ctime`` (Date), ``metric``,
       ``builder_id``, and ``build_number``.
   * - ``scaleDataCounts``
     - ``Map<number, number>``
     - Maps each scale value to its data point count, used to label the
       dropdown options.
   * - ``currentFilters``
     - ``Object``
     - Holds the active filter state: ``{ scale: number|null, branches: string[] }``.
   * - ``DOMElements``
     - ``Object``
     - Cache of frequently accessed DOM/D3 element references, populated
       once during ``cacheDOMElements()`` to avoid repeated queries.
   * - ``currentChartData``
     - ``Array<Object>``
     - The subset of ``allData`` that is currently rendered in the chart,
       used by ``updateYAxis()``.
   * - ``xScale``, ``yScale``
     - D3 scales
     - Time and linear scales for the main chart. Mutated in place during
       zoom/pan operations.
   * - ``xAxis``, ``yAxis``, ``xGrid``, ``yGrid``, ``line``
     - D3 functions
     - Axis generators and line generator initialized in ``renderChart()``
       and reused in ``updateYAxis()`` and ``redrawChart()``.


Data Layer
==========

Source Data Format
------------------

``RESULTS_DATA`` is a nested object with the following shape:

.. code-block:: json

   {
     "100": {
       "REL_13_STABLE": {
         "reversed": [
           {
             "revision": "3850fcca...",
             "ctime": 1708386677,
             "metric": 564578.0,
             "builder_id": 5,
             "build_number": 42
           }
         ]
       }
     }
   }

The top-level keys are scale factors (e.g. ``"100"``). Under each scale are
branch names, each containing a ``reversed`` array of test result objects.

``transformChartData(nestedData)``
----------------------------------

Flattens the nested structure into a plain array for D3 consumption:

.. code-block:: text

   1. Initialize empty flatData array
   2. FOR each scale in nestedData:
   3.   FOR each branch in scale:
   4.     FOR each result in branch.reversed:
   5.       CREATE flat object {
               branch:       string,
               scale:        number  (parsed from string key),
               revision:     string,
               ctime:        Date    (converted from UNIX timestamp * 1000),
               metric:       number  (parsed),
               builder_id:   number,
               build_number: number
             }
   6.       PUSH to flatData
   7. RETURN flatData

Scale Count Pre-calculation
---------------------------

After flattening, the count of data points per scale is cached using
``d3.rollup()``:

.. code-block:: javascript

   const counts = d3.rollup(
     allData,
     (v) => v.length,   // aggregation: count items
     (d) => d.scale     // group by: scale value
   );
   scaleDataCounts = new Map(counts);

This map is used to display result counts in the scale dropdown, e.g.
``"100 (312 results)"``.

Control & UI Layer
==================

Filter State
------------

All user selections are stored in a single object:

.. code-block:: javascript

   const currentFilters = {
     scale: number | null,    // single selected scale
     branches: string[]       // array of selected branch names
   };

Any change to these values is followed by a call to
``applyFiltersAndRender()``, which re-derives and re-renders everything.

Scale Dropdown (``#chooseScale``)
---------------------------------

``setupFilterControls()`` populates the ``<select>`` element with sorted
scale values and sets the first scale as the default. On change:

.. code-block:: javascript

   scaleFilter.addEventListener('change', (event) => {
     currentFilters.scale = +event.target.value;
     updateBranchFilter(true);  // reset branch selection
     applyFiltersAndRender();
   });

Branch Filter Dropdown (``#branch-filter-dropdown``)
----------------------------------------------------

``updateBranchFilter(reset)`` rebuilds the checkbox list for the currently
selected scale. Branches are sorted and reversed for consistent ordering:

.. code-block:: javascript

   const availableBranches = [
     ...new Set(
       allData
         .filter(d => d.scale === currentFilters.scale)
         .map(d => d.branch)
     )
   ].sort().reverse();

A single delegated event listener on the container handles all checkbox
changes, avoiding the overhead of per-checkbox listeners:

.. code-block:: javascript

   // Efficient: one listener handles all checkboxes
   branchListContainer.addEventListener('change', (event) => {
     if (event.target.type === 'checkbox') {
       currentFilters.branches = Array.from(
         branchListContainer.querySelectorAll('input:checked')
       ).map(cb => cb.value);
       updateBranchButtonText();
       applyFiltersAndRender();
     }
   });

The "Select All" and "Deselect All" buttons call ``toggleAllBranches(bool)``,
which sets all checkboxes programmatically and updates the filter state.

Y-Axis Mode Radio Buttons
-------------------------

Two modes are available: ``zero`` (Y-axis starts at 0) and ``zoom``
(Y-axis zooms to the visible data range). Changing the radio button does
**not** trigger a full re-render; it only calls ``updateYAxis()``:

.. code-block:: javascript

   document.querySelectorAll('input[name="y_axis_mode"]').forEach(radio => {
     radio.addEventListener('change', () => {
       updateYAxis(currentChartData);
     });
   });

``applyFiltersAndRender()``
---------------------------

The central function that connects filters to rendering:

.. code-block:: text

   1. Filter allData by currentFilters.scale        → dataForScale
   2. Extract sorted unique branches for the scale  → allBranchesForScale
   3. Create d3.scaleOrdinal (Tableau10) with these branches as domain
   4. Filter dataForScale by currentFilters.branches → filteredData
   5. renderChart(filteredData, colorScale)
   6. renderLegend(allBranchesForScale, colorScale)

Filtering complexity is O(n) for scale filtering and O(m) for branch
filtering where m = |dataForScale|.

Rendering Layer
===============

SVG Structure
-------------

``renderChart()`` always performs a full SVG teardown before re-drawing
(``svg.selectAll('*').remove()``). The resulting SVG hierarchy is:

.. code-block:: text

   <svg id="chart-svg">
     ├── <defs>
     │     └── <clipPath id="chart-clip">
     │           └── <rect>                        (clip boundary)
     │
     ├── <g class="main-chart">                    [Layer 1: Main Chart]
     │     ├── <g class="grid x-grid">             (background grid)
     │     ├── <g class="grid y-grid">
     │     ├── <g class="axis x-axis-group">       (axes on top of grid)
     │     ├── <g class="axis y-axis-group">
     │     │
     │     └── <g clip-path="url(#chart-clip)">   [Layer 2: Clipped Content]
     │           ├── <path class="line">           (data lines — behind circles)
     │           ├── <circle class="data-circle"> (data points — interactive)
     │           │
     │           └── <g class="brush xaxis-brush"> [Layer 3: Main Brush]
     │                 └── <rect class="overlay"> (transparent interaction zone)
     │
     └── <g class="context">                       [Layer 4: Context Chart]
           ├── <path class="context-line">         (mini timeline lines)
           ├── <g class="axis">                    (context x-axis)
           └── <g class="brush context-brush">     (context brush)
                 └── <rect class="overlay">

.. note::
   After the brush overlay is appended, ``branchCircles.raise()`` is called
   to move the circles above the brush layer. This is critical: without it,
   the brush overlay intercepts mouse events and tooltips stop working.

``renderChart(data, colorScale)`` — Step-by-Step
------------------------------------------------

.. code-block:: text

   1.  Clear previous SVG              svg.selectAll('*').remove()
   2.  Validate data                   if length === 0 → display "No data" message
   3.  Calculate dimensions            mainChartHeight=400, contextChartHeight=80,
                                       margins, chartWidth from SVG bounding rect
   4.  Initialize D3 scales
         xScale  = d3.scaleTime()   domain: [earliest ctime, latest ctime]
         yScale  = d3.scaleLinear() domain: [0, max(metric)*1.05]
         xScale2 = d3.scaleTime()   same domain as xScale (for context chart)
         yScale2 = d3.scaleLinear() same domain as yScale (for context chart)
   5.  Adjust Y-scale for mode        if mode==="zoom": domain=[min*0.95, max*1.05]
   6.  Create SVG groups              main chart group, context group (offset below)
   7.  Draw grid lines                x-grid (vertical), y-grid (horizontal)
   8.  Draw axes                      x-axis (bottom), y-axis (left)
   9.  Draw data lines
         group data by branch using d3.group()
         create line generator:
           line.x(d => xScale(d.ctime))
           line.y(d => yScale(d.metric))
         bind groupedData to <path> elements via .data().join()
   10. Draw context chart             mini lines + context x-axis
   11. Attach brushes                 main chart brush, context brush
   12. Draw data circles              one <circle> per data point
   13. Raise circles above brush      branchCircles.raise()
   14. Attach event handlers
         tooltip (mouseover/mouseout on circles)
         double-click on clipArea → resetZoom()
         keyboard (mouseover/mouseout on svg → add/remove keydown listener)

D3 Scale Details
----------------

.. code-block:: javascript

   xScale = d3.scaleTime()
     .domain(d3.extent(data, d => d.ctime))  // [earliest, latest]
     .range([0, chartWidth]);                 // [left pixel, right pixel]

   yScale = d3.scaleLinear()
     .domain([0, d3.max(data, d => d.metric) * 1.05])
     .range([chartHeight, 0]);               // inverted: bottom=0, top=max

Line Generator
--------------

.. code-block:: javascript

   line = d3.line()
     .x(d => xScale(d.ctime))    // date → x pixel
     .y(d => yScale(d.metric));  // metric → y pixel

Lines are bound via:

.. code-block:: javascript

   clipArea.selectAll('.line')
     .data(groupedData)
     .join('path')  // handles enter, update, exit automatically
     .attr('stroke', ([branch]) => colorScale(branch))
     .attr('d', ([, values]) => line(values.sort((a, b) => a.ctime - b.ctime)));

Context (Overview) Chart
------------------------

A smaller chart is rendered below the main chart at
``y = mainChartHeight + margin.top``. It uses ``xScale2`` and ``yScale2``
(fixed domains, never mutated) and serves as a navigation reference for the
context brush.

Interaction Layer
=================

Brush Interactions
------------------

Two independent ``d3.brushX()`` instances are used:

**Main Chart Brush (``xAxisBrush``)** — overlaid on the main chart's clip area:

.. code-block:: text

   User drags on main chart
     └─> brushedXAxis(event)
           └─> [x0, x1] = event.selection.map(xScale.invert)
           └─> xScale.domain([x0, x1])
           └─> clear main brush visually
           └─> sync contextBrush to new domain
           └─> redrawChart(500ms)

**Context Brush (``contextBrush``)** — overlaid on the context chart:

.. code-block:: text

   User drags on context chart
     └─> brushedContext(event)
           └─> [x0, x1] = event.selection.map(xScale2.invert)
           └─> xScale.domain([x0, x1])
           └─> clear main brush
           └─> redrawChart(500ms)

``redrawChart(duration)``
-------------------------

Transitions chart elements to the updated ``xScale`` domain without a full
re-render:

.. code-block:: javascript

   function redrawChart(duration = 500) {
     const t = svg.transition().duration(duration).ease(d3.easeCubicInOut);
     xAxisGroup.transition(t).call(xAxis.scale(xScale));
     chart.select('.x-grid').transition(t).call(xGrid.scale(xScale));
     updateYAxis(currentChartData, duration);
   }

``updateYAxis(data, duration)``
-------------------------------

Recalculates the Y domain based on currently visible data points and the
selected mode, then transitions all dependent elements:

.. code-block:: text

   1. Get current X-domain [xMin, xMax] from xScale
   2. Filter data to visible range:
        visibleData = data.filter(d => d.ctime >= xMin && d.ctime <= xMax)
   3. Determine new Y-domain:
        IF mode === "zoom" AND visibleData.length > 0:
          yScale.domain([min*0.95, max*1.05])
        ELSE:
          yScale.domain([0, max*1.05])
   4. Transition: Y-axis, Y-grid, all .line paths, all .data-circle cy values

Keyboard Navigation
-------------------

Keyboard listeners are added when the mouse enters the SVG and removed when
it leaves, preventing unintended global key capture:

.. code-block:: text

   Escape       → resetZoom()
   ArrowLeft    → pan domain left by 10% of current time span
   ArrowRight   → pan domain right by 10% of current time span

Pan logic:

.. code-block:: javascript

   const timeSpan = x1.getTime() - x0.getTime();
   const shift = timeSpan * (±0.1);
   xScale.domain([new Date(x0 + shift), new Date(x1 + shift)]);
   contextBrushGroup.call(contextBrush.move, xScale.domain().map(xScale2));
   redrawChart(100);  // fast 100ms redraw for responsive feel

Reset Zoom
----------

Triggered by double-click on ``clipArea`` (not ``svg``) or the Escape key:

.. code-block:: javascript

   function resetZoom() {
     xScale.domain(xInitialDomain);
     contextBrushGroup.call(contextBrush.move, null);
     xAxisBrushGroup.call(xAxisBrush.move, null);
     redrawChart();
   }

Tooltip System
==============

``setupTooltip(circles, colorScale)``
-------------------------------------

On ``mouseover`` of a data circle:

1. The hovered circle is enlarged (radius 4 → 7) via transition.
2. Tooltip HTML is built containing: formatted date, branch color swatch,
   branch name, metric value, a link to the GitHub commit, and a link to
   the Buildbot test results.
3. Smart positioning is applied (see below).

On ``mouseout``, the circle shrinks back and tooltip hide is delayed 100ms
to allow the user's mouse to move onto the tooltip without it disappearing.

Smart Tooltip Positioning
-------------------------

Four candidate placements are evaluated in priority order:
``top → bottom → right → left``. The first placement whose bounding box
fits entirely within the container is selected:

.. code-block:: javascript

   const placements = [
     { name: 'top',    x: pointX - w/2, y: pointY - h - offset },
     { name: 'bottom', x: pointX - w/2, y: pointY + offset },
     { name: 'right',  x: pointX + offset, y: pointY - h/2 },
     { name: 'left',   x: pointX - w - offset, y: pointY - h/2 },
   ];

   let finalPlacement = placements.find(p =>
     p.x >= 0 && p.y >= 0 &&
     p.x + w <= containerRect.width &&
     p.y + h <= containerRect.height
   );

If no placement fits, the ``top`` position is used and clamped to the
container boundaries. A CSS arrow class (``arrow-down``, ``arrow-up``,
``arrow-left``, ``arrow-right``) is applied to match the chosen placement.

Legend
======

``renderLegend(branches, colorScale)``
--------------------------------------

Renders one legend item per branch. Items use D3's ``join()`` with explicit
enter, update, and exit selections. Clicking a legend item toggles the
corresponding branch checkbox and dispatches a ``change`` event to trigger
the normal filter update flow:

.. code-block:: javascript

   item.on('click', (event, d) => {
     const checkbox = branchListContainer.querySelector(`input[value="${d}"]`);
     if (checkbox) {
       checkbox.checked = !checkbox.checked;
       checkbox.dispatchEvent(new Event('change', { bubbles: true }));
     }
   });

The ``inactive`` CSS class is applied to branches that are not in
``currentFilters.branches``, visually dimming unselected items.

Utilities & Error Handling
==========================

``updateLastUpdated(data)``
---------------------------

Finds the maximum ``ctime`` in ``allData`` and writes a formatted date
string to the ``#lastUpdated`` element.

``handleError(error)``
----------------------

Logs the error to the console and renders a red error message centered in
the SVG, replacing any previous chart content.

User Action → Render Flow Summary
=================================

.. code-block:: text

   User Action             → State Change               → Re-render Path
   ─────────────────────────────────────────────────────────────────────
   Scale dropdown change   → currentFilters.scale       → full renderChart()
   Branch checkbox change  → currentFilters.branches    → full renderChart()
   Select All/Deselect All → currentFilters.branches    → full renderChart()
   Y-axis mode change      → (no filter change)         → updateYAxis() only
   Main chart brush        → xScale.domain mutated      → redrawChart()
   Context chart brush     → xScale.domain mutated      → redrawChart()
   Arrow key pan           → xScale.domain mutated      → redrawChart(100ms)
   Escape / double-click   → xScale.domain reset        → redrawChart()
   Circle hover            → (no state change)          → tooltip show
   Legend item click       → toggles branch checkbox    → (follows branch flow)

Performance Notes
=================

- **DOM caching**: ``DOMElements`` is populated once and reused everywhere,
  avoiding repeated ``getElementById`` / ``querySelector`` calls.
- **Delegated listeners**: A single listener on ``branchListContainer``
  handles all checkbox ``change`` events instead of one per checkbox.
- **Selective re-render**: Zoom and pan use ``redrawChart()`` (transition
  only), not a full ``renderChart()``. Only filter changes trigger the
  more expensive full teardown and rebuild.
- **Transition durations**: 100ms for keyboard pan (feels instant), 500ms
  for brush zoom (visible, smooth animation), 250ms for Y-axis mode switch.
- **``branchCircles.raise()``**: Called once after the brush overlay is
  appended to ensure circles stay on top without re-inserting DOM nodes
  on every frame.