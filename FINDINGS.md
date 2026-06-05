# Harness Tester Challenge Findings

Target: https://github.com/commaai/harness_tester_challenge at `069f724`.

This list counts distinct bugs, not repeated KiCad DRC instances. I used the
schematic netlist exported by KiCad 10.0.3, the PCB DRC report, the firmware
source, and current component documentation.

## Verification Sources

- KiCad ERC: 107 violations. Actionable hard errors include undriven power and
  antenna input pins; most library warnings are environmental.
- KiCad DRC: 853 violations, plus 1 unconnected item and 2 schematic/PCB parity
  issues.
- Exported schematic netlist: `/tmp/harnesstester_reports/netlist.xml` during
  analysis.
- Firmware source: `firmware/firmware.ino`, `firmware/CY8C9560.cpp`,
  `firmware/CY8C9560.h`.
- Official docs checked:
  - PJRC Teensy 4.1: https://www.pjrc.com/store/teensy41.html
  - PJRC Teensy UART: https://www.pjrc.com/teensy/td_uart.html
  - PJRC Teensy Wire/I2C: https://www.pjrc.com/teensy/td_libs_Wire.html
  - u-blox NEO-M8 data sheet:
    https://content.u-blox.com/sites/default/files/NEO-M8_DataSheet_%28UBX-13003366%29.pdf
  - u-blox NEO-M8 hardware integration manual:
    https://content.u-blox.com/sites/default/files/NEO-M8_HardwareIntegrationManual_%28UBX-13003557%29.pdf
  - Infineon CY8C95xxA data sheet:
    https://www.infineon.com/assets/row/public/documents/30/57/infineon-cy8c9520a-cy8c9540a-cy8c9560a-20--40--and-60-bit-i-o-expander-with-eeprom-datasheet-additionaltechnicalinformation-en.pdf
  - Analog Devices MAX2679 product page:
    https://www.analog.com/en/products/max2679.html
  - Analog Devices MAX2679 data sheet:
    https://www.analog.com/media/en/technical-documentation/data-sheets/MAX2679-MAX2679B.pdf

## Bugs

1. GPS UART is wired straight-through instead of crossed.
   Netlist: `UBX-TXD` connects U3 TXD/SPI_MISO to Teensy pin 1/TX1, and
   `UBX-RXD` connects U3 RXD/SPI_MOSI to Teensy pin 0/RX1. For UART, module TX
   must go to MCU RX and module RX must go to MCU TX, so `Serial1` cannot
   receive GPS data.

2. The CY8C9560 I2C SDA pull-up is a pull-down.
   Netlist: R3.1 is `GND`, R3.2 is `CY_SDA`. SCL has R2 to `+3.3V`, but SDA is
   held low through 4.7k, so the I2C bus is stuck.

3. The RGB LED has no current-limiting resistors.
   Netlist: D3 common anode is tied to `+3.3V`; D3 blue/green/red cathodes go
   directly to Teensy pins 7/6/5. That can overcurrent the LEDs and MCU pins.

4. The MAX2679 LNA is overvolted.
   Netlist: U5 VCC is on `Net-(U3-VCC_RF)`, driven from NEO-M8 `VCC_RF`. The
   MAX2679 operates from 1.08 V to 1.98 V and has a 2.2 V absolute VCC maximum,
   while the NEO-M8 design powers the receiver at 3.3 V and exposes VCC_RF as an
   RF supply rail.

5. The MAX2679 `RFOUT/SHDNB` pin is not biased high to enable the LNA.
   Netlist: U5.A2 only connects to L1. The MAX2679 data sheet shows
   `RFOUT/SHDNB` as both RF output and shutdown control; the typical application
   biases it with 25k to VCC. Here it floats through the RF path, so the LNA can
   remain shut down or unstable.

6. The MAX2679 input matching/DC-block network is missing at the antenna input.
   Netlist: AE1 antenna connects directly to U5.B1/RFIN. The MAX2679 data sheet
   requires off-chip input matching using an inductor in series with a
   DC-blocking capacitor.

7. The only visible 12 nH RF inductor is on the MAX2679 output path, not the
   input path where the data sheet requires the matching inductor.
   Netlist: U5.A2 -> L1 -> C5 -> U3.RF_IN. That puts L1 after RFOUT instead of
   in front of RFIN.

8. The NEO-M8 `SAFEBOOT_N` service pin is routed to a Teensy GPIO.
   The u-blox NEO-M8 pin table describes `SAFEBOOT_N` as reserved/service and
   says to leave it open. A firmware or boot-time GPIO mistake can hold it low,
   which starts safe boot mode instead of GNSS operation.

9. The schematic/firmware assume cable pins map contiguously to CY8C9560 bits,
   but `CBL_20` through `CBL_27` are actually on CY bits 24 through 31.
   The firmware drives bit 20 for harness pin 20; hardware routes pin 20 to
   GPort3_Bit0, which is input register bit 24.

10. `CBL_28` through `CBL_35` are mapped four bits higher than firmware expects.
    Hardware routes them to CY bits 32 through 39, while firmware treats them as
    bits 28 through 35.

11. `CBL_36` through `CBL_39` are routed beyond the firmware's tested bit range.
    Hardware routes them to CY bits 40 through 43, but the firmware loops
    `j < NUM_HARNESS_PINS` and only evaluates bits 0 through 39.

12. The PCB has an actual open on `+3.3V`.
    KiCad DRC reports one unconnected item: two `+3.3V` F.Cu tracks at about
    `(167.320101, 35.774840)` and `(158.100000, 32.900000)` are missing a
    connection.

13. The PCB violates its own minimum track width on 199 tracks.
    DRC: board setup minimum width is 0.2000 mm; actual traces are 0.1270 mm on
    many cable, LED, RF, and power nets.

14. The PCB violates its own clearance rule hundreds of times.
    DRC reports 481 clearance violations at 0.1505 mm against a 0.2000 mm rule,
    mostly signal or power tracks/vias against the GND and +3.3V pours.

15. Power nets are involved in copper-clearance failures.
    DRC includes `+12V`, `+5V`, `+3.3V`, and `Net-(U3-VCC_RF)` clearance
    violations to copper pours. These are not only low-speed signal issues.

16. RF nets are involved in copper-clearance and width failures.
    DRC flags `Net-(AE1-A)`, `Net-(U3-RF_IN)`, and `Net-(U3-VCC_RF)`. The
    u-blox integration manual calls out careful RF layout and 50 ohm antenna
    routing, so these rule failures directly affect GPS reception.

17. Copper is too close to the board edge and mounting/mechanical features.
    DRC reports copper-edge violations at J1 NPTH pads and U2 pads, including
    U2 pad 48 on `+5V`, below the 0.5000 mm edge rule.

18. The board has isolated copper islands on GND and +3.3V pours.
    DRC reports 32 isolated copper fills. Floating islands do not provide useful
    return paths and can couple noise into the GPS/RF section.

19. C6 and U5 courtyards overlap.
    DRC reports a courtyard overlap between the MAX2679 WLP and its capacitor,
    which is an assembly/placement error in the RF section.

20. SW1 schematic and PCB footprint disagree.
    DRC schematic parity reports no PCB pad for schematic SW1 pin 4
    (`BTN_TEST`) and no PCB pad for schematic SW1 pin 3 (`GND`). The switch
    still has some duplicated pad names, but the schematic and footprint are not
    a one-to-one match.

21. Firmware never configures LED pins as outputs.
    `set_status()` calls `digitalWrite()` on pins 5, 6, and 7, but `setup()`
    never calls `pinMode(..., OUTPUT)` for those pins. On Arduino/Teensy this
    toggles input pullups instead of driving the common-anode LED cathodes.

22. Firmware never configures GPS reset/safeboot pins as outputs.
    `setup()` calls `digitalWrite(PIN_UBX_SAFEBOOT, LOW)` and
    `digitalWrite(PIN_UBX_RST_N, HIGH)` without `pinMode(..., OUTPUT)`, so those
    intended control writes do not drive the NEO-M8 pins.

23. Firmware never calls `cy.begin()`.
    The CY8C9560 object is constructed, but `setup()` never initializes it, never
    starts `Wire2`, never verifies the ID, and never configures/reset-releases
    the expander before using it.

24. The CY8C9560 reset sequence leaves reset asserted.
    `CY8C9560::begin()` drives `CY_RST` high, then low, then returns while it is
    still low. The hardware net is `CY_RST_N`, so the active-low reset remains
    asserted even if `begin()` is added.

25. The harness test uses 32-bit shifts for 40 pins.
    `uint64_t output_mask = 1 << i` and `(values & (1 << j))` use an `int` left
    operand. Pins 32 through 39 overflow or invoke undefined behavior instead of
    setting 64-bit masks.

26. Each selected output is immediately changed back to an input.
    In the loop, firmware calls `cy.set_output(output_mask, output_mask)` and
    then `cy.set_pd_inputs(~output_mask)`. `set_pd_inputs()` writes
    `REG_PIN_DIRECTION` to `0xFF` for every port, so the selected driven pin is
    no longer an output when inputs are read.

27. The CY8C9560 helper functions clobber whole ports instead of selected pins.
    `set_output()` writes `REG_PIN_DIRECTION = 0x00` for every port, while
    `set_pd_inputs()`/`set_pu_inputs()` write `0xFF` for every port. They do not
    preserve directions for nonselected pins or clear stale drive-mode bits.

28. The pass/fail aggregation is wrong.
    `passed` starts false and is set true if any one row matches
    `EXPECTED_CONNECTIONS[i]`. It is never set false on a mismatch, so one
    correct pin can make a bad harness pass.

29. The start button logic is inverted.
    Hardware has `BTN_TEST` pulled up by R4 to `+3.3V` and SW1 pulls it to GND.
    Firmware returns when the input is LOW, so it does not test while pressed
    and instead tests when the button is not pressed.

30. The NMEA receive buffer can overflow.
    `nmea_buf` is 64 bytes, `nmea_idx++` is unchecked, and `process_nmea()` then
    writes `buf[len] = 0`. Standard NMEA sentences can be longer than this.

31. The GPS lock check accepts invalid RMC sentences.
    `process_nmea()` ignores the RMC status field (`A` valid versus `V` invalid)
    by parsing it with `%*c`. It sets `time_fixed = true` even when the receiver
    explicitly reports no valid fix.

32. The GPS parser does not validate the NMEA checksum.
    Any corrupted or partial `$GPRMC` line with parseable fields can set the
    device to ready and provide the logged date/time.

33. The firmware is level-triggered and logs repeatedly while the button is in
    the active state.
    There is no edge detection or debounce around `PIN_BTN_TEST`; combined with
    the inverted logic, it can run and append results continuously while idle.

## Count

Total distinct bugs listed: 33.
