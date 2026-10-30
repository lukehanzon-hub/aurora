# AURORA — Project Context for Claude Code

## Goal
Cross-platform rocket flight simulator in Rust that combines and extends the capability of
OpenRocket, RockSim, RASAero II, and openMotor:
- All phases of flight: rail departure, boost, coast, staging, apogee, recovery/descent
- Subsonic through upper supersonic aerodynamics (Mach 0 to ~5). Hypersonics are OUT OF SCOPE.
- Multistage vehicles
- Active control (canards / TVC / roll control) with software-in-the-loop, later hardware-in-the-loop
- Modern visualization (2D plots + 3D trajectory replay)
- Targets: macOS, Windows, Linux; later web (wasm), cloud, mobile
- Future: STL/3D CAD import, live simulation

## Status
- Repo state: new, empty private GitHub repo (blank slate). Owner: lukehanzon-hub.
- Naming (decided): AURORA = the flight simulator. ARCTAS = the owner's separate flight
  computer project. Do not mix ARCTAS firmware code into AURORA crates.
  ARCTAS is a future SIL/HIL integration target for aurora-gnc.
- Repo location (decided): AURORA gets its own new PRIVATE GitHub repo, separate from
  ARCTAS. Do not commit AURORA code to the ARCTAS repo.

## Licensing (decided: maximum owner flexibility)
- All rights reserved. No open-source license file. Do NOT add MIT, Apache, GPL,
  PolyForm, or any other LICENSE file.
- Add a short notice at repo root (NOTICE or COPYRIGHT file) and in Cargo.toml:
  `license-file = "COPYRIGHT"` and `publish = false` in every crate.
  COPYRIGHT text: "Copyright (c) 2026 Luke Hanzon. All rights reserved.
  No license is granted to use, copy, modify, or distribute this software
  without written permission."
- Keep the repo private until the owner chooses a license.
- No outside contributions without a Contributor License Agreement (CLA) that grants
  the owner relicensing rights. Flag any PR from someone else.

## Dependency and code-provenance rules (protect relicensing freedom)
- Allowed dependency licenses: MIT, Apache-2.0, BSD-2/3, ISC, Zlib, Unicode-DFS, CC0.
- Forbidden: GPL, LGPL, AGPL, MPL (unless owner approves), SSPL, any non-commercial license.
- Enforce with `cargo-deny` (deny.toml with the allowlist above) in CI from day one.
- OpenRocket and openMotor are GPL-licensed. Do NOT copy, translate, or port their source code.
  Implement every model from the published primary literature (Barrowman 1967,
  NACA/NASA reports, USSA76, DATCOM, Niskanen thesis as a method reference) and cite it.
  Reading/writing their FILE FORMATS (.ork, .ric) is fine; reimplement parsers from scratch.
- RASAero II is closed-source; use its published comparisons only as references.
- Record the literature source for each model in a doc comment.

## Flight envelope and accuracy philosophy
Engineering methods with explicit validity envelopes, not CFD.
- Supported envelope: Mach 0 to 5.0. Mach 5 is the hard upper limit.
- If a simulation exceeds Mach 5, emit an out-of-envelope warning and flag all downstream
  results as unvalidated. Do not silently extrapolate. No hypersonic methods
  (modified Newtonian, real-gas, etc.) are to be implemented.
- Every aero output carries its regime and an uncertainty band.
- External aero tables (RASAero exports, CFD, wind tunnel) can override built-in methods.
- Regimes:
  - Subsonic to ~M0.8: Barrowman normal force + component drag buildup
  - Transonic M0.8–1.2: empirical drag rise — weakest regime, expect ±10–20% Cd
  - Supersonic M1.2–5.0: second-order shock-expansion, Van Dyke / Syvertson body methods,
    supersonic fin (linear/Busemann) theory
- Aeroheating at M3–5 (fin leading edges, nose tip) is relevant but deferred: optional
  Eckert reference-temperature skin heating in a later phase.

## Proposed workspace layout (proposal — confirm before building)
| Crate | Responsibility |
|---|---|
| aurora-core | State, quaternions, frames (body/NED/ECEF/ECI), adaptive RK45, event detection |
| aurora-atmo | US Standard Atmosphere 1976 (to 86 km); NRLMSISE-00 later; wind, Dryden turbulence |
| aurora-aero | Component buildup, regime switching (M0–5), Cnα / Cp / Cd / damping derivatives, table import |
| aurora-prop | .eng/.rse thrust curves; openMotor-style internal ballistics (BATES, Finocyl, Saint-Robert, nozzle) |
| aurora-vehicle | Component tree, time-varying mass/CG/inertia, stage definitions and separation |
| aurora-gnc | Sensor models, estimator, controllers, actuator dynamics, SIL interface (HIL with ESP32 later) |
| aurora-thermal | (Later phase) supersonic skin heating, skin temperature history |
| aurora-recovery | Deployment events, drogue/main descent, inflation loads |
| aurora-sim | Orchestrator, Monte Carlo dispersions, landing ellipse |
| aurora-io | Import .ork (OpenRocket), .CDX1 (RASAero), .ric (openMotor); CSV/HDF5 export |
| aurora-app | egui/eframe UI + wgpu 3D replay |
| validation/ | Reference datasets + harness comparing Aurora output to published data |

Design rules:
- Core is headless and deterministic; UI only consumes it (enables CLI/cloud builds).
- SI units internally; unit conversion only at I/O and UI boundaries.
- Every physics model gets unit tests against published reference values.

## P0 prototype scope
- 6DOF rigid body, USSA76, .eng thrust curve
- Barrowman + compressibility-corrected drag to M3 (extend to M5 in P1)
- One booster→sustainer staging event
- Rail departure, apogee, deploy events
- CSV output + basic plot (CLI is fine for P0)

## Validation (current revs): openly available published data only
Personal flight data will be used in future revs, not now.

Priority sources:
1. ARCAS sounding rocket wind tunnel data (ARCAS Short and ARCAS Long configurations):
   subsonic, transonic, and supersonic CP and CD, including Mach 3–4 CP.
   Owner of the comparison: Rogers Aeroscience (RASAero II author); underlying data is NASA/NACA.
2. NACA RM A53D02 (Ames free-flight ballistic wind tunnel): CD vs. Mach.
   Use only the Mach ≤ 5 portion.
3. NACA/NASA reports linked from www.rasaero.com ("Comparisons with Wind Tunnel Data"),
   including sounding-rocket CP data at Mach 1.5–4.5 and free-flight model base drag data.
4. RASAero "Comparisons with Flight Data" page (https://www.rasaero.com/comparisons-flight.htm):
   backed-out in-flight CD and apogee-vs-flight comparisons for high-power rockets.
   Use as integrated-trajectory sanity checks.
5. NASA Technical Reports Server (ntrs.nasa.gov) for any additional public-domain
   NACA/NASA tabulated aero data.

Rules for validation data:
- Pull from the ORIGINAL NACA/NASA reports (US-government public domain) wherever possible.
  Do NOT commit RASAero's own charts/images into the repo; cite and link them instead.
- Prefer tabulated values from the reports. Where only plots exist, digitize
  (e.g. WebPlotDigitizer), record the digitization method, and assign a digitization
  uncertainty band (typically ±1–3% of axis range).
- Store each dataset as text (CSV) with a header block: source citation, report number,
  page/figure/table, Reynolds number / test conditions, units, and digitization notes.
- Match Reynolds number to the test conditions when comparing (wind tunnel ≠ flight Re).
- The harness reports error per Mach point and per regime, with pass/fail thresholds
  agreed with the owner (proposed starting point: CD ±10% subsonic/supersonic,
  ±20% transonic; CP ±0.25 caliber).

## First-session tasks
1. Confirm the repo is private (`gh repo view --json visibility`). Stop if it is public.
2. Add COPYRIGHT, README (one paragraph + "All rights reserved"), .gitignore (Rust),
   rust-toolchain.toml (pin stable), and deny.toml (license allowlist above).
3. Scaffold the Cargo workspace with empty crates per the layout above; `publish = false`
   and `license-file = "COPYRIGHT"` in every Cargo.toml.
4. CI (GitHub Actions): fmt, clippy -D warnings, test, cargo-deny. Private-repo Actions
   minutes are limited on free plans — keep the workflow lean.
5. Commit, then ask the owner before starting P0 physics.

## Open questions for the owner
1. Validation pass/fail thresholds (proposed values above)
2. Active control priority (canards, TVC, or roll) and whether controllers are native Rust
   or external SIL code (e.g. ARCTAS firmware logic)

## Owner preferences
- Honest, direct assessment; state uncertainty explicitly
- Implementation-level detail over conceptual overviews
- Data-grounded: cite the method/source for each model
