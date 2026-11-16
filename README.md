[![GDS](../../actions/workflows/gds.yaml/badge.svg)](../../actions/workflows/gds.yaml)
[![DRC](../../actions/workflows/drc.yaml/badge.svg)](../../actions/workflows/drc.yaml)
[![LVS](../../actions/workflows/lvs.yaml/badge.svg)](../../actions/workflows/lvs.yaml)
[![DOCS](../../actions/workflows/docs.yaml/badge.svg)](../../actions/workflows/docs.yaml)

# Who
wulff

# Why

<explain why you made this module>

# How

<explain short how you made this module>


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
