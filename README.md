# Repairing and Upgrading a Sarotech/Wizplat NAS-20, in 2021

The Wizplat NAS-20 is a cheap NAS box running linux. I got a free one, with the
hardware in good condition, but unresponsive: connecting the device to the network
and starting it would power up the Ethernet link, but not generate any traffic.

## Disassembly

The device was easy to disassemble, using only a couple of standard screws at the front and back.
It contains a single circuit board, fitted between the two hard disk slots. Here are some pictures
of the board:

![PCB front](images/nas40-pcb-over.jpg)

On the top side of the board, we can identify the following components,
and identify them with a web search. From left to right, we have:

  - Two DDR SDRAM chips, with a capacity of 64 MiB each (Samsung K4H51638D-UCCC)
  - A System-on-Chip (SoC) containing an ARM CPU and several integrated peripherals (Storlink SL3516/Cortina CS3516)
  - A Gigabit Ethernet tranceiver (Marvell 88E1111-RCJ1)
  - A 16MiB flash memory chip (Cypress S29GL128P)

![PCB back](images/nas40-pcb-under.jpg)

On the underside of the board, we find:
  - Two SATA cables soldered directly to the board
  - A buzzer
  - A ribbon connector going to the front panel, bearing 8 LEDs
  - A mysterious unlabeled 4-pin connector


Furthering the search, we find the [OpenGemini] project, that supports a bunch of routers
and NAS boxes based on the same SoC.

The SL3516 contains an ARMv4 CPU core, and various integrated controllers (network, usb,
uart, crypto acceleration engine, among others). It is supported by Linux under the code name
"Gemini" 

## Probing the debug port

We probe the 4 pins with a multimeter, first in continuity mode when power is off. We discover
that pin 0 (leftmost) is connected to the ground (taken from the metal screw at the bottom right
side of the board.)

We then power up the board, and measure the 3 other pins in voltage mode, first in DC mode, then in AC mode.
Pin 3 (rightmost) exhibits +5V DC, with no AC component. Pin 1 exhibits around +1V, and pin 2 around +3.5V,
with fluctuations at boot.

At this point, I would use an oscilloscope or logic analyzer to see what is happening on the pin;
unfortunately, I don't have access to one. So, we will take a wild guess, and assume that the connector
is wired to the serial UART of the SoC. I used a [Bus Pirate] to connect to it, but probably any FTDI cable
should work fine. According to google, the Gemini platforms typically operate the UART at 19200bps in standard mode.

The assumed pin layout would be, from left to right:

 - Ground
 - RX (console to device)
 - TX (device to console)
 - 5V source.

We connect the Bus Pirate to the host (linux) computer, and start Minicom on the new serial port
that just appeared:

```
minicom -D /dev/ttyUSB0 
```

Minicom start with hardware flow control enabled; We go to the setup menu (`Ctrl-A Z`), 
C`O`nfiguration, `Serial Port Setup`, and check that it is set to 115200bps, in standard
mode (8 data bits, no parity bit, 1 stop bit). If all is well, you should see the Bus Pirate
prompt `HiZ>`, meaning High-Z mode (the safe default of just behaving like a voltmeter.)

We key in `m` to change the mode of the Bus Pirate. In the menu, choose UART mode, then
pick a speed of 19200 bps, standard mode (8 data bits, no parity, 1 stop bit). Because pin 1 was
idling close to 4V after the boot, we choose Idle 1 (default) for the polarity. Then we key in
`(0)` to see the macro menu, and
`(2)` to start the Live monitor macro. We plug the Bus Pirate ground wire (brown) to pin 0 (ground),
the MISO wire (black) to pin 2 (TX). We leave the other wires unconnected; in particular,
do not connect the 5V supply wire.

We boot the device, and the following appears:

```


```

We have a working bootloader; but the kernel crashes on boot, apparently because of some
memory problem.

## Accessing the debug console

Now that we know that the UART console reception is working, we can set up the other direction.
For this, we will set the Bus Pirate in UART transparent mode. This requires the virtual serial
port to use the same settings as the UART.

First, we key in `b` to change the virtual port speed on the Bus Pirate, and select 19200.
The device answers:

```
Adjust your terminal                                                                                                                                                                           
Space to continue 
```

 Then, we go to Minicom settings (`Ctrl-A`, `O`, `Serial port setup`) and also set the speed
to 19200. The console works again. Next, we key in
`(1)` to start the Transparent UART bridge macro. Once enabled, the only way to exit
this mode is to power-cycle the Bus Pirate.

We then connect the BusPirate MOSI (grey) wire to pin 1 (RX) on the board,
and reset the NAS. At the prompt, we hit `Ctrl-C` to enter the bootloader menu.

Note: for some reason, the system initializes differently if it powers up with the UART
RX pin connected. Power up the NAS with the grey wire disconnected; then either connect
it and hit `Ctrl-C` before the 10-second bootloader delay expires, or use the reset button
afterwards, without turning the power off.

## Building a modern kernel

We will use the [Buildroot] project, which is a collection of makefiles and scripts
to automate the software build of embedded linux systems. It can build a cross-compilation
toolchain for the host, and a kernel and root filesystem for the target.

We start by downloading the latest buildroot from the official Git repository,
and enter the configuration interface:

```
git clone git://git.buildroot.net/buildroot
cd buildroot
make menuconfig
```

The buildroot menuconfig interface, like the linux kernel one, saves its settings in a
file `.config`, in the top-level buildroot directory.
Here is the [full config](./buildroot-config) file, after making the correct choices.

Under Target Options, we choose the ARM (little endian) architecture. As we saw from the
failing boot log, the CPU inside the Storlink SoC is a Faraday 526 ARMv4 core, so we pick this
as the target variant.

The plain Linux kernel does not come with a default configuration for our obscure board.
We will specify a full custom kernel configuration file.

## Creating a Device Tree

A CPU architecture specification determines the behavior of the CPU, but not the configuration
of the entire system (and in particular, the I/O ports and/or memory addresses for the peripherals,
IRQ lines, etc). On PC platforms, such information is typically decided by the designer of
the mainboard, and then hardcoded into the firmware (BIOS or UEFI) of the board. This information
is then passed to the kernel at boot time, via ACPI tables or DMI structures.

For embedded linux, no equivalent standard existed for a long time. The current standard
is [Devicetree]

## Credits

This project would never have happened without the outstanding open work from the [Buildroot]
project, from [OpenGemini], and from [Linus Walleij].

[Bus Pirate]: https://dangerousprototypes.com/docs/Bus_Pirate
[Buildroot]: https://buildroot.org
[Devicetree]: https://devicetree.org
[OpenGemini]: http://opengemini.free.fr
[Linus Walleij]: https://dflund.se/~triad/krad
