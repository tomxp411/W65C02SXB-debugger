# W65C02SXB-debugger
Information for the W65C02SXB dev board's monitor and debugger

This repository is not created by or endorsed by Western Design. These are my own findings based on analyzing WDC's manuals and a disassembly of the WDCMON firmware. 

## WDCMON

WDCMon is a very simple monitor/debugger running on the WDC dev board. The whole thing is only 1328 bytes, incluing some apparently random metadata and WDC's Copyright and information notice.

The FTDI interface chip is connected to VIA 2 on the board; WDCMon talks to a PC by sending data into the VIA's FIFO, then activating "handshake mode", which causes the VIA to clock the data out, one byte at a time, through the parallel I/O pins. 

There is a nice disassembly [here](https://gist.github.com/kalj/66b23c440557839b82728850555af283). 

It looks like the monitor accepts a few commands: sync, echo, read data, write data, and execute. There is also a "register" dump, but it just returns static data. (The code is literally a set of LDA#/send operations. So this is not actually a dump of the CPU registers.) Presumably, the debugger derives the register dump by reading the stack in the NMI cycle and dumping that to a pre-defined memory location. 

To send a command, the PC must send two bytes: $55 $AA. The system responds with $CC

After that, the PC should send one of the following command bytes:

Those are:

| Byte | Command |
|------|---------|
| 00   | resync data port: returns a 0 to the PC |
| 01   | Echo response (this sends back a 01, then seems to require a 00 00 to terminate the echo command.) |
| 02   | read_from_PC / write to dev board: $02, 3 byte address, 2 byte block size, then _block size_ bytes of data. |
| 03   | write_to_PC / read from dev board: $03, 3 byte address, 2 byte block size. The board then sends back _block size_ bytes of data. |
| 04   | registers: always returns the same data.
| 05   | Execute code: TODO: at label `handle_cmd_exec:` in the disassembly (more later)
| 08   | Appears to send the state of the Carry flag: 00 or 01. However, a CLC operation in the routine always clears carrry. So I don't know what the true purpose is.
| 09   | Appears to read a byte, then send 00. The routine may have once sent 01 when the byte is not null. Purpose unclear. 


## Shadow Registers

Written to by int_save_state (part of the NMI handler) and read by handle_cmd_exec. The idea is that the CPU registers are stored in these memory locations to be inspected by the debugger, and the debugger can modify these values to pre-load the registers prior to calling a routine in memory.

#define STATE_A      $7e00
#define STATE_Ahi    $7e01
#define STATE_X      $7e02
#define STATE_Xhi    $7e03
#define STATE_Y      $7e04
#define STATE_Yhi    $7e05
#define STATE_PClo   $7e06
#define STATE_PChi   $7e07
#define STATE_SP     $7e0a
#define STATE_P      $7e0c

Other values are saved in the startup routine and then restored by handle_cmd_exec. Presumably this is to allow the system to boot to user code, then break into the monitor without destorying user data stored in Zero Page.

#define STATE_ZP     $7e14
#define STATE_ZP0    $7e14
#define STATE_ZP1    $7e15
#define STATE_ZP2    $7e16
#define STATE_ZP3    $7e17
#define STATE_ZP4    $7e18

#define STATE_PCR    $7e1c
#define STATE_DDRB   $7e1d
#define STATE_DDRA   $7e1e

#define IRQ_HANDLER_ADDRESS  $7e70
#define NMI_HANDLER_ADDRESS  $7e72

The "shadow vectors" are addresses for user code that executes after Reset, IRQ, or NMI. This allows you to set up your own NMI handler. I'm not sure if this code executes before or after the debugger has had its turn. 

The USR IRQ Handler always executes after the monitor's IRQ handler. If the ["B" flag](https://www.nesdev.org/wiki/Status_flags#The_B_flag) is set, the user IRQ handler is skipped. B is set when BRK is executed while the CPU is already in an interrupt.

#define SHADOW_VECTOR_BASE       $7efa
#define USR_NMI_HANDLER_ADDRESS  $7efa
#define USR_RST_HANDLER_ADDRESS  $7efc
#define USR_IRQ_HANDLER_ADDRESS  $7efe

The values ending in hi would be for the 65816, which has the option for 16-bit registers. 

## Plans 

I have two goals for this project:

1. Write a Windows based debugger/monitor that can be used to view and modify memory on the W65xx dev board. 
2. Update WDCMon to accept ASCII commands via the USB port. I would like to maintain compatibility with the existing monitor, hooking into the intial startup routine to switch to the text mode monitor when the user sends CR, instead of the $55 $AA sequence. 

