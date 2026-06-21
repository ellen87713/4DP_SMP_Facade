# 4D-Printed Shape Memory Polymer for Responsive Facade
This repository contains the parametric Grasshopper definition and Rhino model developed for the MSc thesis:

**"4D-Printed Shape Memory Polymers for Responsive Facades"**
Jiang Yu-Ai (Ellen) — TU Delft, 2026
Supervisors: Dr. Serdar Așut (Digital Technologies), Ir. Eric van den Ham (Environmental & Climate Design)

The tool links **experimentally-derived SMP material behavior** (infill ratio, thickness, infill pattern → curvature) to **parametric façade geometry generation**, **fabrication-ready G-code output**, and **environmental performance simulation** (CFD ventilation + Radiance daylighting) within a single Grasshopper workflow.

> 📄 Full methodology, experiments, and results are documented in the accompanying thesis PDF. This README only covers software setup and usage.

---

## Repository Contents

| File | Description |
|---|---|
| `4DP_SMP_Rhino.3dm` | Rhino model file. Contains reference geometry, baseline curves/surfaces, and the test office room used for CFD/Radiance simulation (Section 5.1). |
| `4DP_SMP_GHfile.gh` | Main Grasshopper definition. Contains all 5 modules described below (Material Behavior, Unit Design, Façade System, Performance Evaluation, Fabrication). |

---

## 1. Requirements

### 1.1 Software

| Software | Version | Notes |
|---|---|---|
| **Rhino** | Rhino 8 | Required. Grasshopper is bundled with Rhino 8 (no separate install needed). |
| **Grasshopper** | Built into Rhino 8 | — |

### 1.2 Grasshopper Plugins

Install the following via Rhino's **Package Manager** (`Rhino > Tools > Package Manager`, or type `PackageManager` in the command line):

| Plugin | Purpose in this file | Install via |
|---|---|---|
| **Ladybug** | Climate data import (EPW), sun-path analysis, environmental metrics | Package Manager → search `ladybug` |
| **Honeybee** | Builds optical/radiance models from façade geometry, daylight & glare simulation (sDA, ASE, DGP) | Package Manager → search `honeybee` |
| **Butterfly** | Interfaces Grasshopper with OpenFOAM for CFD natural ventilation simulation (airflow velocity, ACH) | Package Manager → search `butterfly` |

> Ladybug Tools (Ladybug + Honeybee + Butterfly) are normally installed together as a suite. See [https://www.ladybug.tools](https://www.ladybug.tools) for the official installer and version compatibility notes.

### 1.3 External Engines (required by the plugins above, not Grasshopper plugins themselves)

| Engine | Used by | Notes |
|---|---|---|
| **Radiance** | Honeybee | Daylight/glare ray-tracing engine. Installed automatically by the Ladybug Tools installer, or install separately from [https://www.radiance-online.org](https://www.radiance-online.org). |
| **OpenFOAM** | Butterfly | CFD solver used for ventilation simulation (`blockMesh`, `snappyHexMesh`, solver execution). On Windows, this typically runs through a Docker container or the Blue CFD / OpenFOAM-Windows distribution — see the [Butterfly documentation](https://github.com/ladybug-tools/butterfly) for the supported setup on your OS. |

### 1.4 Optional (for fabrication / G-code preview only)

| Software | Purpose |
|---|---|
| **Bambu Studio** | Used only to preview/verify the exported G-code (travel path, print quality) before sending it to a Bambu Lab X1-Carbon printer. Not required to run the Grasshopper file itself — G-code is generated directly from Module 05 without external slicing software. |

---

## 2. Installation Steps

1. Install **Rhino 8** (trial or licensed) — [https://www.rhino3d.com/download](https://www.rhino3d.com/download)
2. Open Rhino 8 → launch **Grasshopper**.
3. Open `PackageManager` from the Rhino command line.
4. Search and install: `Ladybug Tools` (this installs Ladybug, Honeybee, and Butterfly together in recent releases — if not, install each individually).
5. Restart Rhino after installation.
6. On first run of any Ladybug/Honeybee/Butterfly component, you may be prompted to confirm installation of the underlying Python environment / Radiance / OpenFOAM dependencies — follow the plugin's own setup prompts.
7. Clone or download this repository.

```bash
git clone https://github.com/ellen87713/4DP_SMP_Facade.git
```

---

## 3. Opening the File

1. Open `4DP_SMP_Rhino.3dm` in Rhino 8 first (this loads the reference geometry/units that the GH file expects).
2. From within that Rhino session, open `4DP_SMP_GHfile.gh` in Grasshopper (`File > Open`, or drag-and-drop into the Grasshopper canvas).
3. Grasshopper components referencing Rhino geometry (curves, surfaces, the test room volume) will automatically re-link to the geometry in the open `.3dm` file. If any component shows a red error icon, right-click it → confirm it is still referencing the correct Rhino layer/object.

> ⚠️ Keep both files in the same relative location if the GH file uses any relative file paths (e.g. for G-code export or EPW climate files — see Section 6).

---

## 4. Workflow Structure (Module Overview)

The Grasshopper file is organized into 5 modules, matching the CAD tool framework in the thesis (Section 4.3, Figure 42):

```
01 — Material Printing Behavior   →  Material lookup table (infill ratio, thickness, pattern → curvature)
02 — Unit Design                  →  Pathway A (parametric) + Pathway B (inverse/freeform analysis)
03 — Façade System Design         →  Tiling, panel size, louver tilt angle aggregation
04 — Performance Evaluation       →  Butterfly (CFD) + Honeybee/Ladybug (daylight, glare)
05 — Fabrication                  →  Toolpath planning + G-code generator
```

Each module is grouped/labeled on the canvas — use Grasshopper's **scribble/group panels** to navigate, or use `Display > Draw Fancy Wires` off for a clearer view of large clusters.

### Module 01 — Material Behavior (Inverse Lookup)
- Input: target curvature (κ*) and desired bending-axis direction.
- Output: recommended `thickness`, `infill ratio`, and `infill pattern direction`, interpolated from the experimental dataset (Chapter 3, Table 5).
- A warning flag is triggered if infill ratio drops below 55% (extreme deformation / structural instability zone).

### Module 02 — Unit Design
- **Pathway A (forward/parametric):** Define a flat polygon panel → set cut pattern, fold axis, target curvature per segment, anchor position → generates both the as-printed flat state and the thermally-activated curved state.
- **Pathway B (inverse/freeform):** Input an arbitrary developable curved surface (your target façade shape) → the definition segments it by curvature continuity, reconstructs the rolling axis, and flattens it back into a fabrication-ready 2D panel.

### Module 03 — Façade System Design
- Aggregates individual panels/louvers into an array.
- Adjustable parameters: panel size, tiling arrangement, louver tilt angle (see Table 7 in thesis).

### Module 04 — Performance Evaluation
- **Ventilation (Butterfly → OpenFOAM):** Requires OpenFOAM correctly installed/linked. Generates a CFD case from the façade geometry as an aerodynamic boundary, runs `blockMesh` → `snappyHexMesh` → solver, and returns airflow velocity / ACH at probe points.
- **Daylighting (Honeybee → Radiance):** Converts façade geometry into Honeybee surfaces with the translucent SMP material modifier (diffuse reflectance 0.40 / transmittance 0.30), runs point-in-time or annual daylight recipes, and returns sDA, ASE, and DGP.
- You will need an **EPW climate file** for your target location (e.g. Delft or Taipei) — download from [https://www.ladybug.tools/epwmap](https://www.ladybug.tools/epwmap) and point the Ladybug "Import EPW" component to its local path.

### Module 05 — Fabrication
- Converts the curved-surface geometry into a multi-zone toolpath (slicing, zone-based orientation, infill path generation, zig-zag sorting).
- Outputs raw G-code (`G1` movement + extrusion commands) directly — no external slicer required.
- Optionally preview the resulting G-code in **Bambu Studio** before sending to print.

---

## 5. Quick Start

1. Open both files (Section 3).
2. In Module 01, set a target curvature value to see the recommended fabrication parameters.
3. In Module 02 (Pathway A), toggle the polygon cut pattern / fold axis sliders to preview the as-printed vs. activated shape live in the Rhino viewport.
4. In Module 03, array the unit across a façade grid and adjust tiling/tilt sliders.
5. (Optional, requires OpenFOAM + Radiance set up) Run Module 04 to evaluate ventilation and daylighting for your configuration.
6. In Module 05, bake/export the G-code output to a `.gcode` file and preview it in Bambu Studio if desired.

---

## 6. Known Dependencies / Things to Check Before Running

- [ ] Rhino 8 installed and licensed
- [ ] Ladybug, Honeybee, Butterfly all show as "Loaded" in Grasshopper's plugin list (`File > Special Folders > check installed plugins`, or look for them in the Grasshopper toolbar tabs)
- [ ] Radiance is on your system PATH (Honeybee will warn on first run if it can't find it)
- [ ] OpenFOAM is installed and accessible to Butterfly (Windows users: check the Butterfly docs for the recommended Docker/BlueCFD setup)
- [ ] An EPW file for your climate of interest is downloaded locally (only needed for Module 04)
- [ ] Both `4DP_SMP_Rhino.3dm` and `4DP_SMP_GHfile.gh` are kept together / paths re-linked on first open

---

## 7. Citation

If you use or build on this tool, please cite:

> Jiang, Y.-A. (2026). *4D-Printed Shape Memory Polymers for Responsive Facades* [Master's thesis]. TU Delft.

---

## 8. AI Tool Disclosure

Parts of this codebase were co-developed with the assistance of Claude (computational logic prototyping, later reconstructed and implemented in Grasshopper by the author) and Gemini (G-code generator scripting in Python/Grasshopper). All final integration, logic, and outputs were independently engineered by the author. See the thesis Acknowledgement section for full details.

---

## 9. Contact / Issues

For questions about this repository, please open an issue on GitHub or contact the author via TU Delft.
