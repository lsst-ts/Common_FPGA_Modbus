# Modbus communication library

This library provides functions to send and receive messages over serial line
to/from other devices. Runs only on FPGA. Used mainly to communicate on a
master-slave bus, usually with daisy chained addressing of slave modules.

**[CRC16](https://crccalc.com) calculation is not included**. Calling process should calculate and
include proper CRC16 as data payload, and check received data CRC16 (or any
other CRC).

## Physical interface

Serial interface is constructed from two DIO pins. Obviously one pin has to be
configured for input (receiving), the other for output (transmitting). Usually
NI-9401 module, providing total 8 DIOs, with split for 4 in and 4 output ports,
is used.

## LabView/Host interface

[Common_FPGA_HealthAndStatus
library](https://github.com/lsst-ts/Common_FPGA_HealthAndStatus/) is used for
health and status FIFO. That provides datatype for the FIFO.

For each use, one input and one output DIO are needed. Six FIFOs has to be
provided - transmitting, receiving, timestamps, healt & status for Rx and Tx,
and internal transmitting queue. Three boolean registers are needed - IRQ,
WaitForIRQ, and ModbusTrigger (starting physical write after receving 0x8000
"Wait for trigger command").

FIFOs needs to be "never arbitrate" on side used inside module - write for
outputs/receiving, read for inputs/transmitting. The other side should be set
to never arbitrate as well, but that's isn't required.

For each use:

1. Create a U16 FIFO for received data.

2. Create a U16 FIFO for transmitted data.

3. Create a custom type ModbusTxInstruction FIFO for internal usage while
   transmitting data. Specify never arbitrate for read and write.

4. Create two custom type HealthAndStatusUpdate FIFOs - one for receiving,
   second for transmission.

5. Create a boolean register for IRQ control and "WaitForRXFrame". Specify
   never arbitrate for write.

6. Generate a boolean register for the software controlled trigger. Specify
   never arbitrate for write. You can use a single trigger for controlling
   multiple ports so this is an optional step if a register has already been
   generated.

7. Create a FIFO for Tx timestamps. Specify a data type of U64. Specify never
   arbitrate for write.

8. Drop the FPGAModbus/ModbusPort.vi onto your main VI. Setup the port settings
   using the resources defined above. Sets base address (shall be multiply of
   6) and IRQ (ordinals).  Review the comments in ModbusPort.vi for more
   information.

### Optional

To copy health and status data FIFOs into a single FIFO:

9. Drop the FPGAHealthAndStatus/HealthAndStatusFIFOCopy.vi. Set the internal
   health and status FIFO. Set the source as the FIFO generated in step 2.

10. Drop the FPGAHealthAndStatus/HealthAndStatusFIFOCopy.vi. Set the internal
    health and status FIFO. Set the source as the FIFO generated in step 5.

# Theory of operation

FIFOs are used for what they are best suited - to provide a channel between
variable speeds write and read ends. Usually one end of the FIFO is inside
[SCTL](https://knowledge.ni.com/KnowledgeArticleDetails?id=kA00Z000000P8sWSAS&l=en-US).
SCTL are used to grant execution of the commands inside FPGA in a single tick.

Link speed is hard-coded to 44 40Mhz ticks per bit, which roughly equal to
921600 bauds *(actual number is 40M / 44 = 909090.909; but as delays are
inserted between frames, and frames aren't long, 44 works, sampling a slightly
different part of receiving signal for each received bit)*.

One start and one stop bits are written (must be included in data, see below).

Transmitting FIFO is U16 typed. The most significant nibble is used for
instruction. Nibbles 2-4 are used for data.

Data from transmitting FIFO are parsed into instructions and data, and placed
into internal FIFO. Instructions are then processed from the internal FIFO -
see below for instructions types. Proper signal (frame start, start bit, data,
stop bit, frame delay) are generated.

Received line is checked for frame start. Bytes are then stored into receiving
FIFO. Frame end is detected on line silence. IRQ is raised when frame is
received. As the expected use is in a master-slave communication, IRQ doesn't
have to be handled outside ModbusPort subVI. Data are writen into receiving
FIFO.

CRC16 isn't checked on received FIFO and the CRC16 (if part of the
communication) is stored into receiving FIFO.

# Instructions

Opcodes 1-8 are used for transmission (Tx FIFO). Instructions 9 and 10 are used
in received response (Rx FIFO).

 OpCode | Description
 ------ | -----------
 0x1    | sends data. Data are in bits 9-0, and includes start and stop bits. The least significant bit (bit 0) is a start bit, must be 0. 8 data bits follow. Bit 9 is stop  bit, must be 1. To write 8 bit data, shift left (with 0 fill) by 1, or result with 0x0200
 0x2    | End of frame, delay in ticks (data). Should be 0xda, resulting in 0x20da written
 0x3    | delay for data ticks
 0x4    | delay for data microseconds
 0x5    | delay for data milliseconds
 0x6    | wait for a received frame. Data = timeout in us (microseconds, 1/1M of a second, 40 ticks). Recommended value is 0x3e8 = 1ms.
 0x7    | triggers IRQ
 0x8    | wait for trigger (ModbusTrigger == true)
 0x9    | received data (on received FIFO)
 0xA    | received end of frame (on received FIFO)

For synchronization with the other modbus ports, a frame should be prefixed
with 0x8 command. Data wouldn't be transmitted before ModbusTrigger == true.
ModbusTrigger shall be turned to false by process/subVi calling 0x8 command.

## Example

As an example, to call Modbus function 17 = 0x11 for address 0x81, so the
payload is:

```
0x81 0x11
```

| Serb. | Func. | CRC   |
| Addr  | Code  |       |
| ----- | ----- | ----- |
| 81    | 11    | A1 EC |

*[CRC16/MODBUS](https://crccalc.com) of 0x81 0x11 = 0xeca1, transmitted as low
endian (hence 0xa1 0xec, right shifted by 1 = 0x142 0x1d8; with stop bit =
0x342 0x3d8).*

the following has to be commanded:

* set WaitForTrigger
* write payload and CRC
* wait 1ms for reply
* signal TriggerIRQ

For that, the following 2 bytes numbers (in hex) shall be written into Tx FIFO:

```
0x8000 0x1302 0x1222 0x1342 0x13d8 0x20da 0x63e8 0x7000
```

Assuming the device returns the following data:

| Serb. | Func. | Length | Unique ID         | ILC     | Net. | ILC      | Net.Node | Maj. | Min. | Firmware    | CRC   |
| Addr  | Code  |        |                   | AppType | Node | Sel.Opt. | OPtions  | Rev. | Rev. | Name        |       |
| ----- | ----- | ------ | ----------------- | ------  | ---- | -------- | -------- | ---- | ---- | ----------- | ----- |
| 81    | 11    |   10   | 12 34 56 78 90 AA |   FF    |   BB |   CC     |   DD     |   EE |   11 | 53 74 61 72 | A7 9F |

this shall be read from receiving FIFO:

```
0x9302 0x9222 0x9220 0x9224 0x9268 0x92ac 0x92f0 0x9320 0x9354 0x93fe 0x9398 0x93ba 0x93dc 0x9222 0x92a6 0x92e8 0x92c2 0x92e4 0x934e 0x933e 0xA000
```

**It's reader's responsibility to check CRC16.**

# Health and Status telemetry

Health and status telemetry is provided as instructions written into Health and
Status FIFO. Please see [Common_FPGA_HealthAndStatus
library](https://github.com/lsst-ts/Common_FPGA_HealthAndStatus/) for details.

The following offsets from the base address specified in ModbusPort settings
are populated:

| **0** | flags. See below for details                                            |
| **1** | transmitted (Tx) byte count                                             |
| **2** | transmitted (Tx) frame count                                            |
| **3** | received (Rx) byte count                                                |
| **4** | received (Rx) frame count                                               |
| **5** | transmitted (Tx) instructions count; shall be higher than Tx byte count |

## Error Flags

| Bit | Value |  Hex |  Name                         |
| --- | ----- | ---- | ----------------------------- |
|  0  |  1    | 0001 | TxInternalFIFO Overflow       |
|  1  |  2    | 0002 | Invalid Instruction           |
|  2  |  4    | 0004 | Wait For Rx Frame Timeout     |
|  3  |  8    | 0008 | Start Bit Error               |
|  4  |  16   | 0010 | Stop Bit Error                |
|  5  |  32   | 0020 | RxDataFIFO Overflow           |
|  6  |  64   | 0040 | Waiting for software trigger  |
