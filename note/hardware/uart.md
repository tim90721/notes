- [Introduction](#introduction)
- [Configuration](#configuration)
  - [Data frame](#data-frame)
    - [Start bit (not configurable)](#start-bit-not-configurable)
    - [Data size (character size)](#data-size-character-size)
    - [Parity (Data corruption check)](#parity-data-corruption-check)
    - [Stop bit](#stop-bit)
  - [Baudrate (Speed)](#baudrate-speed)
  - [Flow control (CTS/RTS)](#flow-control-ctsrts)
  - [Common configuration](#common-configuration)
- [Hardware Implementation](#hardware-implementation)
  - [ARM PL011](#arm-pl011)
    - [Registers](#registers)
      - [UARTDR](#uartdr)
      - [UARTFR](#uartfr)
      - [UARTIBRD \& UARTFBRD](#uartibrd--uartfbrd)
      - [UARTLCR\_H](#uartlcr_h)
      - [UARTCR](#uartcr)
      - [UARTIMSC](#uartimsc)
      - [UARTDMACR](#uartdmacr)
- [Reference](#reference)

# Introduction
- UART (Universal Asynchronous Receiver/Transmitter) is commonly referring to both 
  - A serial communication protocol
    - Actually, the protocol includes RS232, RS449, RS423, RS422, RS485 specification
  - A hardware
- UART is serial
  - Data is transmitted **bit by bit**
- UART supports full duplex
  - Data can be transferred in **both directions** (Tx/Rx) **at the same time**
- To have a successful UART communication, both the transmitter and receiver must use the same
  - Data frame format
    - 1 data frame carries 1 character
  - Bit transfer speed
- Optional function to check data corruption

# Configuration
## Data frame
### Start bit (not configurable)
Indicate the start of the data frame. The hardware will start to demodulate the data after the start bit.

### Data size (character size)
This configuration is usually depending on the hardware. Common configurable data size are
- 5 bits
- 6 bits
- 7 bits
- 8 bits

### Parity (Data corruption check)
Append an additional bit after the data for data corruption check. Usually the available configurations are
- Check even
  - If the data contains odd number of '1', then this bit is set to '1'. 
    Otherwise, set to '0'
- Check odd
  - If the data contains even number of '1', then this bit is set to '1'.
    Otherwise, set to '0' 
- Sticky parity
  - Force set the parity bit to '0' or '1'.

### Stop bit
Stop bit length can be configurated. Common configurations are
- 1 stop bit
- 1.5 stop bits
- 2 stop bits

## Baudrate (Speed)
How many bits can be transferred per second.
Depending on different hardware, the configuration of baudrate may not be the same way (?)

## Flow control (CTS/RTS)

## Common configuration
The most common used configuration is
- 8 data bits (corresponding to C char size)
- no parity bit
- 1 stop bit
- baudrate can be varied, depending on the use case. Usually 115200 is enough for debugging

# Hardware Implementation
Most of the UART implementation are based on UART8250 or NS16550

## ARM PL011
[ARM PL011 Reference manual](https://developer.arm.com/documentation/ddi0183/g/)
- PL011 appears to be a decendent of NS16550 hardware
- PL011 has some Tx/Rx buffers (FIFO) so that we can queue multiple data to be transmitted at the same time
- Capable of generating some intterupts when the hardware 
  - Received some data
  - Transmitted the data completely

### Registers
#### UARTDR
UART data register. The data to be transmitted or the received data is put in here by the SW or the HW.

#### UARTFR
UART flag register. This register contains several bits to indicate the status of the hardware. 
For instance, 
- TXFE
  is the Tx FIFO empty
- TXFF
  is the Tx FIFO full
- RXFE
  is the Rx FIFO empty
- RXFF
  is the Rx FIFO full
- BUSY
  UART is busy transmitting a data frame

#### UARTIBRD & UARTFBRD
UART baudrate configuration register.
In PL011, The baudrate is calculated based on the following equation
$$ BAUD\_DIV = \frac {F_{uart\_clock}}{16 \times Baudrate} $$
Basically, The hardware will use the UART reference clock $F_{uart\_clock}$ divided with $16$ and $BAUD\_DIV$ to get the desired baud rate. $F_{uart\_clock}$ is the UART's clock and $Baudrate$ is the desired baudrate.

The 2 registers UARTIBRD and UARTFBRD
- UARTIBRD
  - Stores the integer part of the baudrate divisor
  - A 16-bit wide register
- UARTFBRD
  - Stores the fractional part of the baudrate divisor
  - A 6-bit wide register

When configuring baudrate in the program, some tricks can help us to easily configure these 2 registers.

First, by times 64 in both side of the equation
$$ BAUD\_DIV \times 64 = \frac {F_{uart\_clock}}{16 \times Baudrate} \times 64 $$

which can be simplified to
$$ BAUD\_DIV \times 64 = \frac {F_{uart\_clock}}{Baudrate} \times 4 $$

By doing so, the fractional part will be the last 6 bits of the result of 
$$ DIV_{fractional} = \frac {F_{uart\_clock}}{Baudrate} \times 4 $$

And the integer part, we can just simply divide 64 to the above result
$$ DIV_{integer} = (\frac {F_{uart\_clock}}{Baudrate} \times 4) / 64 $$

The divide 64 operation is just a logical shift operation in C programming

Following is a sample code to calculate the value of UARTIBRD and UARTFBRD
```
void baudrate_configure(uint32_t baudrate)
{
        uint32_t div = 4 * uart_clock / baudrate;
        uint32_t d_frac = div & 0x3F;
        uint32_t d_int = (div >> 6) & 0xFFFF;
}
```

> [!WARNING] UARTIBRD, UARTFBRD and UARTLCR_H form a 30-bit wide register and the content is only updated when UARTLCR_H is updated. As a result, Always write UARTLCR_H at the end.

#### UARTLCR_H
UART line control register. Configuration for data frame format such as parity, data length, stop bit. Note that, PL011 seems to only support 1 or 2 stop bits.

Some miscellaneous configurations are to enable FIFO.

#### UARTCR
UART control register. Enable or disable UART, enable Tx or Rx, configure for CTS/RTS, look back mode etc.

> [!NOTE] The control flow to program UARTCR is described below
> 1. Disable UART
> 2. Wait for the end of transmission or reception of the current data frame
> 3. Flush the FIFO by setting the FEN bit to 0 in UARTLCR_H
> 4. Configure UARTCR
> 5. Enable UART

#### UARTIMSC
UART interrupt mask set/clear register.

#### UARTDMACR
UART DMA control register.

# Reference
https://krinkinmu.github.io/2020/11/29/PL011.html