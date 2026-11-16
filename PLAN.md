# REY_BGR plan: curvature-compensated bandgap on a transistor grid

Status 2026-11-16: the IP is created from the `rey` template and this plan is
written. Nothing has been designed or simulated yet, so every number below is a
hand estimate for milestone M1 to check. See [Handoff](#handoff) at the end.

## Goal

A current-mode bandgap reference with these properties:

1. It is based on the bandgap core in `lelo_temp_sky130a`
   (`design/LELO_TEMP_SKY130A/LELOTEMP_BIAS_IBP.sch`).
2. It adds the second-order (curvature) compensation from
   <https://analogicus.com/aic2027/assets/media/l3_curv_tikz.svg>.
3. It is built from **tiles of rey_atr cells**. Every transistor, resistor,
   capacitor and bipolar sits on the 8.0 x 4.0 um rey_atr transistor slot.

### Initial targets (goals, confirm in M1)

| Parameter | Target | Note |
| :--- | :---: | :--- |
| VREF | ~0.6 V | Current mode, set by R3 |
| VDD | 1.7 / 1.8 / 1.9 V | |
| Temperature | -40 .. 125 C | |
| TC without curvature correction | measure | the baseline (I_NL branch off) |
| TC with curvature correction | measurably better than baseline, goal < 10 ppm/C typ | box method |
| Supply current | < 10 uA | |
| Line regulation | measure | decides whether the mirrors need cascodes |
| Startup | reliable at all corners, < 10 us | |
| Outputs | VREF, a zero-TC current I_REF, and I_PTAT | |

## Architecture

### 1. Core: lelo_temp loop plus Banba split resistors

lelo_temp's `LELOTEMP_BIAS_IBP` is a PTAT core:

- Q1 is 1x and Q2 is 8x.
- The PMOS mirror is 8:1.
- `LELOTEMP_OTAR` forces VD1 = VR1.
- R1 sits between VR1 and VD2, which gives I_PTAT = VT ln(64)/R1, about 1.1 uA at 25 C.
- It puts out only I_PTAT and VC. It has no VREF.

The change: add **R2A from VD1 to VSS** and **R2B from VR1 to VSS**, with
R2A = R2B/M, where M is the mirror ratio I_A/I_B. Each mirror branch then
carries I_PTAT + V_EB/R2, which is temperature-flat to first order. Per unit
(branch B):

    I_PTAT = VT ln(M N) / R1                      (N = Q2/Q1 area ratio)
    I_CTAT = V_EB,Q1 / R2B
    I_REF  = I_PTAT + I_CTAT                      first-order zero TC when
    R2B/R1 = |dV_EB/dT| / ((k/q) ln(M N))

lelo_temp uses M = 8 and N = 8, which costs 18 PMOS slots once cascoded. To
fit the grid, start from **M = 1, N = 8** (dV = VT ln 8, about 54 mV). Let the
model trade OTA offset sensitivity against slot count.

### 2. Curvature loop (from the slide)

- **QA** is the core's Q1. Its current, M*I_PTAT, is PTAT.
- **QB** is a new PNP identical to Q1. A mirror copy K*I_REF (temperature-flat) feeds it.
- **R4** runs from the QB emitter to node X.
- **OTA2** (+ at X, - at VD1) drives MPC, which pushes I_NL into X:

      I_NL = (V_EB,A - V_EB,B) / R4

  Because I_A is proportional to T and I_B is constant, the difference carries
  `+ VT ln(T/T0)`.
- V_EB has curvature `-(eta-1) VT ln(T/T0)` at PTAT bias, so I_CTAT inherits it
  divided by R2B. They cancel when

      R4 = R2B / (eta - 1)            eta about 3.5..4, extract it from the sky130 PNP model in M1

- I_NL also has a linear part (VT ln(M I_PTAT / K I_REF) / R4), so re-balance
  R1/R2B with I_NL included. The model does this numerically.
- QB carries K*I_REF + I_NL, not exactly K*I_REF. This is a second-order error; the model includes it.

### 3. Output

- MPD copies I_NL to the output node. A mirror copy of I_REF also feeds it.
- `VREF = R3 (I_REF + I_NL)`.
- Every resistor is the same poly type, so resistor TC drops out of every ratio
  and VREF depends only on R3/R1, R3/R2B and R3/R4.

A cheaper alternative, if the slot budget is too tight: Malcovati et al. (JSSC
2001) reuse the core OTA's virtual ground. They put a resistor from each core
OTA input to the QB emitter, so no second OTA and no MPC/MPD are needed. Keep the
slide's topology as the baseline, because it is what the lecture shows. Revisit
this only if M3 overflows the tiles.

### First sizing estimate (M = 1, N = 8, I_PTAT about 1 uA)

| R | Value | Comment |
| :--- | ---: | :--- |
| R1 | ~54 k | VT ln 8 / 1 uA |
| R2A, R2B | ~540 k each | R2B/R1 about 10 |
| R4 | ~180 k | R2B/(eta-1), eta about 4 |
| R3 | ~270 k | 0.6 V / about 2.2 uA |
| **Sum** | **~1.6 M** | |

## The grid

### Slot unit (from rey_atr_sky130a, magscale 1 2, so 1 unit = 5 nm)

| Cell | Size (um) | Slots |
| :--- | :--- | :--- |
| `REYATR_{N,P}CH_<n>C{5F0,1F2}[D]`, `LVT_` | 8.0 x 4.0 | 1 (every contact count is the same box) |
| `REYATR_*CTAPTOP/BOT` | 8.0 x 2.4 | column cap |
| `REYATR_RES_36C2F0` (2 x res_high_po W0.4 L8.16) | 16.0 x 4.0 | 2 |
| `REYATR_HRPPO2/8/12` (columns of the row above) | 16.0 x (n/2 * 4 + 4.8) | n |
| `REYATR_CAPX1` (MIM 4.8 x 5.0, about 48 fF, no guard) | 8.0 x 8.0 | 2 (and it can sit over devices on M4/M5) |
| sky130 `rf_pnp_05v5_W3p40L3p40` | 6.7 x 6.7 | 2 once it has an 8 x 8 wrapper (**to make**) |
| `REYATR_RES2/RES4/RPPO2/RPPO4` | off grid or legacy | **do not use** |

### The tile: 4 columns x 6 rows = 24 slots, 32.0 x 28.8 um

The request was "about 20 transistors". Use 24, because it is the size that
everything already in rey_atr fills exactly:

- **MOS tile**: 4 columns (each one type, NCH or PCH) x 6 rows, tap rows top and bottom.
- **RES tile**: two `HRPPO12` side by side (each 2 columns x 6 rows + taps = 16 x 28.8 um). That is 24 resistors.
- **PNP tile**: 4 x 3 wrapped PNPs = 12, which is Q1 + Q2[8] + QB + 2 dummies.
  QA (Q1) and QB go in the centre, next to each other. QA and QB match each other; Q1/Q2 need common centroid.
- **CAP tile**: 4 x 3 CAPX1 = 12, about 0.58 pF. It probably costs nothing because it can lie over the MOS/RES tiles.

A 4 x 5 tile (20 slots) leaves the PNP grid at half a row and has no matching
HRPPO column, so it is not used.

### The resistor problem (must solve before layout)

`res_high_po` is about 317 ohm/sq. The rey_atr row gives about 6.5 kohm per slot.

- 1.6 Mohm in high_po is about 245 slots, roughly 10 RES tiles. Not acceptable.
- `res_xhigh_po` is 2 kohm/sq (TC1 about -2.5e-3/K, fine because only ratios
  matter). It gives about 41 kohm per slot, so 1.6 Mohm is about 40 slots, or
  **2 RES tiles** with spare for trim and dummies.
- The other option is to raise the current 5x (5x the power).

**Plan**: add `REYATR_XRES_36C2F0` (xhigh_po, identical footprint, pins and
`cic/res` symbol) and `REYATR_XHRPPO{2,8,12}` columns to `rey_atr_sky130a`.
Copy how `RES_36C2F0` and `HRPPO*` are generated in
`rey_atr_sky130a/cic/resistors_uniform.json`.

### Device budget (estimate)

| Block | MOS slots | Other |
| :--- | ---: | :--- |
| PMOS mirrors: MA, MB, M_QB, M_OUT, MPC, MPD | 6 (12 with cascodes) | |
| OTA1 (core): PMOS-input 5T, since the input CM is V_EB about 0.6-0.7 V | 5 | |
| OTA2 (curvature) | 5 | |
| Bias for the OTA tails | 2 | |
| Startup (lelo_temp style) | 3 | |
| Power down | 3 | |
| **Total** | **24 (no cascodes) .. 30** | |
| Bipolars | | 10 + 2 dummies, 1 PNP tile |
| Resistors (xhigh_po) | | about 40, 2 RES tiles |
| Compensation caps | | up to 12 CAPX1, over the devices |

That is **4-5 tiles**, for example a 3 x 2 arrangement of about 96 x 58 um.

## Milestones

### M0: IP and plan (done)

`cicconf newip bgr`, PLAN.md, GitHub repo `wulffern/rey_bgr_sky130a`.

### M1: Model (`py/model.py`, with notebook or markdown output)

1. ngspice on the sky130 PNP (`pnp_05v5` W3p40L3p40), temperature -40..125:
   - V_EB(T) at constant I and at PTAT I. Fit V_G0, eta and the effective TC.
   - xhigh_po R(T).
2. Analytic plus numeric sizing of M, N, K, R1, R2A, R2B, R3 and R4. Plot VREF(T)
   with and without I_NL, and report ppm/C (box) for both.
3. Map the resistors to xhigh_po unit counts, and the PMOS ratios to slot counts.
   Check the budget table above.

Acceptance: the model shows the curvature-corrected VREF beating the uncorrected
one, with a quantified residual, and every component is an integer number of slots.

### M2: Grid cells in rey_atr_sky130a (separate repo, wulffern/rey_atr_sky130a)

- `REYATR_XRES_36C2F0`, `REYATR_XRES_36CTAPTOP/BOT`, and `REYATR_XHRPPO{2,8,12}`.
- `REYATR_PNPX1`: the sky130 rf PNP in an 8.0 x 8.0 frame. Use CAPX1 as the
  pattern for an on-grid box. Give it a guard or a well story, and a symbol whose
  pins match `sky130_fd_pr/pnp_05v5.sym`.
- Make DRC and LVS clean on each. Bump rey_atr in this IP's config.yaml if it gets pinned.

### M3: Schematics (`design/REY_BGR_SKY130A/`)

- `REYBGR_CORE`: copied from `LELOTEMP_BIAS_IBP`, plus R2A/R2B and an I_REF
  output mirror. Copy `LELOTEMP_OTAR` as `REYBGR_OTA`.
  - Do **not** include lelo_temp cells across repos. Rename them `REYBGR_*`.
- `REYBGR_CURV`: QB, R4, OTA2 (an instance of `REYBGR_OTA`), MPC and MPD.
- `REY_BGR`: the top level, with R3 and the output.
  - Pins: VDD_1V8, VSS, PWRUP_1V8, VREF, IREF_1U, IPTAT_1U.
- Until M2 lands, use `sky130_fd_pr/res_xhigh_po` and `pnp_05v5` primitives
  directly, sized like the future cells, so the netlist is unchanged when they swap in.
- Add a switch that disables the curvature loop for A/B comparison. For example,
  a `CURV_EN` pin that gates MPD, or a testbench `.alter` that sets MPD's m=0.

### M4: Simulation (`sim/REY_BGR/`)

Use `lelo_temp_sky130a/sim/LELOTEMP_BIAS_IBP/` as the template (`dc.spi`,
`dc.meas`, `dc.yaml`, `lstb.*`, `tran.*`, `cicsim.yaml`).

- `dc`: `.dc temp -40 125 5`. Measure VREF at -40, 27 and 125 C, VREF max-min,
  ppm/C (box), and I_DD. Run once with curvature on and once with it off.
- `lstb`: loop gain, phase margin and gain margin of **both** OTA loops.
- `tran`: startup from PWRUP and a VDD ramp at all corners. Check for the zero-current state.
- Line: VREF over VDD 1.7..1.9.
- Corners: `Tt Tl Th` x `Vt Vl Vh`, plus process corners via the cicsim corner config.
  Do Monte Carlo after the layout is clean.
- Put the spec table in `dc.yaml` and friends, and use `cicsim summary` for the README.

### M5: Layout

- `py/<CELL>.py` hooks and `*.groups.yaml` place the tiles. See the
  `lelo_temp_sky130a/design/LELO_TEMP_SKY130A/*.groups.yaml` and `LELO_TEMP.py` examples.
- `cd work && make mag CELL=...`, then `make drc cdl lvs OPT='--short'`, then lpe, then re-run M4 on `Lay`.
- M1 carries power. M2, M3 and M4 carry signal. The caps go on M4/M5 over the devices.

### M6: Documentation and TinyTapeout

- README: who, why, how, signal interface, key parameters (from `cicsim summary`) and figures.
- `info.yaml`: `cell: REY_BGR`, plus author and github fields.
- `tt_um_rey_bgr_analogicus.sch/.mag` came from the template and is wired for
  `OSC_TEMP_1V8`. Rewire it to bring out VREF on an analog pin (`ua[0]`).

## Handoff

### Where things are

- IP: `/Users/wulff/data/2023/aicex/ip/rey_bgr_sky130a`. Remote: `git@github.com:wulffern/rey_bgr_sky130a.git`.
- `design/REY_BGR_SKY130A/REY_BGR.sch` is still the template placeholder (two
  m4 resistors from PWRUP to OSC_TEMP). Replace it in M3.
- The top-level `ip/config.yaml` has a new entry for `rey_bgr_sky130a`.
  Commit it in the aicex repo if it is not already committed.

### References

- Base design: `lelo_temp_sky130a/design/LELO_TEMP_SKY130A/LELOTEMP_BIAS_IBP.{sch,md}`
  (core and derivation), `LELOTEMP_OTAR.sch` (OTA), `LELOTEMP_BIPOLAR.sch`
  (Q1 + Q2[7:0]), and `py/LELO_TEMP.py` (model style).
- Grid library: `rey_atr_sky130a` (cells in `design/REY_ATR_SKY130A`, generators
  in `cic/*.json`). Read the git log, which explains the grid choices: commits
  243b3bc (CAPX1 on grid), 2dff8c3 (horizontal resistor row), and 21a0a57
  (`cic/res` symbol pins).
- Curvature slide: rendered copy at `media/l3_curv_tikz.svg`.

### Next action

Start M1: the PNP characterisation and `py/model.py`.

### Gotchas

- Never `git add` IP folders to the top-level aicex repo. Use cicconf and config.yaml.
- CI takes the cell from `info.yaml`, not from `work/Makefile`. Keep `info.yaml`'s
  `cell:` on something that is LVS-clean, or CI goes red.
- The legacy `REYATR_RES2/RES4/RPPO*` cells are off grid. Use `RES_36C2F0`/`HRPPO*`, or the XRES variants.
- Use Click for any Python CLI. Use logging, not print (repo rule).
