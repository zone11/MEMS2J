# MEMS2J Interface — Hardware Interface

## Vehicle Connector — OBD2 Diagnostic Port
**Not** OBD2-compliant — proprietary KWP-2000
16-pin OBD2 form factor (SAE J1962)

### Pin Assignment (only relevant pins)

| Pin | Signal | Usage |
|---|---|---|
|4|Chassis Ground|Interface GND|
|5|Signal Ground|Interface GND|
|7|K-Line (ECU data)|K-Line interface|
|15|L-Line (slow-init)|Not used at the moment|
|16|Battery +12V |Interface DC-DC Converter input|

### K-Line Electrical Characteristics

| Parameter | Value |
|---|---|
| Logic HIGH | Battery voltage (12–14.4V, engine running) |
| Logic LOW | 0V (GND) |
| Topology | Single-wire, half-duplex |
| Echo | Every transmitted byte is echoed back on RX |

## Tranceivers to interface K-Line with MCU

### Newest available tranceivers
- [TI TLIN1027-Q1](https://www.ti.com/product/TLIN1027-Q1) Automotive local interconnect network (LIN) transceiver for K-line applications (Favorite)
- [TI TLIN1021A-Q1](https://www.ti.com/product/TLIN1021A-Q1) Automotive Fault-Protected LIN Transceiver with Inhibit and Wake

### Still avilable tranceivers
- [SN65HVDA100-Q1](https://www.ti.com/product/SN65HVDA100-Q1) LIN Physical Interface
- [SN65HVDA195-Q1](https://www.ti.com/product/SN65HVDA195-Q1) Automotive Catalog LIN, MOST ECL, and K-Line Physical Interface

### Legacy tranceivers
- [NXP 33290](https://www.nxp.com/docs/en/data-sheet/MC33290.pdf) ISO K Line Serial Link Interface
- [ST L9637](https://www.st.com/resource/en/datasheet/l9637.pdf) Monolithic bus driver with ISO 9141 interface

## Implementation with TI TLIN1027-Q1
<img width="504" height="279" alt="fbd_sllsf58b" src="https://github.com/user-attachments/assets/d14a9371-f05e-4bf6-820a-a94b9bd07df3" />
