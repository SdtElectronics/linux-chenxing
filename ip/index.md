# IP blocks

## PM domain

Mostly a mystery. Hardware in this area is kept alive even when the chip is in deep sleep.
For infinity3 and mercury5 at least PM domain blocks are easy to see because they mapped 
to 0x1f00xxxx.


[Register map](pm_regmap.md)

### sleep intc

32 interrupts forwarded to the IRQ intc via a single interrupt. 
This is mostly for the pm gpio interrupts.

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

### PM GPIO

pm_gpio4 on infinity3 (maybe others) is weird and needs to be "unlocked" before it can be set.

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

### EFUSE

[more info](efuse.md)


## Interrupt controllers

[more info](intc.md)

### IRQ intc

64 interrupts forwarded to the GIC

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

### FIQ intc

32 interrupts forwarded to the GIC as FIQs

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |



## Pinmux

[More details](pinctrl.md)

### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | wip   |

## Clocks

See [clks](clks.md). 

## Bus Glue

### L3 Bridge/AXI interface

There is some fabric between the CPU, other bus masters and the memory.
Some magic bits in here are used to flush pending cpu writes to memory
so other bus masters can see them.

[Rough register descriptions](https://github.com/fifteenhex/SDK_pulbic/blob/master/Mercury5/proj/sc/driver/hal/mercury/kernel/inc/kernel_axi.h)

### MIU

MIU or "memory interface unit" is a multiport DDR controller that is wired to the CPU(s)
and DMA capable perpherials like USB, Ethernet and so on.

[More info](miu.md)

### RIU

RIU of "register interface unit" is a brige between the CPU and perpherial registers.
It's fairly straight forward for the most part with one annoying quirk; 32 bit registers
are split into two 16 bit locations that are spaced 4 bytes apart. This means that
any existing drivers that expect 32 bit registers aligned to 4 bytes needs to have a quirk
added to read the two 16 bit parts and stitch them back together.

For some peripherals there seems to be a second way to access the registers that uses the same
offsets as the RIU but presents 32bit wide registers. For example on i3 the emac is present
as split registers at ```0x1f2a2000``` and 32bit wide registers with the same offsets at ```0x1f343c00```.
For i1 only the split interface exists.

### IMI

IMI or "internal memory interface"? interface for embedded SRAM. It seems to have multiple
ports so the CPU and some perpherials are able to access the SRAM but not a lot is known about
that yet.

## Timers
Appart from the ARMv7 builtin timer, there are 3 SoC specific timer peripherals at 0x1f006040, 0x1f006080 and 0x1f0060c0. They are running at 12MHz. Each timer has its interrupt but they are not yet supported by the linux driver.

Registermap:

| Offset | Name  | Comment |
| ---    | ---   | ---     |
| 0x00   | CTRL  | bit0 - ~oe <BR> bit1 - trig <BR> bit3 - clear <BR> bit4 - capture <BR> bit8 - int |
| 0x08   | MAX_L | Low 16 bits of the max value |
| 0x0c   | MAX_H | High 16 bits of the max value |
| 0x10   | CNT_L | Low 16 bits of the counter |
| 0x14   | CNT_H | High 16 bits of the counter |


## RTC

Base address is 0x1f202400 (on infinity3). Can be a wakeup source, can generate alarm interrupt, does not seem to have an eeprom.

Registermap:

| Offset | Name  | Comment |
| ---    | ---   | ---     |
| 0x00   | CTRL  | bit fileds below |
| bit 0  | CTRL_SOFT_RSTZ | it is set to 1 by the driver, not sure what it means |
| bit 1  | CTRL_CNT_EN    | start the counter?? set to 1 in the probe of driver [0] |
| bit 2  | CTRL_WRAP_EN | ?? |
| bit 3  | CTRL_LOAD_EN | should read back in a loop to ensure HW latch, needed to set RTC_LOAD_VAL |
| bit 4  | CTRL_READ_EN | should read back in a loop to ensure HW latch, needed to read RTC_CNT_VAL |
| bit 5  | CTRL_INT_MASK | mask alarm interrupt  (when RTC_MATCH_VAL reached) |
| bit 6  | CTRL_FORCE | force an alarm interrupt maybe? |
| bit 7  | CTRL_INT_CLEAR |  set to clear the interrupt status |
| 0x04   | RTC_FREQ_CW_L  | low 16bits of clock frequency divider value |
| 0x08   | RTC_FREQ_CW_H  | high 16bits of clock frequency divider value |
| 0x0C   | RTC_LOAD_VAL_L | low 16bits for the load value (to change the current time), set LOAD_EN after changing this, the driver also zeroes this after the LOAD_EN_BIT was set |
| 0x10   | RTC_LOAD_VAL_H | hih 16bits for the load value (to change the current time), see the note above |
| 0x14   | RTC_MATCH_VAL_L | low 16bits of match value (for alarm) |
| 0x18   | RTC_MATCH_VAL_H | high 16bits of match value (for alarm) |
| 0x20   | RTC_CNT_VAL_L   | low 16bits of counter |
| 0x24   | RTC_CNT_VAL_H   | high 16bits of counter |

Note [0]: IPL expects this bit to be set after a software reset - otherwise it assumes a HW reset happened. The only difference is in the messages printed and the fact that bit 1 in RSTLEN of WDT is cleared in software reset case, though.

### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

## DMA

### BDMA

BDMA or "Byte DMA" is a simple A -> B DMA engine. It's mainly used to
move data from the memory mapped SPI NOR into main memory so the CPU
doesn't have to do it. It also apparently supports doing CRC calculations.

#### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

### CMDQ

CMDQ or "command queue" is a descriptor list based DMA engine that seems
to be intended to be used to tie the parts of the camera pipeline together
so that the CPU doesn't need to be involved.

Note: This seems to live in the "RIU" register address space and maybe can't see
the system memory.

#### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | wip   |
| infinity3 |        | wip   |

## Crypto

See [Crypto](crypto.md)

#### Support Matrix

| block | family     | u-boot | linux |
|-------|------------|--------|-------|
| RNG   | infinity   |        | yes   |
|       | infinity3  |        | yes   |

## Ethernet

### 100mbit phy

| offset | name                 | 15 | 14 | 13 | 12 | 11 | 10 | 9 | 8 | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 | default | working value | writeable | msc313 | msc313e |
|--------|----------------------|----|----|----|----|----|----|---|---|---|---|---|---|---|---|---|---|---------|---------------|-----------|--------|---------|
| 0x74   |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0xe4   | lpbk enable set to 0 |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         | 0x4a0         |           | x      | x       |
| 0x29c  | det max              |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         | 0x24c         |           | x      | x       |
| 0x2a0  | det min              |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         | 0x160         |           | x      | x       |
| 0x2e0  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x2ec  | snr len              |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         | 0x1800        |           | x      | x       |
| 0x368  | gain shift           |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         | 0x0002        |           | x      | x       |
| 0x374  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x388  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x398  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x3f8  | ldo power down?      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         | 0x0000        | 0xffff    | x      | x       |
| 0x460  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x470  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x500  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x514  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x540  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x588  |                      |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |
| 0x5e4  | 200 gat              |    |    |    |    |    |    |   |   |   |   |   |   |   |   |   |   |         |               |           |        | x       |

### Cadence EMAC

#### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  | yes    | yes   |
| infinity3 | yes    | yes   |

### GMAC

- infinity2m has something called a GMAC. Might be a Cadence GEM.

## USB

### UTMI

This is the USB PHY. It's probably a Faraday design to go with the Faraday EHCI host but this isn't confirmed.
This also supplies the clocks for the UHC and OTG so it's not safe to access them before enabling the clocks here first.

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

### BC

This seems to be a way of presenting the right resistor values on the data lines to trigger chargers into supplying more current.

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        |       |
| infinity3 |        |       |
| mercury5  |        |       |

### USBC

This seems to be essentially a mux that sits between the UTMI and the UHC and OTG blocks so that UTMI can be connected to the right block for the current role the port is in.

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

### UHC

This is a usb host controller that seems to be based on a Faraday design. It's a broken-EHCI controller.

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

### OTG

This seems to be an musb USB device controller.

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | wip   |
| infinity3 |        | wip   |
| mercury5  |        | wip   |

## ADC

### SAR

This seems to be a fairly standard ADC. Channel 7 is an on-die [temp sensor](https://github.com/fifteenhex/linux-ssc325/blob/89341c7012404c72e192f198b2ea6405ec80d15d/drivers/sstar/cpufreq/infinity3-cpufreq.c#L222).

[rough register descriptions](https://github.com/schreibikus/wenshuai.xi/blob/7906c437f42aabfc8ac2e103efac3e885030f0cd/Mstar_code/alkaid/sdk/mhal/i2/utpa/modules/sar/hal/i2/sar/regSAR.h)

#### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

#### Registers

*note* these seem to be slightly different for older chips. The listing here is for the infinity/mercury5.

| address | name          | 15     | 14      | 13 | 12 | 11      | 10      | 9       | 8       | 7     | 6          | 5            | 4                 | 3              | 2                   | 1                   | 0                   |
|---------|---------------|--------|---------|----|----|---------|---------|---------|---------|-------|------------|--------------|-------------------|----------------|---------------------|---------------------|---------------------|
| 0x0     | ctrl          |        | load en |    |[0] | nch en  | sel     | freerun | adc pd  | start | digital pd | mode         | single channel en | level trigger  | single channel mask | single channel mask | single channel mask |
| 0x4     | sample period |        |         |    |    |         |         |         |         |       |            | 0 - one shot |                   |                |                     |                     |                     |
|         |               |        |         |    |    |         |         |         |         |       |            | 1 - free run |                   |                |                     |                     |                     |
| 0x40    | reg_pm_dmy    |        |         |    |    |         |         |         |         |       |            |              |                   |                |                     |                     |                     |
| 0x44    | gpio ctrl     |        |         |    |    | CH3_OEN | CH2_OEN | CH1_OEN | CH0_OEN |       |            |              |                   | CH3_EN         | CH2_EN              | CH1_EN              | CH0_EN              |
| 0x48    | gpio data     |        |         |    |    | CH3_IN  | CH2_IN  | CH1_IN  | CH0_IN  |       |            |              |                   | CH3_OUT        | CH2_OUT             | CH1_OUT             | CH0_OUT             |   
| 0x50    | int mask      |        |         |    |    |         |         |         |         |       |            |              |                   |                |                     |                     |                     |
| 0x54    | int clear     |        |         |    |    |         |         |         |         |       |            |              |                   |                |                     |                     |                     |
| 0x58    | int force     |        |         |    |    |         |         |         |         |       |            |              |                   |                |                     |                     |                     |
| 0x5c    | int status    |        |         |    |    |         |         |         |         |       |            |              |                   |                |                     |                     |                     |
| 0x60    | cmp out rdy   |        |         |    |    |         |         |         |         |       |            |              |                   |                |                     |                     |                     |
| 0x64    | vref sel      |        |         |    |    |         |         |         |         |       | ch7        |              |                   |                |                     | ch1                 |                     |
|         |               |        |         |    |    |         |         |         |         |       | 0 - 2.0v   |              |                   |                |                     | 0 - 2.0v            |                     |
|         |               |        |         |    |    |         |         |         |         |       | 1 - avdd   |              |                   |                |                     | 1 - avdd            |                     |
| 0x118   | ch7 data      |        |         |    |    |         |         |         |         |       |            |              |                   |                |                     |                     |                     |
|         |               |        |         |    |    |         |         |         |         |       |            |              |                   |                |                     |                     |                     |


CH?_EN bits control if the pin is an ADC pin or GPIO, 1 = ADC, 0 = GPIO
CH?_OEN bits control the gpio direction, 1 = input, 0 = output
CH?_IN bits represent the gpio input level
CH?_OUT bits set the gpio output level

```
 * # devmem 0x1f0018bc
 * 	0x00000000
 * # devmem 0x1f001cbc
 *	0x00000000
 * # devmem 0x1f001d90 16 0x2400
 *
 * This seems to be power down for the temp sensor
 * # devmem 0x1f001cbc 16 0x4
```

## Serial

Serial seems to be a standard Designware UART with USR register in a different location (at offset 0x38). The usual 8250 registers are at offsets divisible by 8 (the reg-shift is 3). So:
- **0x00** - THR, RBR, DLL
- **0x08** - IER, DLH
- **0x10** - IIR, FCR
- **0x18** - LCR
- **0x20** - MCR
- **0x28** - LSR
- **0x30** - MSR
- **0x38** - USR

There seems to be some unknown register at offset 0x70, the bootROM zeroes bit 0 of it and then sets it back to 1 before configuring UART so it is resetting something.

#### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  | yes    | yes   |
| infinity3 | yes    | yes   |
| mercury5  | yes    | yes   |

## i2c

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

[Rough register descriptions](https://github.com/fifteenhex/linux_mstar_3.18/blob/e424e92e5442490e2f19b6d18464be3730e0678c/drivers/mstar/i2c/infinity/mhal_iic_reg.h#L208)

## spi

## pwm

## spi-nor

spi-nor functionality is made of 3 different IP blocks; ISP, FSP and QSP.
These are all basically slightly different SPI masters.

- ISP is the simplest and seems to be for doing basic SPI transactions
- FSP allows for parts of an SPI transaction come from registers, like the opcode and address, while
  other parts are read direcly from memory. This seems to be for doing fast zero copy writes.
- QSP seems to implement the memory mapped interface that allows the CPU and BDMA to read from flash
  as if it was memory.

### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  | yes    | yes   |
| infinity3 | yes    | yes   |
| mercury5  | yes    | yes   |

## SD/SDIO

There seem to be many versions or revions of the SD/SDIO block. Some versions seem to also support memory stick etc.

### *v5* as seen in i3 and m5

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  |        | yes   |
| infinity3 |        | yes   |
| mercury5  |        | yes   |

## Display pipeline

This display pipeline(s) are made up of a bunch of different blocks that can be changed/mixed together and then thrown out of an output. The vendor code for this area is a complete mess so it's going to be very hard to work out how to use any of it.

For a mipi display on the m5 the pipeline seems to look like this:

```
 -----      ----------      ------
| PNL | -> | MIPI DSI | -> | DPHY |
 -----      ----------      ------
```


### MIPI DSI

Seems to be the same as the [mediatek one](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/mediatek/mtk_dsi.c) based on [this header](https://github.com/fifteenhex/linux_mstar_3.18/blob/another_codedrop/drivers/mstar/driver/hal/infinity2/mipi_dsi/inc/reg_mipi_dsi.h).

### GOP

"Graphics Output Path". This is a simple framebuffer that uses a chunk of system memory.

### PNL

PaNeL? Seems to be incharge of driving LCDs either via a parallel interface or MIPI.

## Camera

The pipeline for the m5 seems to look like this:

```
          
 -------         -----      ------
|  VIF  | <-X-- | CSI | <- | DPHY |
 -------    |    -----      ------
            |    -------
             \- | BT656 |
                 -------
```

There seems to be 3 "vif" blocks. Then two csi blocks and two dphys. The sr0 dphy seems to only support two lanes. The sr1 dphy supports 4.

### VIF

vif, video interface?, seems to also be called "sensor" seems to be the sink for the video after it's come from mipi or somewhere else.

[Rough register descriptions](https://github.com/fifteenhex/SDK_pulbic/blob/master/Mercury5/proj/sc/driver/hal/mercury/isp/inc/mercury5_reg_vif.h)

### ISP

This is the main camera sensor interface block. It can either take input from a parallel sensor or from the CSI and in turn a MIPI CSI sensor.

### CSI

This is a frontend for the ISP that allows it to interface with a MIPI CSI sensor.

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  | n/a    | wip   |
| infinity3 | n/a    | wip   |
| mercury5  | n/a    | wip   |

## Video Encoder/Decoder

### MFE

"Multi-Format Encoder".

- Infinity1 is apparently "mfe5"
- Infinity3 is apparently "mfe6"
 
[Rough register descriptions](https://github.com/fifteenhex/SDK_pulbic/blob/master/Mercury5/proj/sc/driver/hal/mercury/mfe/inc/hal_mfe_reg.h)
 
### VFE 

- Infinity1 is apparently "h2v1"
- Infinity3 is apparently "h2v3"

### JPE

Hardware JPEG encoder

[Rough register descriptions](https://github.com/fifteenhex/SDK_pulbic/blob/master/Mercury5/proj/sc/driver/hal/mercury/jpe/pub/hal_jpe_reg.h)

## Audio

### BACH

BACH is a fairly generic DMA engine with a DAC attached audio block.

[Reverse engineering](bach.md)

#### Support Matrix

|           | u-boot | linux |
|-----------|--------|-------|
| infinity  | n/a    | wip   |
| infinity3 | n/a    | wip   |
| mercury5  | n/a    | wip   |

## Misc

### Mailbox

### AI/NN

Some chips seem to contain a [CEVA XM6](https://www.ceva-dsp.com/product/ceva-xm6/).

### SMP glue

For the infinity2, 2m and 6e where there is a second core there are registers to set
the entry address for the secondary core and then a register that presumably unlocks
the reset line of the second core and lets it go.
