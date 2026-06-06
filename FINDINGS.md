# Harness Tester Challenge Hardware Findings

Target: https://github.com/commaai/harness_tester_challenge at `069f724`.

This is a strict hardware list. It does not count firmware bugs, bulk ERC/DRC
output, cosmetic silk issues, generic width/clearance rule violations, or
"would be nicer" layout recommendations.

## Verification Sources

- KiCad 10.0.3 schematic netlist exported from
  `kicad_files/hardware_challenge.kicad_sch`.
- KiCad 10.0.3 PCB DRC with schematic parity enabled on
  `kicad_files/hardware_challenge.kicad_pcb`.
- PJRC Teensy 4.1 documentation:
  https://www.pjrc.com/store/teensy41.html
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

10. The MAX2679 RFIN input matching inductor is missing.
    The MAX2679 input network needs a series inductor with the RFIN
    DC-blocking capacitor. The antenna feed goes straight into U5 B1/RFIN
    with no series input inductor on that side of the LNA.

11. The MAX2679 enable/shutdown pin has no DC bias.
    U5 A2 is the combined `RFOUT/SHDNB` pin. The data sheet says the IC is
    enabled by applying a logic high through a 25k resistor and disabled by
    applying a logic low through a 25k resistor. This board connects A2 only
    through L1 to C5, and C5 blocks DC, so the enable input is left floating.

12. The MAX2679 output match is corrupted by L1.
    MAX2679 `RFOUT/SHDNB` is internally matched to 50 ohms, but the board
    inserts the only 12 nH inductor, L1, in series between U5 A2/RFOUT and
    the NEO-M8 `RF_IN` coupling capacitor. That puts the input-matching part
    in the output path where it does not belong.

13. The GNSS antenna tuning footprint is missing.
    TE/Linx recommends at least a 3-element pi matching network footprint in
    all ANT-GNSSCP-TH25L1 designs, with the series element populated as 0
    ohms and the shunt capacitors DNP if no tuning is needed. The board
    routes AE1 directly into U5 RFIN, leaving no antenna-side tuning option.

14. The GNSS RF routing is not a controlled 50 ohm RF path.
    u-blox requires a controlled 50 ohm route to `RF_IN`, and MAX2679 requires
    controlled-impedance lines on high-frequency inputs and outputs. The board
    uses inconsistent RF trace widths through the antenna/LNA/module path
    instead of one defined transmission-line geometry.

15. The reverse-polarity protection is bypassed by D1.
    D1 is a unidirectional SMAJ16A from the raw input jack node to ground and
    is placed before the PMOS reverse-protection stage. With reverse input
    polarity, D1 is forward biased directly across the supply, so the PMOS does
    not provide controlled reverse-polarity protection.

16. The PMOS gate clamp is the wrong device type.
    D2 is `PMEG10020ELR`, a Schottky rectifier, connected between the PMOS gate
    and source where a VGS clamp would need to limit negative gate-source
    voltage. It does not clamp the PMOS gate during positive input transients.

17. The input surge clamp can overvoltage the PMOS gate oxide.
    Q1 is rated for +/-20 V gate-source voltage. D1 is a SMAJ16A with a
    26 V maximum clamp voltage, and the PMOS gate is held near ground by R1
    while the source follows the protected `+12V` rail. During the positive
    transient D1 is meant to absorb, Q1 can see about -26 V VGS because D2 is
    not a Zener-style gate clamp.

18. The PMOS gate pull-down is overstressed.
    R1 is `1k` in an 0402 footprint from the PMOS gate to ground. At a normal
    12 V input it dissipates 144 mW continuously, and at 14.4 V it dissipates
    about 207 mW.

19. The Teensy external-power integration does not isolate VIN from USB power.
    The carrier board powers Teensy VIN from the 5 V regulator. PJRC documents
    that VIN and VUSB are connected unless the underside pads are cut, and that
    VIN should not be powered while USB is connected. This board has no carrier
    isolation for that required external-power case.

20. The harness pins have no current limiting or protection.
    Every J3 cable pin connects directly to a CY8C9560 GPIO. The CY8C9560
    absolute maximum is 50 mA into any port pin, with normal output ratings of
    10 mA source and 25 mA sink per pin. A harness tester must tolerate shorted
    pins while it drives and senses them, but this board provides no series
    resistance, switched current limit, or input protection between the cable
    fault and the expander.

21. The PCB has a real open on `+3.3V`.
    KiCad reports one missing connection on `+3.3V` between F.Cu track islands
    near `(167.320101, 35.774840)` and `(158.100000, 32.900000)`. This is an
    actual split power net, not a repeated clearance/width rule violation.

## Count

Total strict hardware bugs listed: 21.
