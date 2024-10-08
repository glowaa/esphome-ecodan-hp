# Mitsubishi-CN105-Protocol-Decode
For Ecodan ASHP Units
Copy from: https://github.com/F1p/Mitsubishi-CN105-Protocol-Decode/blob/master/README.md
With my own findings using a procon as proxy.

# New findings
- 0x03 : error codes
- 0x05 : heat source
- 0x0B : refrigerant liquid temperature
- 0x10 : In1 thermostat H/C status
- 0x14 : booster / immersion heater states
- 0x15 : pump status
- 0x28 : forced dhw status 
- 0x35 : room temp setpoint (signed) with flags
- 0xC9 : configuration command. It reports back controller version and much more, need more investigation.

### Experimental prohibit commands
The procon send these commands when setting prohibit to true, but it does not seem to affect the heatpump. It turns out that these command works in server control mode.

prohibit cool zone 1
{ .Hdr { FC, 41, 02, 7A, 10 } .Payload { 34, 10, 00, 00, 00, 00, 00, 01, 00, 00, 00, 00, 00, 00, 00, 00 } .Chk { EE } }

prohibit cool zone 2
{ .Hdr { FC, 41, 02, 7A, 10 } .Payload { 34, 40, 00, 00, 00, 00, 00, 00, 00, 01, 00, 00, 00, 00, 00, 00 } .Chk { BE } }

prohibit heating zone 1
{ .Hdr { FC, 41, 02, 7A, 10 } .Payload { 34, 08, 00, 00, 00, 00, 01, 00, 00, 00, 00, 00, 00, 00, 00, 00 } .Chk { F6 } }

prohibit heating zone 2
{ .Hdr { FC, 41, 02, 7A, 10 } .Payload { 34, 20, 00, 00, 00, 00, 00, 00, 01, 00, 00, 00, 00, 00, 00, 00 } .Chk { DE } }

prohibit dhw
{ .Hdr { FC, 41, 02, 7A, 10 } .Payload { 34, 04, 00, 00, 00, 01, 00, 00, 00, 00, 00, 00, 00, 00, 00, 00 } .Chk { FA } }

# Physical
Serial, 2400, 8, E, 1
# Command Format
| Header | Payload | Checksum |
|--------|---------|----------|
| 5 Bytes | 16 Bytes | 1 Byte |
## Header
| Sync Byte | Packet Type | Uknown | Unknown | Payload Size |
|---|---|--|---|---|
| 0xfc | Type | 0x02 | 0x7a | Length |
### Sync Byte 
0xfc
### Packet Types
| Value | Packet Type      | Direction      |
|-------|------------------|----------------|
|  0x41 | Set Request      | To Heat Pump   |
|  0x61 | Set Response     | From Heat Pump |
|  0x42 | Get Request      | To Heat Pump   |
|  0x62 | Get Response     | From Heat Pump |
|  0x5A | Connect Request  | To Heat Pump   |
|  0x7A | Connect Response | From Heat Pump |
|  0x5B | Configuration Request  | To Heat Pump   |
|  0x7B | Configuration Response | From Heat Pump   |
### Length
Payload Size (Bytes)
## Payload
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| Command | x | x | x | x | x | x | x | x | x | x  | x  |  x |  x |  x |  x |  x |
## Checksum
Checksum = 0xfc - Sum ( PacketBytes[0..20]) ;
# Set Request - Packet Type 0x41
## Available Commands 
Active commands so far identified.
| Command | Brief Description |
| ------- | ----------- |
| 0x32 |  Update Settings |
| 0x34 | Controller Settings |
| 0x35 | Room Settings |
### 0x32 - Set Options
|   0   |   1   | 2 | 3 | 4 |  5  |  6  |  7  |   8   |   9   |  10  |  11  |  12  |  13  | 14 | 15 | 16 |
|-------|-------|---|---|---|-----|-----|-----|-------|-------|------|------|------|------|----|----|----|
| 0x32  | Flags | Z | P |   | DHW | HC1 | HC2 | DHWSP | DHWSP | Z1SP | Z1SP | Z2SP | Z2SP |    |    |    |  

* Flags : Flags to Indicate which fields are active
  * 0x80 : Set Zone Setpoints, Byte[2] determines which Zones 
  * 0x40 : Unknown 
  * 0x20 : Set Hotwater Setpoint
  * 0x10 : Unknown
  * 0x08 : Set Heating Control Mode
  * 0x04 : Set Hot Water Mode
  * 0x02 : Unknown
  * 0x01 : Set System Power Power
* Z : Zones the Command Applies to
  * 0x00 : Zone 1
  * 0x01 : Zone 2 ( Probably )
  * 0x02 : Both
* P : System Power
  * 0x00 : Standby
  * 0x01 : Power On
* DHW : Hot Water Mode
  * 0x00 : Normal
  * 0x01 : Eco
* HC1 : Heating Control Mode Zone 1
  * 0 : Temperature Mode
  * 1 : Flow Control Mode
  * 2 : Compensation Curve Mode
* HC2 : Heating Control Mode Zone 2
  * 0 : Temperature Mode
  * 1 : Flow Control Mode
  * 2 : Compensation Curve Mode
* DHWSP : Hot Water Setpoint (Temperature * 100)
* Z1SP : Zone 1 Setpoint (* 100)
* Z2SP : Zone 1 Setpoint (* 100)
### 0x34 - Hot Water and Holiday Mode
|   0   |  1  | 2 | 3 |   4  |  5   |   6  |  7  |   8   |   9   |  10  |  11  |  12  |  13  | 14 | 15 | 16 |
|-------|-----|---|---|------|------|------|-----|-------|-------|------|------|------|------|----|----|----|
| 0x34  |Flags|   |DHW|  HOL | IDHW | Z1HI | Z1CI| Z2HI  | CZ2I  | SCM  |      |      |      |    |    |    |  
* Flags : Flags to Indicate which fields are active
  * 0x01 : Hot Water Force (Boost)
  * 0x02 : Holiday Mode
  * 0x04 : DHW Inhibit   -- Suspected
  * 0x08 : Heating Z1 Inhibit  -- Suspected
  * 0x10 : Cooling Z1 Inhibit  -- Suspected
  * 0x20 : Heating Z2 Inhibit  -- Suspected
  * 0x40 : Cooling Z2 Inhibit  -- Suspected
  * 0x80 : Server Control Mode
* DHW : On (1) / Off (0)
* HOL : On (1) / Off (0)
* SCM : Server Control Mode On (1) / Off (0)
* IDHW : Inhibit DHW On (1) / Off (0)  -- Suspected
### 0x35 - Set Zone Setpoint (signed)
Identified so far, this must do far more that this!
|   0   |   1  | 2 | 3 |   4  |  5   | 6    |  7   |   8   |   9   |  10  |  11  |  12  |  13  | 14 | 15 | 16 |
|-------|------|---|---|------|------|------|------|-------|-------|------|------|------|------|----|----|----|
| 0x35  | flag |   | CH| Z1SP | Z1SP | Z2SP | Z2SP |       |       |      |      |      |      |    |    |    |  
* flag : zone 1 / zone 2 flag
  * 0x02 : Zone 1
  * 0x08 : Zone 2
* CH: Cool/Heat flag, 1 when setting room temp in cool mode, 0 in heating mode
* Z1SP : Zone 1 Setpoint (* 100)
* Z2SP : Zone 2 Setpoint (* 100)
# Get Request - Packet Type 0x42
## Available Commands 
Active commands so far identified, 0x00 to 0xff. Commands not listed appear to generate no resaponse. Some command listed have empty, payload 0x00, response.
| Command | Brief Description |
| ------- | ----------- |
| 0x01 | Time & Date |
| 0x02 | Defrost |
| 0x03 | Error codes |
| 0x04 | Compressor Frequency |
| 0x05 | Hot Water Flag |
| 0x06 | Unknown - Empty Response |
| 0x07 | Output Power |
| 0x08 | Unknown |
| 0x09 | Zone 1 & 2 Temperatures and Setpoints, Hot Water Setpoint |
| 0x0b | Zone 1 & 2 and Outside |Temperature
| 0x0c | Water Flow Temperatures |
| 0x0d | Boiler Flow Temperatures |
| 0x0e | Unknown |
| 0x10 | Thermostats |
| 0x11 | Unknown |
| 0x13 | Run Hours |
| 0x14 | Primary Flow Rate & Booster/Immersion |
| 0x15 | Pump status |
| 0x16 | Unknown |
| 0x17 | Unknown - Empty Response |
| 0x18 | Unknown - Empty Response |
| 0x19 | Unknown - Empty Response |
| 0x1a | Unknown - Empty Response |
| 0x1c | Unknown - Empty Response |
| 0x1d | Unknown - Empty Response |
| 0x1e | Unknown - Empty Response |
| 0x1f | Unknown - Empty Response |
| 0x20 | Unknown - Empty Response |
| 0x26 | Various Operantion Mode Flags |
| 0x27 | Unknown |
| 0x28 | Various Operantion Mode Flags |
| 0x29 | Zone 1 & 2 Temperatures |
| 0xa1 | Unknown |
| 0xa2 | Unknown |
| 0xa3 | Unknown - Empty Response |
| 0xc9 | Hardware configuration |
### Payload - All Commands
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| Command | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0  | 0  |  0 |  0 |  0 |  0 |  0 |
# Set Response - Packet Type 0x61
## Available Commands 
Active commands so far identified.
| Command | Brief Description |
| ------- | ----------- |
| 0x00 |  OK |
### 0x00 - Set command ACK
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| Command | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0  | 0  |  0 |  0 |  0 |  0 |  0 |
# Get Response - Packet Type 0x62
## Resposes 
Responses so far identified.
| Command | Brief Description |
| ------- | ----------- |
| 0x01 | Time & Date |
| 0x05 | Various Flags |
| 0x09 | Zone 1 & 2 Temperatures and Setpoints, Hot Water Setpoint |
| 0x0b | Zone 1 & 2 and Outside Temperature |
| 0x0c | Water Flow Temperatures |
| 0x0d | Boiler Flow Temperatures |
| 0x26 | Various Operantion Mode Flags |
| 0x28 | Various Operantion Mode Flags |
| 0x29 | Zone 1 & 2 Temperatures |
### 0x01 - Time & Date
|   0   | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x01  | Y | M | D | h | m | s |   |   |   |    |    |    |    |    |    |    |  
* Y: Year
* M: Month
* D: Day
* h: Hour
* m: Minute
* s: Second
### 0x02 - Defrost
|   0   | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x02  |   |   | D |   |   |   |   |   |   |    |    |    |    |    |    |    |  
* D: Defrost
### 0x03 - Error codes
|   0   | 1 | 2 | 3 | 4 | 5 | 6 | 7 |  8  | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|---|---|---|---|-----|---|----|----|----|----|----|----|----|
| 0x03  | RE|FC1|FC2|FT1|FT2|   |   |  M  | S |    |    |    |    |    |    |    |  
* RE: refrigerant error code
* FC1+FC2: Error code FC1 * 100 + FC2
* FT1: fault code letter (first)
* FT2: fault code letter (second)
* M: Multi Zone Running Parameter (3 = Z2/2 = Z1/1 = Both Active)
* S: Single Zone Running Parameter (TBC)?
### 0x04 - Various Flags
|   0   | 1  | 2 | 3 | 4 | 5 | 6 |  7  | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|----|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x04  | CF |   |   |   |   |   |   |   |   |    |    |    |    |    |    |    |  
* CF : Compressor Frequency
### 0x05 - Various Flags
|   0  | 1 | 2 | 3 | 4 |  5 | 6 |  7  | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|------|---|---|---|---|----|---|-----|---|---|----|----|----|----|----|----|----|
| 0x05 |   |   |   |   | DE | HS| HSP |   |   |    |    |    |    |    |    |    |  
* DE : Value of 7 when in Hot Water on Temp Drop Mode
* HS : Heat source
  * 0 : heatpump
  * 1 : immersion heater
  * 2 : booster heater
  * 3 : booster heater + immersion heater
  * 4 : boiler
* HSP : Heat Source Phase – DHW, 0 = Normal, 1 = H/P Phase, 2 = Heater Phase
### 0x07 
|   0   | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x07  |   |   |   |   |   |   |   |   | P |    |    |    |    |    |    |    |  
* P : Heater Power (*10 = %?)
### 0x09 - Zone 1 & 2 Temperatures and Setpoints, Hot Water Setpoint
| 0    |   1  |   2  | 3    | 4    | 5    | 6    | 7    | 8    |  9  |  10 |  11 | 12 | 13 | 14 | 15 | 16 |
|------|------|------|------|------|------|------|------|------|-----|-----|-----|----|----|----|----|----|
| 0x09 | Z1T  | Z1T  | Z2T  | Z2T  | Z1ST | Z1SP | Z2SP | Z2SP | LSP | LSP | HWD |  ? | ?  |    |    |    |
* Z1T  : Zone1 Target Temperature * 100
* Z2T  : Zone2 Target Temperature * 100;
* Z1SP : Zone 1 Flow SetFlow Setpoint * 100
* Z2SP : Zone 3 Flow SetFlow Setpoint * 100
* LSP  : Legionella Setpoint * 100;
* HWD  : DHW Max Temp Drop;
### 0x0b - Zone 1 & 2 and Outside Temperature
|   0  |  1  |  2  |  3  | 4 | 5 | 6 | 7 |  8  | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|------|-----|-----|-----|---|---|---|---|-----|---|----|----|----|----|----|----|----|
| 0x0b | Z1T |     | Z2T |   |   |   |   |  RT |   |    | O  |    |    |    |    |    |
* Z1T : Zone1 Temperature * 100
* Z2T : Zone2 Temperature * 100
* RT : Refrigerant liquid temperature * 100
* O : Outside Temp  +40 x 2 
### 0x0c - Heater Flow Temps
|  0   | 1  | 2  | 3 | 4  | 5  | 6 | 7  |  8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|------|----|----|---|----|----|---|----|----|---|----|----|----|----|----|----|----|
| 0x0c | OF | OF |   | RF | RF |   | HW | HW |   |    |    |    |    |    |    |    |
* OF : Heater Water Out Flow  * 100
* RF : Heater Return Flow Temperature * 100
* HW : Hot Water Temperature * 100
### 0x0d - Boiler Temps
|  0   | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|------|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x0d | F1| F1|   | R1| R1|   | F2| F2|   |  R2|  R2|    |    |    |    |    |
* F1 : Boiler? / Z1 Flow Temperature * 100;
* R1 : Boiler? / Z1 Return Temperature * 100;
* F2 : Z2 Flow Temperature * 100;
* R2 : Z2 Return Temperature * 100;
### 0x0e - Unknown Temps
|  0   | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|------|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x0e |   |   |   |   |   |   |   |   |   |    |    |    |    |    |    |    |
Several Unknown Temperatures
### 0x10 - External sources
|   0   | 1  | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|----|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x13  | T1 | T2| T3|   |   |   |   |   |   |    |    |    |    |    |    |    |  
* T1 : In1 room thermostat 1 H/C request status (on/off)
* T2 : In6 room thermostat 2 H/C request status (on/off)
* T3 : In5 outdoor thermostat (on/off)
### 0x13 - Run Hours
|   0   | 1  | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|----|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x13  | U  |   |RH | RH|RH |   |   |   |   |    | PF |    |    |    |    |    |  
* U : Unknown
* RH: Run Hours
### 0x14 - Primary Cct Flow Rate
|   0   | 1  | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|----|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x14  |    | B |   |   | I |   |   |   |   |    | PF |    |    |    |    |    |  
* PF : Primary Flow Rate (l/min)
* B : Booster heater active
* I : Immersion heater active
### 0x15 - Pump status
|   0   | 1  |  2 |  3 |  4 | 5 |  6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|----|----|----|----|---|----|---|---|---|----|----|----|----|----|----|----|
| 0x15  | P1 |    |    | P2 |   | V1 | V2|   |   |    |    |    |    |    |    |    |  
* P1 : Water pump 1 status
  * 0 : Off
  * 1 : On
* P2 : Water pump 2 status
  * 0 : Off
  * 1 : On
* V1 : 3 way valve 1 status
  * 0 : Off
  * 1 : On
* V2 : 3 way valve 2 status
  * 0 : Off
  * 1 : On
### 0x16 - Unkown
|   0   | 1  |  2 |  3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|----|----|----|---|---|---|---|---|---|----|----|----|----|----|----|----|
| 0x16  | DHW| U1 | U2 |   |   |   |   |   |   |    |    |    |    |    |    |    |  
* DHW : DHW Pump Running Flag? 3 Way Valve?
* U1 : Heating Flag?
* U2 : Heating Flag?
### 0x26
| 0 | 1 | 2 |  3  | 4  | 5  | 6  |  7 |   8  |  9   |  10 |  11 | 12 | 13 | 14 |
|---|---|---|-----|----|----|----|----|------|------|-----|-----|----|----|----|
|   |   |   | Pwr | OM | HW |OpZ1|OpZ2| HWPS | HWSP | HSP | HSP | SP | SP |    |
* Pwr - Power
  * 0 : Standby
  * 1 : On
* OM Operation Mode
  * 0 : Off
  * 1 : Hot Water On
  * 2 : Heating On
  * 5 : Frost Protect
  * 6 : Legionella
* HW - Hot Water Mode
  * 0 : Normal
  * 1 : Economy
* Op - Operation Mode (Zone 1 / Zone 2): 
  * 0 : Temperature Mode
  * 1 : Flow Control Mode
  * 2 : Compensation Curve Mode
* HWSP : HotWater SetPoint * 100;
* HSP : Heating Flow SetPoint * 100;
* SP : Unknown Setpoint:
### 0x28 - Various Flags
|   0   | 1 | 2 | 3 | 4  | 5  |  6 | 7  |  8 |  9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|----|----|----|----|----|----|----|----|----|----|----|----|----|
| 0x28  |   |   |FD | HM | HT |PHZ1|PCZ1|PHZ2|PCZ2|    |    |    |    |    |    |  
* FD : Forced DHW active
* HM : Holiday Mode
* HT : Prohibit DHW
* PHZ1 : Prohibit Heating Zone1
* PCZ1 : Prohibit Cooling Zone1
* PHZ2 : Prohibit Heating Zone2
* PCZ2 : Prohibit Cooling Zone2
### 0x29 - 
|   0   | 1 | 2 | 3 |  4  |  5  |  6  |  7  | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|-----|-----|-----|-----|---|---|----|----|----|----|----|----|----|
| 0x29  |   |   |   | Z1T | Z1T | Z2T | Z2T |   |   |    |    |    |    |    |    |    |  
* Z1T : Zone1 Temperature * 100
* Z2T : Zone2 Temperature * 100
### 0xA1 - Consumed Energy
|   0   | 1 | 2 | 3 |  4  |  5  |  6  |  7  | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|-----|-----|-----|-----|---|---|----|----|----|----|----|----|----|
| 0xA1  | Y | M | D | Heat|     |     |Cool |   |   |DHW |    |    |    |    |    |    |  
* Y: Year
* M: Month
* D: Day
### 0xA2 - Delivered Energy
|   0   | 1 | 2 | 3 |  4   |  5  |  6  |  7   | 8 | 9 | 10  | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|------|-----|-----|------|---|---|-----|----|----|----|----|----|----|
| 0xA2  | Y | M | D | Heat |     |     | Cool |   |   | DHW |    |    |    |    |    |    |  
* Y: Year
* M: Month
* D: Day
### 0xC9 - Hardware configuration
|   0   | 1 | 2 | 3 |  4   |  5  |  6  |  7   | 8 | 9 | 10  | 11 | 12 | 13 | 14 | 15 | 16 |
|-------|---|---|---|------|-----|-----|------|---|---|-----|----|----|----|----|----|----|
| 0xC9  |   |   |   |      |     |  C  |      |   |   |     |    |    |    |    |    |    |  
* C: Controller version
  * 0: FTC2B
  * 1: FTC4
  * 2: FTC5
  * 3: FTC6
  * 128: CAHV1A
  * 129: CAHV1B
  * 130: CRHV1A 
  * 131: CRHV1B
  * 132: EAHV1A
  * 133: EAHV1B
  * 134: QAHV1A
  * 135: QAHV1B
  * 144: PWFY1
