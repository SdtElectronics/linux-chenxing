# IPL blobs

## First stage IPL loaded by the bootrom

This is usually copied from memory mapped SPI NOR into the internal SRAM (IMI).

- First 4 bytes are an ARM instruction to jump somewhere else in the IPL binary like a reset vector.
   This also seems to be generally used to switch from ARM to Thumb execution.
- Second 4 bytes are "IPL_"
- Next 2 bytes are the size of the IPL
- Next byte (byte 10) is an optional CID (chip ID?) that will be checked against value stored at address 0x1f00700c, if that value is non-zero (it is zero on MSC313e). If those do not match, an error "CID check failed! [HALT]" will be printed by BootROM and the CPU will go into infinite loop.
- If next byte (byte 11) has value 0xFA, the authentication of the image will be run. If this authentication fails, "Authenticate failed! [HALT]" will be printed by BootROM and CPU will go into infinite loop. Any other value does not trigger authentication unless bit 0 of the value stored at 0x1f0071c0 is 1.
- Forth 4 bytes (bytes 12-15) are a checksum of the upcoming data. The checksum is just each u32 of the data added together. The boot rom doesn't seem to actually check this and not all IPL version check it before loading the second stage IPLC.

### mercury5 notes

- This *seems* to be located at 0x1000 in SPI NOR or the file IPL on an SD card in a fat16 partition.
- The vendor IPL loads a kernel image from the file RTS on an SD card in a fat16 partition. It seems to load to 0x20008000 (in DDR) and jumps there.

### infinity3 notes

- Located at 0x4000 in SPI NOR.
- At least infinity BootROM accepts the image that does not have "IPL_" string at offset 0x4. In this case it will not take the size of the image from the halfword at offset 0x8 but will assume it is 0x9f00 and will continue the boot process.

### authentication notes

In case authentication is performed, a 256 bytes are expected just after normal IPL image. Those bytes are passed through crypto peripheral (0x1f224480 - 0x1f2244af) starting from the last byte till the first one. 256 bytes value seams to be computed out of which 32 last are taken.
A SHA256 value of the IPL image is calculated by other part of crypto engine (0x1f224400 - 0x1f22445f) and its result is compared with the hash explained above.

### deep sleep notes

The IPL checks the 0x1f001c48 regiter value and if it is 0xBABE,
the board may be coming back from deep sleep, otherwise it will determine the type of reset (WDT, HW or SW). 
If 0xBABE is found:
* PM_GPIO4 is configured as input
* 0x1f001cdc and 0x1f001c48 are set to 0
* bits 2 and 3 of 0x1f001c70 are cleared
* bit ~~31~~ 15 of 0x1f001c24 is checked, if it is ~~1~~ 0, a return from deep sleep is peformed: (I think it's bit 15, all of the registers are only 16 bits wide; true, I didn't notice there is SXTH there, also the logic is the other way around after all)
  * ... TBD
  * UTMI (USB PHY) is configured
  * CPUCLK and L3 bridge is configured ... TBD
  * clkgen muxes are configured ... TBD
  * MIU is configured .. TBD
  * registers 0x1f20248c, 0x1f2024cc, 0x1f20250c, 0x1f20254c, 0x1f20220c, 0x1f20224c, 0x1f2023cc are zeroed
  * register 0x1f20243c is written 0x8c08
  * the unknown peripheral at 0x1f0040XX is being configured (efuse)
  * the jump address is taken from 0x1f001cf0 and 0x1f001cec
  
  https://github.com/fifteenhex/linux_mstar_3.18/blob/taobao_sdk/arch/arm/mach-mstar/infinity3/sram.S
  
  https://github.com/fifteenhex/linux_mstar_3.18/blob/taobao_sdk/drivers/mstar/pm/ms_pm.c

#### Version capability/feature matrix

| version               | msc313 | msc313e | msc316dc | checks IPLC chksum | notes |
|-----------------------|--------|---------|----------|--------------------|-------|
| I3g0c706bbIPL_QFN64MW | y      | y       | n[0]     | n                  |       |
| I3gd156225IPL_EFUSE   | ?      | y       | y        | y                  |[1]    |

- 0 - Runs but can't init the DRAM
- 1 - Makes extensive use of magic numbers in the efuse area, this version can apparently
      detect if the included memory is DDR2 or DDR3

### infinity6 notes

- Located at 0x0 in SPI NOR.

#### Version capability/feature matrix

| version               | ssc325 | checks IPLC chksum | notes |
|-----------------------|--------|--------------------|-------|
| I6gf1d3932IPL         | y      |                    |       |

## Second stage IPL loaded by the first stage IPL

The header format is the same as the first stage IPL but the second 4 bytes are "IPLC"




