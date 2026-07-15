# Enabling Flexibility in PLC-Controlled Robots Assembly Sequence Generation

**Geometry-driven Assembly Sequence Planning from STEP CAD Models — a bridge between digital product design and executable manufacturing/PLC control knowledge**

Master's Thesis · M.Sc. Mechatronics · University of Siegen · Chair of Interconnected Automation Systems (Prof. Dr.-Ing. Oliver Wallscheid), in cooperation with Fraunhofer IGCV, Augsburg

- **Author:** Monish Senthil Kumar
- **First Supervisor:** Dr.-Ing. Thomas Schulte (University of Siegen)
- **Second Supervisor:** Andreas Gaugenrieder, M.Eng. (Fraunhofer IGCV)
- **Submission Date:** 24.03.2026

---

## 1. Project Summary

Modern production lines — especially in battery module manufacturing — face constant product design changes. In classical industrial practice, an engineer manually reads a CAD model, decides the assembly order, and then hand-codes that order into robot programs and **PLC/robot control sequences**. This is slow and does not scale when the product changes every few weeks.

This project implements a **computational pipeline that reads a raw STEP CAD assembly and automatically derives a feasible assembly/disassembly sequence**, without any manually created assembly graph, liaison matrix, or semantic labeling. The output is a structured, machine-readable **process plan (JSON)** that is directly usable as the specification layer for a robot/PLC control program — i.e. it produces exactly the kind of ordered task/action list that a PLC or robot controller program (e.g. Siemens TIA Portal SCL/SFC, or a robot task sequencer) would otherwise be built from by hand.

**Core question:** *Can assembly knowledge be extracted directly from CAD geometry, and turned into a sequence that a real automated production cell (conveyor, linear robot, cobot) can execute?*

---

## 2. Why This Matters for Automation / PLC Engineering

Although the thesis is framed as a CAD/robotics research project, the deliverable is directly relevant to control and automation engineering work:

| Thesis output | Automation engineering equivalent |
|---|---|
| Dependency graph (precedence constraints) | Interlocking / sequencing logic in a PLC step chain (Sequential Function Chart) |
| Generated assembly sequence (`PLACE`, `INSERT`, `ROTATE`) | Ordered list of robot/PLC program steps |
| JSON process plan with tasks → sub-tasks → actions | Structure equivalent to a PLC task list / HMI work-instruction table |
| Contact & dependency graph from geometry | Equivalent of manually engineered I/O interlock and precedence logic |
| Multi-station system (XTS conveyor, Festo linear axis, UR5, SCARA) | Typical modular automation cell of the kind controlled by S7-1500 / TIA Portal, WinCC, and Profinet, matching my hands-on SPS/PLC background |

In short: this project automates the **planning step that normally precedes PLC/robot programming**, and produces an output format designed to plug into a robot/PLC execution layer — the exact interface point between design engineering and shop-floor automation that a junior SPS/PLC programmer works at.

---

## 3. System Architecture

The target physical system used for the case study is a modular robotic battery-module assembly cell consisting of:

- **XTS conveyor transport system** — moves modules/carriers between stations (Profinet-style modular transport)
- **Festo linear axis robot** — multi-axis assembly/disassembly station
- **UR5 collaborative robot** — places battery cells into the module housing
- **SCARA robot (KUKA)** — pick-and-place between conveyor and stations
- **Future station** — automated screw tightening / locking-ball insertion

```
CAD STEP Model
      │
      ▼
Geometry Extraction  (solids, bounding boxes, volumes, centroids)
      │
      ▼
Contact Relationship Detection  (intersection-based contact graph)
      │
      ▼
Fastener Identification  (volume, aspect ratio, contact-count heuristics)
      │
      ▼
Dependency Graph Creation  (precedence / support constraints)
      │
      ▼
Accessibility & Collision Check  (feasibility of removal/insertion direction)
      │
      ▼
Assembly Sequence Planner  (iterative constraint resolution)
      │
      ▼
Final Assembly Plan  →  Structured Process Plan (JSON)
```

---

## 4. Methodology

### 4.1 Geometric Modeling
Each solid component `p_i` extracted from the STEP model is represented as:

```
p_i = (C_i, B_i, V_i)
```
- `C_i` – centroid (geometric center)
- `B_i` – bounding box dimensions (X, Y, Z)
- `V_i` – volume

### 4.2 Contact Graph
Two components are considered in contact if their intersection volume exceeds a small tolerance `ε`:

```
Volume(p_i ∩ p_j) > ε
```

This produces a graph `G = (P, E)` used as the structural backbone for all further reasoning.

### 4.3 Functional Role / Fastener Identification
Components are heuristically classified as fasteners (screws, locking balls) if they satisfy **all** of:
- volume significantly below the assembly's median component volume
- high aspect ratio (elongated shape, bounding-box based)
- multiple simultaneous contact relationships with neighboring components

### 4.4 Dependency & Support Reasoning
- **Fastener dependency:** a fastener must be removed before the structural parts it connects can be separated.
- **Support dependency:** a component below another along the dominant stacking axis, with overlapping horizontal projection, is treated as a structural support and cannot be removed before the component it supports.
- **Dominant assembly axis:** determined automatically as the axis (X/Y/Z) with the largest cumulative component span — this reflects the typical vertical-insertion nature of battery module assembly.

### 4.5 Accessibility & Collision Feasibility
For each candidate component with resolved dependencies, the framework simulates translational motion along the assembly axis and checks for geometric intersection with neighboring parts (collision detection) before accepting it into the sequence.

### 4.6 Sequence & Process Plan Generation
The planner iterates: at each step, select all components with zero unresolved dependencies, not currently acting as support, and geometrically accessible → append to sequence → remove from model → update dependency graph → repeat until empty.

The disassembly order is generated first, then reversed to obtain the assembly order. The final sequence is exported as a structured **JSON process plan** (Tasks → Sub-tasks → Actions), directly consumable by a downstream robot/PLC execution layer.

---

## 5. Implementation

| Component | Technology |
|---|---|
| CAD interface & geometry kernel | **FreeCAD** (Python API) |
| Algorithm implementation | **Python** |
| Input format | **STEP** (ISO 10303 / AP242) |
| Output format | **JSON** process plan |
| Development/target hardware for case study | Festo Linear Axis Robot, XTS Transport System, UR5, KUKA-SCARA |



## 6. Case Study: Battery Module Assembly

**Product:** modular battery pack (Ground / Middle / Lid structural plates + 20 cylindrical battery cells + 4 screw fasteners), available in a screwed and a locking-ball variant.

| Category | Parts |
|---|---|
| Bottom (Ground) | `Part_Feature` |
| Middle | `Part_Feature001` |
| Top (Lid) | `Part_Feature002` |
| Battery Cells | `Part_Feature003` – `Part_Feature022` |
| Screws (Fasteners) | `Part_Feature023` – `Part_Feature026` |

**Pipeline output (abridged):**
1. 43 objects imported → 27 valid solid parts identified
2. Volumes computed for all parts (structural plates ≈ 150,000–255,000 mm³; screws ≈ 994 mm³)
3. Contact graph built from pairwise intersection tests
4. 4 fasteners automatically detected purely from geometry
5. Dependency graph generated (assembly + disassembly precedence)
6. 30-step assembly sequence generated: `ROTATE → PLACE → INSERT → PLACE → …`
7. Sequence exported as a hierarchical JSON process plan (Tasks → Sub-tasks → Robot Actions), ready to drive station-level execution (e.g. XTS transport request → SCARA/UR5/Festo pick-place → screw tightening)

---

## 7. Research Questions Addressed

| # | Question | Finding |
|---|---|---|
| RQ1 | What are the requirements for automated workflow transfer in modular assembly systems? | Flexibility, reconfigurability, scalability, and interoperability (via standardized STEP input) are the four enabling capabilities identified. |
| RQ2 | Can functional components (e.g. fasteners) be identified from geometry alone? | Yes, with heuristics (low volume, high aspect ratio, high contact count) — with known limitations for atypical/ambiguous geometries. |
| RQ3 | Can geometric reasoning + collision detection determine feasible removal directions? | Yes, for translational motion along a dominant axis; rotational/multi-axis removal is a limitation. |
| RQ4 | Does combining structural dependency reasoning with geometric feasibility checks improve planning? | Yes — each alone is insufficient (logically valid but physically infeasible, or vice versa); combined, the framework produces consistently feasible plans. |

---

## 8. Limitations

- Assumes rigid bodies (no tolerances, compliance, or deformation)
- Assumes a single dominant (vertical) assembly axis; no multi-axis/rotational disassembly reasoning
- Fastener/functional classification is heuristic, not semantic — can misclassify atypical geometries
- Validated on a representative case-study assembly, not yet on large industrial-scale product structures
- No physics-based (force/friction) feasibility simulation

## 9. Future Work

- Direct integration of the generated JSON process plan into robot/PLC motion execution (e.g. mapping tasks to a Siemens S7-1500/TIA Portal step-chain or robot task sequencer)
- Physics-based simulation (contact forces, friction, stability) for feasibility validation
- Multi-axis / rotational disassembly reasoning
- Machine-learning-based functional role classification to complement geometric heuristics
- Digital twin coupling for real-time, adaptive re-planning
- Scalability testing on larger, industrial-scale assemblies

---

## 10. Keywords

`PLC Programming` . `assembly sequence planning` · `CAD-based reasoning` · `STEP (ISO 10303)` · `FreeCAD` · `Python` · `collision detection` · `geometric reasoning` · `modular manufacturing systems` · `robotics` · `battery module assembly` · `digital manufacturing` · `automation engineering`

## 11. References

Full reference list and detailed methodology are available in [`docs/thesis.pdf`](docs/thesis.pdf).

## 12. Contact

**Monish Senthil Kumar**
M.Sc. Mechatronics, University of Siegen
📧 monish.skumar08@gmail.com · 🔗 [LinkedIn](https://linkedin.com/in/monish-senthil-kumar)
