# Harness Tester Challenge Hardware Findings

Target: https://github.com/commaai/harness_tester_challenge at `069f724`.

This is a hardware-only list. It intentionally excludes firmware-only bugs and
does not count repeated ERC/DRC violations as separate findings. KiCad ERC/DRC
was used only to confirm concrete schematic/PCB facts such as an open net,
footprint mismatch, or placement conflict.

## Verification Sources

- KiCad 10.0.3 schematic netlist exported from
  `kicad_files/hardware_challenge.kicad_sch`.
- KiCad PCB DRC with schematic parity enabled on
  `kicad_files/hardware_challenge.kicad_pcb`.
- PJRC Teensy 4.1 docs:
  https://www.pjrc.com/store/teensy41.html
- u-blox NEO-M8 data sheet:
  https://content.u-blox.com/sites/default/files/NEO-M8_DataSheet_%28UBX-13003366%29.pdf
- u-blox NEO-M8 hardware integration manual:
  https://content.u-blox.com/sites/default/files/NEO-M8_HardwareIntegrationManual_%28UBX-13003557%29.pdf
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
- TE/Linx ANT-GNSSCP-TH25L1 product page and data sheet:
  https://www.te.com/en/product-ANT-GNSSCP-TH25L1.html
  https://www.te.com/commerce/DocumentDelivery/DDEController?Action=srchrtrv&DocFormat=pdf&DocLang=English&DocNm=ant-gnsscp-th25l1-ds&DocType=Data+Sheet&PartCntxt=ANT-GNSSCP-TH25L1

## Hardware Bugs

1. The CY8C9560A footprint is the wrong physical package.
   U4 is assigned `Package_QFP:TQFP-100_12x12mm_P0.4mm`, but
   `CY8C9560A-24AXIT` is a 100-pin TQFP 14 mm x 14 mm package with 0.5 mm
   pitch. The expander cannot be assembled on this PCB footprint.

2. The GPS UART is wired straight-through instead of crossed.
   `UBX-TXD` connects NEO-M8 pin 20 `TxD` to Teensy pin 1 `TX1`, and
   `UBX-RXD` connects NEO-M8 pin 21 `RxD` to Teensy pin 0 `RX1`. UART TX must
   connect to the other device's RX.

3. The CY8C9560 I2C SDA line is pulled down instead of pulled up.
   R2 correctly pulls `CY_SCL` to `+3.3V`, but R3 connects `CY_SDA` to `GND`.
   That holds SDA low and prevents normal I2C communication.

4. The RGB status LED has no current-limiting resistors.
   D3 pin 1 is tied directly to `+3.3V`; D3 pins 2/3/4 go directly to Teensy
   GPIO nets. Each LED channel needs a current limiter.

5. The RGB LED red and blue channels are swapped against the selected part.
   The ASMB-KTF0-0A306 data sheet lists pin 2 as red cathode and pin 4 as blue
   cathode. The schematic/PCB connect D3 pin 2 to `LED_B` and pin 4 to
   `LED_R`, so the hardware status colors are wrong.

6. The CY8C9560 reset pin is given the wrong polarity in the schematic.
   The Infineon data sheet describes the external reset as active high `XRES`
   with an internal pull-down. The design labels and routes it as
   `CY_RST_N`/`RESET_N`, an active-low reset. That is a hardware symbol/pin
   function error independent of firmware behavior.

7. The MAX2679 LNA is powered from an overvoltage rail.
   U5 VCC is tied to `Net-(U3-VCC_RF)`. With the NEO-M8 powered at 3.3 V,
   `VCC_RF` is approximately `VCC - 0.1 V`, while MAX2679 operation is
   specified for 1.08 V to 1.98 V.

8. MAX2679 RFIN has no required DC-blocking capacitor.
   The data sheet says RFIN requires a DC-blocking capacitor and external
   matching components. In this design AE1 connects directly to U5 B1/RFIN.

9. MAX2679 RFIN has no input matching inductor.
   The MAX2679 typical input network uses a series inductor with the
   DC-blocking capacitor. There is no inductor between the antenna and U5
   B1/RFIN.

10. The 12 nH RF inductor is on the wrong side of the LNA.
    L1 is connected from U5 A2/RFOUT to C5 and the NEO-M8 RF input. The
    MAX2679 output is internally matched to 50 ohms; the external 12 nH
    inductor belongs in the RFIN input matching network, not in the output
    path.

11. The RF coupling capacitor value is not a GNSS RF value.
    C5 is `100n` in the 1.575 GHz path from the LNA output to NEO-M8 RF_IN.
    The MAX2679 application material uses pF/nF RF capacitors, including
    1000 pF parts; a generic 100 nF 0402 capacitor is not an appropriate RF
    coupling/matching part at GNSS frequency.

12. The GNSS RF routing is not a single controlled 50 ohm geometry.
    The NEO-M8 manual requires a controlled 50 ohm connection to RF_IN. The
    antenna/LNA/module route changes width from 0.8128 mm to 0.127 mm to
    0.508 mm, and the nearest internal plane under the top RF route is the
    `+3.3V` plane, not a continuous RF ground reference.

13. NEO-M8 `SAFEBOOT_N` is routed to a GPIO even though the data sheet says to
    leave it open.
    U3 pin 1 is connected to `UBX-SAFEBOOT` and then to a Teensy GPIO. The
    NEO-M8 pin table describes this pin as reserved/service for future
    service, updates, and reconfiguration, with `leave OPEN` guidance.

14. Reverse-polarity protection is defeated by the unidirectional TVS location.
    D1 is a unidirectional SMAJ16A from the raw input jack node to ground and
    is placed before the reverse-protection PMOS. On a reverse-polarity input,
    that TVS is forward biased directly across the supply.

15. The input TVS has no upstream fuse or current-limiting element.
    D1 is the transient/reverse-energy shunt for the 12 V input, but the design
    has no fuse, PTC, or other series current limiter ahead of it. A sustained
    reverse connection or surge can destroy the TVS or copper instead of
    producing controlled protection.

16. The PMOS gate clamp part is the wrong device type.
    D2 is `PMEG10020ELR`, a 100 V Schottky rectifier. It is connected between
    the PMOS gate and source where a VGS clamp would normally be a Zener/TVS.
    It does not clamp negative VGS during positive input transients.

17. The PMOS gate pull-down resistor is overstressed in the chosen footprint.
    R1 is `1k` in a 0402 footprint from the PMOS gate to ground. At a normal
    12 V input it dissipates about 144 mW, and at 14.4 V it dissipates about
    207 mW, which is too much for an ordinary 0402 gate-bias resistor.

18. The Teensy external-power path can backfeed USB.
    The board powers Teensy VIN from the 5 V regulator, but PJRC documents that
    Teensy VIN and VUSB are connected unless the cut pads are separated. The
    carrier design does not isolate VUSB from VIN, so plugging in USB while the
    tester is externally powered can backfeed the host computer.

19. The PCB has a real open on the `+3.3V` rail.
    KiCad DRC reports one missing connection on `+3.3V` between F.Cu track
    islands near `(167.320101, 35.774840)` and `(158.100000, 32.900000)`.
    That is a split power rail, not a repeated clearance violation.

20. The MAX2679 and C6 courtyards overlap.
    KiCad DRC reports a courtyard overlap between U5 and C6. This is a concrete
    placement/assembly error in the RF section.

21. The GNSS patch antenna is placed with zero board-edge clearance.
    The AE1 footprint is centered so its nominal 25 mm body reaches the board
    edge at x = 119.5 mm. The ANT-GNSSCP-TH25L1 data sheet body is 25.1 mm, so
    the real component has essentially no edge clearance and can overhang the
    PCB.

22. The GNSS patch antenna has essentially no placement clearance to the NEO-M8
    module.
    AE1's 25.1 mm body extends to about x = 144.55 mm, while the U3 NEO-M8
    courtyard starts at about x = 144.59 mm. A ceramic patch antenna and the
    GNSS module should not be placed with only about 0.04 mm nominal clearance.

## Count

Total proper hardware bugs listed: 22.
