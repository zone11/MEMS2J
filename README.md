# MEMS2J Interface

Goal is to build an (maybe ESP32-based) interface for communicating with the **Rover MEMS2J ECU** over K-Line / KWP-2000 to read engine vitals without the need of Rover or 3rd party testing equipment. I'm using my Rover Classic Mini MPI for all the testings.

We'll see how this ends..

## Electrical interface

The connection to the car is made using an ODB2 Connector (16-pin) with a custom pinout.
All communication with the car is done via K-Line (ISO 9141), Single-Wire Half-Duplex bus at 10400 baud (8N1) on Pin 7 ot the connector.

More details about OBD2 pin assignment, K-Line interface (L9637D / transistor), power supply, MCU pins, cable here:
[hardware_interface.md](docs/hardware_interface.md)
## Communication protocol

The MEMS2J (and MEMS3) use KWP-2000 (ISO 14230-4) over K-Line to talk to the ECU. A key is required to access certain commands, therefore an algorithm to calculate the key was required.

More details about the KWP-2000 packet format, connection sequence, all 0x21 frame tables, seed-to-key algorithm, fault codes here:
[protocol_and_data_formats.md](docs/protocol_and_data_formats.md)

### Available Live Data (verified)

| Frame ID | Data point | Unit |
|---|---|---|
| `0x01` | Coolant temperature | °C |
| `0x03` | Intake air temperature (IAT) | °C |
| `0x07` | MAP sensor | kPa |
| `0x08` | Throttle position (TPS) | degrees |
| `0x09` | Engine speed | RPM |
| `0x0A` | Lambda / AFR | mV / AFR |
| `0x0D` | Vehicle speed | km/h |
| `0x0F` | Switch states (throttle, ignition, AC) | bit |
| `0x10` | Battery voltage | V |
| `0x12` | Idle air valve position | steps |
| `0x13` | Closed-loop status | bool |
| `0x19` | Fault codes | bit fields |

---

##