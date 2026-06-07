# Harness Tester Challenge Hardware Findings

Target: https://github.com/commaai/harness_tester_challenge at `069f724`.

This is a strict hardware list. It does not count firmware bugs, bulk ERC/DRC
output, cosmetic silk issues, generic width/clearance rule violations, or
"would be nicer" layout recommendations.

## Verification Sources

- KiCad 10.0.3 schematic netlist exported from
  `kicad_files/hardware_challenge.kicad_sch`.
- KiCad 10.0.3 PCB layer inspection and DRC with schematic parity enabled on
  `kicad_files/hardware_challenge.kicad_pcb`.
- PJRC Teensy 4.1 documentation:
  https://www.pjrc.com/store/teensy41.html
- STMicroelectronics L78 data sheet:
  https://www.st.com/resource/en/datasheet/l78.pdf
- u-blox NEO-M8 data sheet:
  https://content.u-blox.com/sites/default/files/NEO-M8_DataSheet_%28UBX-13003366%29.pdf
- u-blox NEO-M8 hardware integration manual:
  https://content.u-blox.com/sites/default/files/NEO-M8_HardwareIntegrationManual_%28UBX-13003557%29.pdf
- TE Connectivity / Linx ANT-GNSSCP-TH25L1 product page and data sheet:
  https://www.te.com/en/product-ANT-GNSSCP-TH25L1.html
- Infineon CY8C95xxA data sheet:
  https://www.infineon.com/assets/row/public/documents/30/49/infineon-cy8c9520a-cy8c9540a-cy8c9560a-20--40--and-60-bit-i-o-expander-with-eeprom-datasheet-en.pdf
- Analog Devices MAX2679 product page and data sheet:
  https://www.analog.com/en/products/max2679.html
  https://www.analog.com/media/en/technical-documentation/data-sheets/MAX2679-MAX2679B.pdf
- Broadcom ASMB-KTF0-0A306 data sheet:
  https://docs.broadcom.com/doc/ASMB-KTF0-0A306-DS100
- Vishay SiSS27DN data sheet:
  https://www.vishay.com/docs/62847/siss27dn.pdf
- Nexperia PMEG10020ELR data sheet:
  https://assets.nexperia.com/documents/data-sheet/PMEG10020ELR.pdf
- Littelfuse SMAJ16A product page:
  https://www.littelfuse.com/products/overvoltage-protection/tvs-diodes/surface-mount/smaj/smaj16a

## Hardware Bugs

1. The CY8C9560A footprint is the wrong physical package.
   U4 is `CY8C9560A-24AXIT`, which is a 100-pin TQFP 14 mm x 14 mm package
   with 0.5 mm pitch. The PCB assigns `Package_QFP:TQFP-100_12x12mm_P0.4mm`.
   The expander cannot be assembled on that footprint.

2. The CY8C9560 reset pin is represented with the wrong polarity.
   The selected device pin is active-high `XRES` with an internal pull-down.
   The schematic symbol and net call the same pin `RESET_N` / `CY_RST_N`,
   which is an active-low reset interface. That is a hardware symbol/net
   contract error on U4 pin 62.

3. The CY8C9560 I2C SDA line is pulled down.
   R2 pulls `CY_SCL` to `+3.3V`, but R3 connects `CY_SDA` to `GND`. SDA is
   therefore held low through 4.7k and the I2C bus cannot work normally.

4. The GPS UART TX/RX nets are straight-through instead of crossed.
   `UBX-TXD` connects NEO-M8 pin 20 `TxD` to Teensy pin 1 `TX1`, and
   `UBX-RXD` connects NEO-M8 pin 21 `RxD` to Teensy pin 0 `RX1`. UART TX must
   go to the other device's RX.

5. NEO-M8 `SAFEBOOT_N` is routed to the Teensy.
   U3 pin 1 connects to `UBX-SAFEBOOT` and then to a Teensy GPIO. The NEO-M8
   data sheet marks this pin as reserved/service and says to leave it open for
   NEO-M8N/Q/M variants. Holding it low at startup enters Safe Boot Mode instead
   of GNSS operation.

6. The RGB LED has no current-limiting resistors.
   D3's common anode is tied directly to `+3.3V`; the red, green, and blue
   cathodes go directly to Teensy GPIO nets. Each LED channel needs a current
   limiter.

7. The RGB LED red and blue channels are swapped.
   The ASMB-KTF0-0A306 data sheet lists pin 2 as red cathode and pin 4 as blue
   cathode. The design connects D3 pin 2 to `LED_B` and D3 pin 4 to `LED_R`.

8. The MAX2679 LNA is powered from an overvoltage rail.
   U5 VCC is tied to NEO-M8 `VCC_RF`. With the receiver powered at 3.3 V,
   `VCC_RF` is approximately the receiver supply. MAX2679 operation is
   specified only from 1.08 V to 1.98 V, with a 2.2 V absolute maximum.

9. MAX2679 RFIN is missing its required DC-blocking capacitor.
   The MAX2679 pin description requires a DC-blocking capacitor and external
   matching components at RFIN. The PCB connects AE1 directly to U5 B1/RFIN.

10. The MAX2679 input matching inductor is on the wrong side of the LNA.
    The MAX2679 input network needs a series inductor with the RFIN
    DC-blocking capacitor, while RFOUT is already internally matched to
    50 ohms. The only 12 nH inductor, L1, is instead connected between U5
    A2/RFOUT and the NEO-M8 RF input coupling capacitor.

11. The GNSS RF routing is not a ground-referenced 50 ohm path.
    u-blox requires a controlled 50 ohm route to `RF_IN`, Analog requires
    controlled-impedance lines on MAX2679 high-frequency inputs and outputs,
    and TE/Linx recommends a microstrip trace over a ground plane. This board
    routes the top-layer RF trace over a `+3.3V` inner plane and a copper void
    under the antenna, with no continuous RF ground reference.

12. The GNSS patch antenna has no ground-plane counterpoise underneath it.
    The ANT-GNSSCP-TH25L1 data sheet says ceramic patch antennas require a
    ground plane for proper operation and shows a bottom-layer ground plane for
    counterpoise. The PCB clears the plane copper from the antenna footprint on
    every copper layer instead of providing that local ground plane.

13. The reverse-polarity protection is bypassed by D1.
    D1 is a unidirectional SMAJ16A from the raw input jack node to ground and
    is placed before the PMOS reverse-protection stage. With reverse input
    polarity, D1 is forward biased directly across the supply, so the PMOS does
    not provide controlled reverse-polarity protection.

14. The PMOS gate clamp is the wrong device type.
    D2 is `PMEG10020ELR`, a Schottky rectifier, connected between the PMOS gate
    and source where a VGS clamp would need to limit negative gate-source
    voltage. It does not clamp the PMOS gate during positive input transients,
    so the SMAJ16A's 26 V maximum clamp event can exceed Q1's +/-20 V VGS
    rating.

15. The Teensy external-power integration does not isolate VIN from USB power.
    The carrier board powers Teensy VIN from the 5 V regulator. PJRC documents
    that VIN and VUSB are connected unless the underside pads are cut, and that
    VIN should not be powered while USB is connected. This board has no carrier
    isolation for that required external-power case.

16. The RGB LED anode is physically disconnected on the PCB.
    KiCad reports one missing connection on `+3.3V` between the D3 anode branch
    at `(158.100000, 32.900000)` and the rest of the rail near
    `(167.320101, 35.774840)`. This is an actual open in the LED power path,
    not a repeated clearance/width rule violation.

17. The 12 V to 5 V linear regulator is thermally undersized.
    U1 is an L7805 in a DPAK/TO-252 footprint, dropping the nominal 12 V input
    to 5 V for the Teensy. ST lists DPAK junction-to-ambient thermal resistance
    as 100 C/W without adequate heatsinking. At only 100 mA of 5 V load, the
    regulator must burn about 0.7 W from 12 V input and about 0.94 W from a
    14.4 V automotive supply. That is a 70 C to 94 C junction rise before adding
    the Teensy, GNSS, expander, and LED load margin.

18. The NEO-M8 main supply pin has no local input capacitor.
    U3 pin 23 `VCC` and pin 22 `V_BCKP` connect to `+3.3V`, but the only
    capacitors on that rail are at the Teensy/CY8C area. The nearest `+3.3V`
    capacitors, C3 and C4, are roughly 24 mm and 30 mm from U3 `VCC`; C6 is on
    `VCC_RF`, not on `VCC`. u-blox calls for a clean, stable VCC supply and low
    ESR capacitance at the module input to handle startup current peaks.

19. The external GNSS LNA is placed on the receiver side of the RF path, not at
    the passive antenna.
    u-blox says an external LNA is only needed when the passive antenna is far
    away, and in that case it must be placed close to the passive antenna. On
    this PCB the AE1 feed pin is at about `(129.5, 77.0)`, while U5 RFIN is at
    about `(148.0, 68.6)`, leaving roughly 20 mm of unamplified patch-antenna
    trace before the LNA and placing U5 closer to the NEO-M8 RF input than to
    the antenna feed.

20. Harness signal traces run under the GNSS patch antenna.
    The PCB routes `CBL_37`, `CBL_38`, and `CBL_39` on `In1.Cu` inside the
    25 mm x 25 mm AE1 patch antenna body. u-blox warns that passive antennas
    need extra RF-layout care and that weakly shielded PCB lines and unshielded
    connector lines are critical EMI sources for GNSS receivers. These harness
    traces sit in the antenna field/counterpoise area instead of being kept away
    from it.

21. The passive GNSS patch feed lacks RF-input ESD protection.
    AE1 is a passive patch antenna connected directly into the MAX2679 RFIN node
    and then to the NEO-M8 RF input. u-blox warns that exposed antenna areas and
    passive antenna patches can discharge through the receiver RF input, and says
    passive patch designs should add ESD measures such as an LNA with an
    appropriate ESD rating. The MAX2679 data sheet does not specify that kind of
    protected antenna input, and this board has no low-capacitance RF ESD device
    ahead of the receiver.

## Count

Total strict hardware bugs listed: 21.
