# Self Programming Module

The P80C550-SPM is an expansion board for the P80C550-EVN which provides self programming capability by providing an alternate memory device which can be swapped to at runtime. A bootloader will be installed on the onboard ROM of the P80C550-EVN which will offer the ability to program this alternate memory device. It also provides an SPI controller hard configured for MSB first transmission in mode 0 to interface with an SD/MMC card.

![Breadboard](/Assets/breadboard-1.webp)

## Memory Map
| Data Address   | Description            |
| -------------- | ---------------------- |
| 8000h          | SPM Control Register   |
| 8400h          | SPI Data Register      |
| 8401h          | SPI Control Register   |
| A000h -> BFFFh | SPM Memory Window      |

## Self Programming

The self programming control register is overlain on the Power Status Register on the base board, as only bit 7 is defined (the other bits are high-Z). This allows the use of these 7 bits in expansion boards. A total of 19 address bits (512 KiB) are provided for the SPM memory device. When accessed as code memory, A18 -> A16 are set to 0 and A15 -> A13 are forwarded from the address bus. The CPLD has no electrical connection to A12 -> A0 of the memory device. When accessed as xdata memory via an 8 KiB window at A000h, A19 -> A13 are set to the "SPM Memory Page" bits from the control register. 

During development an SST39SF040 was used as the reference memory device. Along with the SST39SF010 (128 KiB) and SST39SF020 (256 KiB), it is one of the few parallel flash memories still in production at a reasonable price ($2 USD). It also has a page size of 4 KiB is awesome, as most comparable flash memories of the era (e.g. AM29F040) had 64 KiB pages. Another option is to use a RAM for the SPM memory device, the tradeoff being that the ROM has to load an image to execute from somewhere (such as an SD/MMC card).

### Self Programming Control Register

| Bit | Description           |
| --- | --------------------- |
|  7  | R: ~PFO / W: PFO_MASK |
|  6  | SPM Memory Enable     |
| 5:0 | SPM Memory Page       |

The SPM memory enable bit controls whether the CPU is executing from the SPM memory device or the onboard ROM. At reset, the onboard ROM is selected and the value of this bit is 0. After ensuring a valid ROM image is loaded into the first 64 KiB of the memory device, writing a 1 to this register will switch the entire code address space to point at the SPM memory. Any writes to this bit are delayed by 3 rising edges of the ~CODE_RD signal (3-bit shift register), giving exactly enough space for a long jump before the switch occurs. The same applies to clearing the bit to allow jumping back into the ROM. This could be an important feature if the program loaded on the SPM memory wants to write to the SPM memory itself, assuming writes and execution can't occur simultaneously (such as if the SPM memory is flash).

```asm
    ; 8051 assembly code for booting
boot:
    ; disable interrupts
    CLR  EA
    ; set SPM memory enable
    MOV  DPTR, #0x8000
    MOVX A, @DPTR
    ORL  A, #0x40
    ; write happens here
    MOVX @DPTR, A
    ; exactly 3 more bytes can be read from code memory
    ; before switch. This is the length of an 8051 LJMP
    ; instruction. I highly recommend jumping to the
    ; reset vector of the new image.
    LJMP 0
    ; can not reach
```

Bits 5 -> 0 ("SPM Memory Page") controls address bits A18 -> A13 of the memory device when accessed via the SPM memory window (A000h -> BFFFh) in the data address space.

### SPI Master

| Bit | Description          |
| --- | -------------------- |
|  7  | SPI Interrupt Flag   |
|  6  | SPI Interrupt Enable |
|  5  | SPI Running / Busy   |
|  4  | reserved             |
| 3:2 | ~SS Enable           |
| 1:0 | SCK Prescaler Select |

- The SPI Interrupt Flag is set after a SPI transfer has completed. It must be cleared before another transfer can take place by writing a 1 to the bit.
- The SPI Interrupt Enable bit controls whether the EX0 interrupt on the CPU is triggered when a transfer completes.
- The SPI Running / Busy Flag reads as 1 when a SPI transfer is currently in progress.
- The ~SS Select bits controls which ~SS line is currently active.
  - 0 = ~SS0, ~SS1, and ~SS2 are deasserted (high)
  - 1 = ~SS0 asserted
  - 2 = ~SS1 asserted
  - 3 = ~SS2 asserted
- The SCK prescaler select bits control the clock divider that generates SCK
  - 0 = f_sck = f_clk / 2
  - 1 = f_sck = f_clk / 8
  - 2 = f_sck = f_clk / 32
  - 3 = f_sck = f_clk / 128

A new SPI transfer is initiated by writing a byte to the SPI data register.

```asm
    ; setup SPI with no interrupts and f_sck = f_clk / 32
spi_init:
    MOV  DPTR, #0x8401
    MOV  A, #0x82
    MOVX @DPTR, A
    ret

    ; perform an SPI transfer
    ; A  - data to send
    ; return: A - data received
    ; clobbers: A, DPTR
spi_transfer:
    ; start transfer
    MOV  DPTR, #0x8400
    MOVX @DPTR, A
    ; wait for transfer complete flag to be set
    INC  DPTR
spi_transfer_wait:
    MOVX A, @DPTR
    RLC
    JNC  spi_transfer_wait
    RRC
    ; clear flag (write contents of control register to control register)
    MOVX @DPTR, A
    ; read data register and return
    DEC  DPL
    MOVX A, @DPTR
    ret
```

### Bootloader

At present, the minimal bootloader is a very simple program that waits for 1 second for any character to be received by the 8051's internal uart. It currently assumes that the memory device is an SST39SF010/SST39SF020/SST39SF040 parallel flash.

- 'P' is received: erase 64 KiB code page of flash and accepts up to 64 KiB of data via xmodem
- 'U' is received: upload 64 KiB code page of flash via xmodem
- 'D' is received: upload all 512 KiB of flash via xmodem
- 'B' is received: boot to flash device
- any other byte is received: print 'N'

### SD Card Benchmark

Using the SDCC 8051 toolchain and a highly unoptimized access routine, I was able to achieve a raw read speed of 24.5 KiB/s

![Benchmark](/Assets/sdcard-1.png)
