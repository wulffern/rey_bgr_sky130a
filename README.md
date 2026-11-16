[![GDS](../../actions/workflows/gds.yaml/badge.svg)](../../actions/workflows/gds.yaml)
[![DRC](../../actions/workflows/drc.yaml/badge.svg)](../../actions/workflows/drc.yaml)
[![LVS](../../actions/workflows/lvs.yaml/badge.svg)](../../actions/workflows/lvs.yaml)
[![DOCS](../../actions/workflows/docs.yaml/badge.svg)](../../actions/workflows/docs.yaml)

# Who
wulff

# Why

A curvature-compensated (second order) current-mode bandgap reference, ~0.6 V
VREF, built from tiles of [rey_atr](https://github.com/wulffern/rey_atr_sky130a)
cells on the 8.0 x 4.0 um transistor slot grid. Based on the bandgap core in
[lelo_temp](https://github.com/wulffern/lelo_temp_sky130a).

# How

Work in progress, see [PLAN.md](PLAN.md). The curvature correction follows

![](media/l3_curv_tikz.svg)


# What


| What            |        Cell/Name |
| :----           |  :----:       |
| Schematic       | design/REY_BGR_SKY130A/REY_BGR.sch |
| Layout          | design/REY_BGR_SKY130A/REY_BGR.mag |
| TT top level schematic | design/REY_BGR_SKY130A/tt_um_rey_bgr_analogicus.sch |
| TT top level layout    | design/REY_BGR_SKY130A/tt_um_rey_bgr_analogicus.mag |


# Signal interface

The TinyTapeout top level ties `uio_oe[7:0]` low, drives `PWRUP_1V8` from
`ui_in[0]` (with antenna diode) and puts `OSC_TEMP_1V8` on `uo_out[0]`. Pin
positions match `tt_block_1x1_pg.def` from TinyTapeout tt-support-tools
(check with `cd work && make ttpins`).


| Signal       | Direction | Domain  | Description                               |
| :---         | :---:     | :---:   | :---                                      |
| VDD_1V8      | Input     | VDD_1V8 | Main supply                               |
| OSC_TEMP_1V8 | Output    | VDD_1V8 | Temperature dependent oscillation frequency|
| PWRUP_1V8    | Input     | VDD_1V8 | Power up the circuit
| VSS          | Input     | Ground  |                                           |


# Key parameters


| Parameter           | Min     | Typ             | Max     | Unit  |
| :---                | :---:   | :---:           | :---:   | :---: |
| Technology          |         | Skywater 130 nm |         |       |
| AVDD                | 1.7     | 1.8             | 1.9     | V     |
| Temperature         | -40     | 27              | 125     | C     |
