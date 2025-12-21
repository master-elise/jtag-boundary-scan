# JTAG boundary scan probing of the Red Pitaya Zynq 7010

Supporting material for J.-M Friedt, "D&eacute;bugger un syst&egrave;me embarqu&eacute; avec GDB et JTAG : mauvais &eacute;l&egrave;ves,
boundary scan et syst&egrave;mes asym&eacute;triques", Hackable 64 (Jan-Feb 2026) [in French].

## Getting familiar with JTAG boundary scan

The <a href="XC7Z010_CLG400.bsdl">BSDL</a> (Boundary Scan Description Language) file describing the cells in the Zynq 7010 was downloaded
from https://bsdl.info/.

The OpenOCD scripts <a href="redpitaya1.openocd">redpitaya1.openocd</a> demonstrates how to generate boundary scan commands for 
probing the JTAG chain and identifying the ARM processors and the FPGA.

The script is executed with the ``-f`` argument of OpenOCD, assuming a Digilent HS2 JTAG probe is used:
```
sudo openocd -f redpitaya1.openocd
```
leading to
```
...
Info : JTAG tap: zynq_pl.bs tap/device found: 0x13722093 (mfg: 0x049 (Xilinx), part: 0x3722, ver: 0x1)
Info : JTAG tap: zynq.cpu tap/device found: 0x4ba00477 (mfg: 0x23b (ARM Ltd), part: 0xba00, ver: 0x4)
Info : zynq.cpu0: hardware has 6 breakpoints, 4 watchpoints
Info : [zynq.cpu0] Examination succeed
Info : zynq.cpu1: hardware has 6 breakpoints, 4 watchpoints
Info : [zynq.cpu1] Examination succeed
Info : [zynq.cpu0] starting gdb server on 3333
Info : Listening on port 4444 for telnet connections
```
with port 4444 opened to send Boundary Scan commands to the chip through OpenOCD. The instruction set
is documented in the BSDL file, with for example
```
attribute INSTRUCTION_OPCODE of XC7Z010_CLG400 : entity is
  "IDCODE   (001001)," & -- DEVICE_ID
  "BYPASS   (111111)," & -- BYPASS
  "EXTEST   (100110)," & -- BOUNDARY
  "SAMPLE   (000001)," & -- BOUNDARY
  "PRELOAD  (000001)," & -- Same as SAMPLE
  "USERCODE (001000)," & -- DEVICE_ID
  "HIGHZ    (001010)," & -- BYPASS
```
so that command 9 will provide the device ID and testing the pins through the <a href="https://interrupt.memfault.com/blog/diving-into-jtag-part-3#extest-instructions">EXTEST instruction</a> 
requires opcode 0x26=38.
```
$ telnet localhost 4444
> scan_chain
   TapName             Enabled  IdCode     Expected   IrLen IrCap IrMask
-- ------------------- -------- ---------- ---------- ----- ----- ------
 0 zynq_pl.bs             Y     0x13722093 0x*3723093     6 0x01  0x03
> poll off
> irscan zynq_pl.bs 9   
> drscan zynq_pl.bs 32 0
13722093
```
We must also know the number of accessible bit configuration registers, as indicated with
``attribute BOUNDARY_LENGTH of XC7Z010_CLG400 : entity is 770;``

## Blinking LED

The engineering schematic of the Red Pitaya board provides the link between the pin of the Zynq7010 and the LED visible on the board

<img src="redpit2.png"><img src="redpit3.png">

demonstrating how the yellow LED6 is connected to pin J15 of the Zynq PL. This pin description is found in the BSDL file as
```
  " 176 (BC_2, *, controlr, 1)," &
  " 177 (BC_2, IO_J15, output3, X, 176, 1, Z)," & --  PAD50
  " 178 (BC_2, IO_J15, input, X)," & --  PAD50
```
so that bit 176 indicates if the Boundary Register Cell is active (set to 0 to activate, with controlr indicating that
the register state can also be read) and bit 177 switches the LED state.

The OpenOCD script <a href="redpitaya2.openocd">redpitaya2.openocd</a> demonstrates how to generate boundary scan commands for 
manipulating the GPIO pins connected to the LEDs by first reading the peripheral state and only modifying the required bits. Notice
that OpenOCD can only handle register descriptions up to 32 bit long, but accepts multiple sequential arguments so that the
770 bits are addressed as 24 integers 32 bit long and one 2-bit request:
```
> irscan zynq_pl.bs 0x26
> drscan zynq_pl.bs 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 32 0xFFFFFFFF 2 0x3 -endstate drpause
```
returning the default state
```
0x7edffffb 0xfb6db6ff 0x5edb7de0 0x6dbffb6c 0xdfedffd9 0xfffeffff 0xffffffff 0xffffffff 0xffffffff 0x6db7fb6f 0xfffffffb 0xffffffff 0xffffffff 0xffffffff 0xfffdffff 0xffffffff 0xffffffff 0xffffffff 0xbedbfdff 0x7fffffed 0xdb6fffff 0xfffffff6 0xbfffffc3 0xffffffd8 2 0x03
```

Demonstration of boundary scan commands sent through the JTAG interface for switching on and off the LEDs:

<img src="LED.jpg">
