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
 - RX (terminal to device)
 - TX (device to terminal)
 - +5V source.

![Debug port](images/debug-port.jpg)

For the first attempt, we do not connect the RX/MOSI pin. In any case, *DO NOT CONNECT 
THE +5V PIN*.  

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
Storlink CIR Initialization
Please reboot now.

No raid assembled




Sarotech NAS, version 1.2.5_P16M
Built by linux, 08:49:04, Sep 16 2009

Processor: SL3516c3
CPU Rate: 300000000
AHB Bus Clock:150MHz    Ratio:2/1
MAC 1 Address: 00:0F:D6:xx:xx:xx
MAC 2 Address: 00:50:C2:xx:xx:xx
inet addr: 192.168.123.33/255.255.255.0
==> enter ^C to abort booting within 10 seconds ......
Load Kern image from 0x30020000 to 0x1600000 size 3145728
Load Ramdisk image from 0x30620000 to 0x800000 size 6291456
Uncompressing Linux..................................................................
Linux version 2.6.15 (root@hwjang-linux) (gcc version 3.4.4) #158 Wed Nov 18 16:51:24 CST 2009
CPU: FA526id(wb) [66015261] revision 1 (ARMv4)
Machine: GeminiA
Ignoring unrecognised tag 0x00000000
Memory policy: ECC disabled, Data cache writeback
CPU0: D VIVT write-back cache
CPU0: I cache: 16384 bytes, associativity 2, 16 byte lines, 512 sets
CPU0: D cache: 8192 bytes, associativity 2, 16 byte lines, 256 sets
Built 1 zonelists
Kernel command line: root=/dev/ram0 rw console=ttySL0,19200 initrd=0x00800000,16M ramdisk_size4
PID hash table entries: 1024 (order: 10, 16384 bytes)
Bus: 150MHz(2/1)
sl2312 console setup :
Dentry cache hash table entries: 32768 (order: 5, 131072 bytes)
Inode-cache hash table entries: 16384 (order: 4, 65536 bytes)
Memory: 128MB = 128MB total
Memory: 108544KB available (3296K code, 1270K data, 104K init)
Mount-cache hash table entries: 512
*** Page_chain_cachep Init!***
CPU: Testing write buffer coherency: ok
checking if image is initramfs...it isn't (no cpio magic); looks like an initrd
Freeing initrd memory: 16384K
NET: Registered protocol family 16
PCI: bus0: Fast back to back transfers disabled
sl2312_pci_map_irq : slot = 0  pin = 1
SCSI subsystem initialized
usbcore: registered new driver usbfs
usbcore: registered new driver hub
Initial RAID Descripter...
XOR:tx_desc = ffc00000
XOR:rx_desc = ffc01000
XOR:tx_desc_dma = 01715000
XOR:rx_desc_dma = 01716000
NetWinder Floating Point Emulator V0.97 (double precision)
Unable to handle kernel paging request at virtual address e3a02001
pgd = c0004000
[e3a02001] *pgd=00000000
Internal error: Oops: 1 [#1]
Modules linked in:
CPU: 0
PC is at kobject_hotplug+0x78/0x2c0
LR is at kobject_unregister+0x18/0x2c
pc : [<c01d9a84>]    lr : [<c01d92c8>]    Not tainted
sp : c0601f08  ip : c0601f54  fp : c0601f50
r10: 00000000  r9 : e3a02001  r8 : c0601f8c
r7 : c0359fd8  r6 : c7271ee0  r5 : c0161b48  r4 : 00000002
r3 : e3a02001  r2 : 00000001  r1 : 00000002  r0 : c0161b48
Flags: NzCv  IRQs on  FIQs on  Mode SVC_32  Segment kernel
Control: 397F  Table: 00004000  DAC: 00000017
Process swapper (pid: 1, stack limit = 0xc0600194)
Stack: (0xc0601f08 to 0xc0602000)
1f00:                   c0601f2c c7273c00 c032f04c 00000003 00000000 c0361b5c
1f20: 00000000 c7271ee0 c7271ee0 00000001 00000009 c0359fd8 c0601f8c 00000001
1f40: c0359fec c0601f64 c0601f54 c01d92c8 c01d9a1c c7271ee0 c0601f84 c0601f68
1f60: c0012b38 c01d92c0 00000005 0000002e 00000014 c0359fd8 c0601fcc c0601f88
1f80: c0012c74 c0012ad4 c0359fd8 70756372 65746164 00000000 00000000 00000000
1fa0: 00000000 c001e7d4 c0600000 00000000 c001ee94 c001e98c 00000000 00000000
1fc0: c0601ff4 c0601fd0 c0022114 c0012b58 00000001 00000000 00000000 00000000
1fe0: 00000000 00000000 00000000 c0601ff8 c003b470 c0022074 840084fb 05807aee
Backtrace:
[<c01d9a0c>] (kobject_hotplug+0x0/0x2c0) from [<c01d92c8>] (kobject_unregister+0x18/0x2c)
[<c01d92b0>] (kobject_unregister+0x0/0x2c) from [<c0012b38>] (kernel_param_sysfs_setup+0x74/0x)
 r4 = C7271EE0
[<c0012ac4>] (kernel_param_sysfs_setup+0x0/0x84) from [<c0012c74>] (param_sysfs_init+0x12c/0x1)
 r7 = C0359FD8  r6 = 00000014  r5 = 0000002E  r4 = 00000005
[<c0012b48>] (param_sysfs_init+0x0/0x178) from [<c0022114>] (init+0xb0/0x258)
[<c0022064>] (init+0x0/0x258) from [<c003b470>] (do_exit+0x0/0xb3c)
 r8 = 00000000  r7 = 00000000  r6 = 00000000  r5 = 00000000
 r4 = 00000000
Code: 0afffffb e5953044 e3530000 11a09003 (e5993000)
 <0>Kernel panic - not syncing: Attempted to kill init!
```

We have a working bootloader; but the kernel crashes early on boot, apparently because of some
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


![Bus Pirate connected to the board](images/bus-pirate.jpg)


```
==> enter ^C to abort booting within 10 seconds ......
PHY 0 Addr 1 Vendor ID: 0x01410cc2
mii_write: phy_addr=0x1 reg_addr=0x4 value=0x5e1
mii_write: phy_addr=0x1 reg_addr=0x9 value=0x300
mii_write: phy_addr=0x1 reg_addr=0x0 value=0x1200
mii_write: phy_addr=0x1 reg_addr=0x0 value=0x9200
mii_write: phy_addr=0x1 reg_addr=0x0 value=0x1200
RTC_start ...
RTC Enable ...
RTC_start: Recoder=1

                              Boot Menu
==============================================================================
T: PCB Test!!!                              Z: BootLoader Update!!!
X: Upgrade Firmware                         L: All LED & Fan (Turn ON) Test!!!
M: All LED & Fan (Turn OFF) Test!!!         S: Shut down PCB!!!
Y: Upgrade MAC                              0: Reboot
1: Start the Kernel Code                    2: List Image
3: Delete Image                             4: Create New Image
5: Enter Command Line Interface             6: Set IP Address
7: Set MAC Address                          8: Show Configuration
9: Upgrade MAC                              F: Create Default FIS
I: Initialize IDE                           X: Upgrade Boot
Y: Upgrade Kernel                           Z: Upgrade Firmware
A: Upgrade Application                      R: Upgrade RAM Disk
N: Upgrade Bootloader(SDK)

=> Select:
```

Whoever hacked this menu together apparently added some options at the top, without
checking for key conflicts. Because of this, the `Upgrade Kernel` and `Upgrade Boot`
commands are inaccessible.

Commands 2,3,4 and F are for manipulating the partition table of the flash memory:

```
=> Select: 2                                                                                                                                                                                   
                                                                                                                                                                                               
Name              FLASH addr           Mem addr    Datalen     Entry point                                                                                                                     
BOOT              0x30000000-3001FFFF  0x00000000  0x00020000  0x00000000                                                                                                                      
FIS directory     0x30FE0000-30FFFFFF  0x30FE0000  0x00001400  0x00000000                                                                                                                      
Kern              0x30020000-3031FFFF  0x01600000  0x00300000  0x01600000                                                                                                                      
Ramdisk           0x30320000-3091FFFF  0x00800000  0x00600000  0x00800000                                                                                                                      
VCTL              0x30F20000-30F3FFFF  0x00000000  0x00020000  0x00000000
```

`FIS directory` is the same name used by the [RedBoot] bootloader to store the
partition table. Could this be a disguised version of RedBoot ?

I'm not sure why the interface prints the partitions in this order. Our 16M flash
chip contains 128 sectors of 128kiB, organized as follows:

- 1 sector (128kiB) for the bootloader, that probably starts executing at 0
- 24 sectors (3MiB) for the kernel image
- 48 sectors (6MiB) for the linux initrd
- 48 unallocated sectors
- 1 sector for a "VCTL" partition (no idea what that does)
- 5 unallocated sectors
- 1 sector for the partition table

I guess the `Mem addr` and `Entry point` columns are hint for the loader. It seems
the bootloader is scripted to load the `Kern` and `Ramdisk` images at fixed
memory locations, then jumps to the kernel entry point. The kernel contains
the address of the ramdisk hardcoded in its command-line arguments.


## Building a modern kernel

We will use the [Buildroot] project, which is a collection of makefiles and scripts
to automate the software build of embedded linux systems. It can build a cross-compilation
toolchain for the host, and a kernel and root filesystem for the target.

We start by downloading the latest buildroot from the official Git repository,
and enter the configuration interface:

```
$ git clone git://git.buildroot.net/buildroot
$ cd buildroot
$ make menuconfig
```

The buildroot menuconfig interface, like the linux kernel one, saves its settings in a
file `.config`, in the top-level buildroot directory.
Here is the [full config](./buildroot-config) file, after making the correct choices.

### Configuring Buildroot

Under `Target Options`, we choose the `ARM (little endian)` architecture. As we saw from the
failing boot log, the CPU inside the Storlink SoC is a Faraday 526 ARMv4 core, so we pick this
as the target variant (`fa526/626`).

The plain Linux kernel does not come with a default configuration for our obscure board.
Under `Kernel`, `Kernel configuration`, we will pick `Use a custom config file`.

Under `Kernel binary format`, we choose `zImage with appended DT`, for reasons that will be
explained in the later paragraph.

This will automatically enable the `Build a Device Tree Blob` option; we will use the
`Out-of-tree Device Tree Source` option to provide our custom device tree.

### Configuring the Kernel

TODO

## Creating a Device Tree

A CPU architecture specification determines the behavior of the CPU, but not the configuration
of the entire system (and in particular, the I/O ports and/or memory addresses for the peripherals,
IRQ lines, etc). On PC platforms, such information is typically decided by the designer of
the mainboard, and then hardcoded into the firmware (BIOS or UEFI) of the board. This information
is then passed to the kernel at boot time, via ACPI tables or DMI structures.

For embedded linux, no equivalent standard existed for a long time. The bootloader would be programmed
to pass a machine ID, and optional list of extra config items, to the kernel. This is the `GeminiA`
that we can see at the very start of the boot log.

Unfortunately, this machine ID does not seem to be referenced anywhere in open-source code; the
manufacturer possibly wrote their own board description without publishing it.

With the modern technique, instead of passing an opaque machine ID, the bootloader passes a pointer
to a Device Tree Blob (DTB), a binary structure containing a description of the hardware on the board.
But on this device, upgrading the bootloader is too risky; we do not have JTAG access, and thus
no easy way to reprogram the flash if we accidentally brick the bootloader.

Fortunately, there is a way to bypass this limitation; instead of having the kernel look for a DTB
at the address passed by the bootloader, we can hardcode a single DTB in the kernel itself. That
results in a non-portable kernel that only works on this specific board.

## Testing our kernel

The bootloader will let us load kernels from flash, tftp, disk, or xmodem (via the serial console).

For development, we don't want to wear out the flash chip by repeatedly rewriting the kernel.
at 19200bps, xmodem would take 20 minutes to load a 3MiB kernel. Disk is inconvenient, as we would need
to repeatedly change the wiring. TFTP is the only viable option.

The device, by default, uses an IP address of `192.168.123.33`. We connect the NAS to our computer's
network port, and configure the local computer to use `192.168.123.1`:
§
```
# ip addr add 192.168.123.1/24 dev eth0
```

We need a TFTP server for the NAS to fetch the test kernel. Tftpy, written in python, is simple enough
and does the job correctly.

```
# pip3 install tftpy
```

Start the TFTP server, publishing buildroot's output image directory:

```
# tftpy_server.py -r buildroot/output/images
```

Note that the TFTP server has to run as root (or with NET_CAP_BIND) to bind the default
TFTP port (udp 69), as the bootloader doesn't seem to offer a way to use an alternate port.

Reboot the NAS, and hit `Ctrl-C` to enter the bootloader menu. Choose option `5` to enter
command line mode, then use the `load` command to load the kernel into memory via TFTP.

We load the kernel at address 0x01600000, since it's what the bootloader does by default:

<pre>
==============================================================================
T: PCB Test!!!                              Z: BootLoader Update!!!
X: Upgrade Firmware                         L: All LED & Fan (Turn ON) Test!!!
M: All LED & Fan (Turn OFF) Test!!!         S: Shut down PCB!!!
Y: Upgrade MAC                              0: Reboot
1: Start the Kernel Code                    2: List Image
3: Delete Image                             4: Create New Image
5: Enter Command Line Interface             6: Set IP Address
7: Set MAC Address                          8: Show Configuration
9: Upgrade MAC                              F: Create Default FIS
I: Initialize IDE                           X: Upgrade Boot
Y: Upgrade Kernel                           Z: Upgrade Firmware
A: Upgrade Application                      R: Upgrade RAM Disk
N: Upgrade Bootloader(SDK)                  

=> Select: <b>5</b>


nas&gt;<b>load</b>
Usage: load -m [tftp | xmodem | disk | flash] -b [location]
  Load data to [location] by/from TFTP, xModem, disk, or flash

nas&gt;<b>load -m tftp -b 0x01600000</b>
TFTP Server IP Address: <b>192.168.123.1</b>
Image Path and name(e.g. /image_path/image_name): <b>/zImage.gemini-nas40</b>
TFTP Download /zImage.gemini-nas40 from 192.168.123.1 ..............................

Successful to download by TFTP! Size=3148379

nas&gt;
</pre>

Next, issue the `go` command to jump at the start of the loaded kernel and execute it:

<pre>
nas>go 0x01600000
Uncompressing Linux... done, booting the kernel.
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.13.9 (user@embedded) (arm-buildroot-linux-uclibcgnueabi-gcc.br_real (Buildroot 2021.08-rc3) 10.3.0, GNU ld (GNU Binutils) 2.36.1) #6 Sun Sep 5 13:40:32 CEST 201
[    0.000000] CPU: FA526 [66015261] revision 1 (ARMv4), cr=0000397f
[    0.000000] CPU: VIVT data cache, VIVT instruction cache
[    0.000000] OF: fdt: Machine model: Sarotech NAS-40
[    0.000000] Memory policy: Data cache writeback
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000000000000-0x0000000003ffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000000000-0x0000000003ffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000000000-0x0000000003ffffff]
[    0.000000] On node 0 totalpages: 16384
[    0.000000]   Normal zone: 128 pages used for memmap
[    0.000000]   Normal zone: 0 pages reserved
[    0.000000]   Normal zone: 16384 pages, LIFO batch:3
[    0.000000] pcpu-alloc: s0 r0 d32768 u32768 alloc=1*32768
[    0.000000] pcpu-alloc: [0] 0
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 16256
[    0.000000] Kernel command line: console=ttyS0,19200 rootdelay=10 root=/dev/sda1
[    0.000000] Dentry cache hash table entries: 8192 (order: 3, 32768 bytes, linear)
[    0.000000] Inode-cache hash table entries: 4096 (order: 2, 16384 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 55848K/65536K available (5120K kernel code, 612K rwdata, 1120K rodata, 1024K init, 201K bss, 9688K reserved, 0K cma-reserved)
[    0.000000] NR_IRQS: 16, nr_irqs: 16, preallocated irqs: 16
[    0.000000] clocksource: FTTMR010-TIMER2: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 76450417870 ns
[    0.000005] sched_clock: 32 bits at 25MHz, resolution 40ns, wraps every 85899345900ns
[    0.000147] Switching to timer-based delay loop, resolution 40ns
[    0.000950] Console: colour dummy device 80x30
[    0.001122] Calibrating delay loop (skipped), value calculated using timer frequency.. 50.00 BogoMIPS (lpj=250000)
[    0.001228] pid_max: default: 32768 minimum: 301
[    0.001472] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.001599] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.003897] CPU: Testing write buffer coherency: ok
[    0.006988] Setting up static identity map for 0x100000 - 0x100048
[    0.010781] devtmpfs: initialized
[    0.038253] random: get_random_u32 called from bucket_table_alloc+0xd4/0xec with crng_init=0
[    0.040967] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.041096] futex hash table entries: 256 (order: -1, 3072 bytes, linear)
[    0.041243] pinctrl core: initialized pinctrl subsystem
[    0.044980] NET: Registered protocol family 16
[    0.049011] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.057113] thermal_sys: Registered thermal governor 'step_wise'
[    0.070306] pinctrl-gemini 40000000.syscon:pinctrl: detected 3516 chip variant
[    0.070418] pinctrl-gemini 40000000.syscon:pinctrl: GLOBAL MISC CTRL at boot: 0x83c22037
[    0.070498] pinctrl-gemini 40000000.syscon:pinctrl: flash pin is set
[    0.075627] pinctrl-gemini 40000000.syscon:pinctrl: initialized Gemini pin control driver
[    0.159996] pl08xdmac 67000000.dma-controller: FTDMAC020 1.16 rel 1
[    0.160097] pl08xdmac 67000000.dma-controller: FTDMAC020 4 channels, has built-in bridge, AHB0 and AHB1, supports linked lists
[    0.160611] pl08xdmac 67000000.dma-controller: initialized 4 virtual memcpy channels
[    0.163877] pl08xdmac 67000000.dma-controller: DMA: PL080 rev0 at 0x67000000 irq 28
[    0.164409] Gemini SoC 3516 revision c3, set arbitration 00200030
[    0.171498] SCSI subsystem initialized
[    0.173238] libata version 3.00 loaded.
[    0.175254] usbcore: registered new interface driver usbfs
[    0.175576] usbcore: registered new interface driver hub
[    0.175892] usbcore: registered new device driver usb
[    0.188239] clocksource: Switched to clocksource FTTMR010-TIMER2
[    0.280504] NET: Registered protocol family 2
[    0.281142] IP idents hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.283927] tcp_listen_portaddr_hash hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.284106] TCP established hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.284260] TCP bind hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.284396] TCP: Hash tables configured (established 1024 bind 1024)
[    0.284928] UDP hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.285094] UDP-Lite hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.286821] NET: Registered protocol family 1
[    0.287001] PCI: CLS 0 bytes, default 32
[    0.291701] Initialise system trusted keyrings
[    0.293976] workingset: timestamp_bits=30 max_order=14 bucket_order=0
[    0.299624] Key type asymmetric registered
[    0.299707] Asymmetric key parser 'x509' registered
[    0.299999] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 251)
[    0.300077] io scheduler kyber registered
[    0.304150] ftgpio010-gpio 4d000000.gpio: FTGPIO010 @(ptrval) registered
[    0.306181] ftgpio010-gpio 4e000000.gpio: FTGPIO010 @(ptrval) registered
[    0.310869] ftgpio010-gpio 4f000000.gpio: FTGPIO010 @(ptrval) registered
[    0.321078] Serial: 8250/16550 driver, 1 ports, IRQ sharing disabled
[    0.328786] printk: console [ttyS0] disabled
[    3.483389] brd: module loaded
[    3.504499] gemini_sata_bridge 46000000.sata: SATA ID 00000e00, PHY ID: 01000100
[    3.548959] gemini_sata_bridge 46000000.sata: set up the Gemini IDE/SATA nexus
[    3.597376] pata_ftide010 63000000.ide: set up Gemini PATA0
[    3.631062] pata_ftide010 63000000.ide: device ID 00000500, irq 26, reg [mem 0x63000000-0x63000fff]
[    3.685562] pata_ftide010 63000000.ide: SATA0 (master) start
[    4.858322] gemini_sata_bridge 46000000.sata: SATA0 PHY not ready
[    4.894935] pata_ftide010 63000000.ide: brought 0 bridges online
[    4.930997] pata_ftide010 63000000.ide: failed to start port 0 (errno=-22)
[    4.972476] pata_ftide010: probe of 63000000.ide failed with error -22
[    5.012128] pata_ftide010 63400000.ide: set up Gemini PATA1
[    5.045818] pata_ftide010 63400000.ide: device ID 00000500, irq 27, reg [mem 0x63400000-0x63400fff]
[    5.100310] pata_ftide010 63400000.ide: SATA1 (master) start
[    5.158269] random: fast init done
[    6.288316] gemini_sata_bridge 46000000.sata: SATA1 PHY not ready
[    6.324929] pata_ftide010 63400000.ide: brought 0 bridges online
[    6.361032] pata_ftide010 63400000.ide: failed to start port 0 (errno=-22)
[    6.402582] pata_ftide010: probe of 63400000.ide failed with error -22
[    6.443646] physmap-flash 30000000.flash: no enabled pin control state
[    6.482872] physmap-flash 30000000.flash: no disabled pin control state
[    6.522577] physmap-flash 30000000.flash: initialized Gemini-specific physmap control
[    6.585532] physmap-flash 30000000.flash: physmap platform flash device: [mem 0x30000000-0x30ffffff]
[    6.640521] 30000000.flash: Found 1 x16 devices at 0x0 in 16-bit bank. Manufacturer ID 0x000001 Chip ID 0x002101
[    6.701593] Amd/Fujitsu Extended Query Table at 0x0040
[    6.732482]   Amd/Fujitsu Extended Query version 1.3.
[    6.762820] number of CFI chips: 1
[    6.783480] Searching for RedBoot partition table in 30000000.flash at offset 0xfe0000
[    6.851421] 6 RedBoot partitions found on MTD device 30000000.flash
[    6.889079] Creating 6 MTD partitions on "30000000.flash":
[    6.922091] 0x000000000000-0x000001000000 : ""
[    6.959190] 0x000000000000-0x000000020000 : "BOOT"
[    6.999948] 0x000000020000-0x00000038e0ee : "Kern"
[    7.028765] mtd: partition "Kern" doesn't end on an erase/write block -- force read-only
[    7.087916] 0x000000620000-0x000000620400 : "Ramdisk"
[    7.118329] mtd: partition "Ramdisk" doesn't end on an erase/write block -- force read-only
[    7.180488] 0x000000f20000-0x000000f40000 : "VCTL"
[    7.220105] 0x000000fe0000-0x000001000000 : "FIS directory"
[    7.275605] libphy: Fixed MDIO Bus: probed
[    7.316202] mdio-gpio mdio: failed to get alias id
[    7.352122] libphy: GPIO Bitbanged MDIO: probed
[    7.385537] tun: Universal TUN/TAP device driver, 1.6
[    7.430780] gmac-gemini 60000000.ethernet: Ethernet device ID: 0x000, revision 0x1
[    7.480141] gemini-ethernet-port 60008000.ethernet-port: probe 60008000.ethernet-port ID 0
[    7.532385] gemini-ethernet-port 60008000.ethernet-port: using a random ethernet address
[    7.740314] Marvell 88E1111 gpio-0:01: attached PHY driver (mii_bus:phy_addr=gpio-0:01, irq=POLL)
[    7.799145] gemini-ethernet-port 60008000.ethernet-port eth0: irq 31, DMA @ 0x0x60008000, GMAC @ 0x0x6000a000
[    7.862604] gemini-ethernet-port 6000c000.ethernet-port: probe 6000c000.ethernet-port ID 1
[    7.914942] gemini-ethernet-port 6000c000.ethernet-port: using a random ethernet address
[    7.964107] gemini-ethernet-port 6000c000.ethernet-port (unnamed net_device) (uninitialized): PHY init failed
[    8.024595] fotg210_hcd: FOTG210 Host Controller (EHCI) Driver
[    8.060972] fotg210-hcd 68000000.usb: Faraday USB2.0 Host Controller
[    8.099259] fotg210-hcd 68000000.usb: new USB bus registered, assigned bus number 1
[    8.147392] fotg210-hcd 68000000.usb: irq 29, io mem 0x68000000
[    8.188357] fotg210-hcd 68000000.usb: USB 2.0 started, EHCI 1.00
[    8.234951] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 5.13
[    8.284587] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    8.327944] usb usb1: Product: Faraday USB2.0 Host Controller
[    8.362447] usb usb1: Manufacturer: Linux 5.13.9 fotg210_hcd
[    8.396428] usb usb1: SerialNumber: 68000000.usb
[    8.430607] hub 1-0:1.0: USB hub found
[    8.453367] hub 1-0:1.0: 1 port detected
[    8.483536] fotg210-hcd 69000000.usb: Faraday USB2.0 Host Controller
[    8.521840] fotg210-hcd 69000000.usb: new USB bus registered, assigned bus number 2
[    8.570144] fotg210-hcd 69000000.usb: irq 30, io mem 0x69000000
[    8.615850] fotg210-hcd 69000000.usb: USB 2.0 started, EHCI 1.00
[    8.652976] usb usb2: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 5.13
[    8.702642] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    8.746048] usb usb2: Product: Faraday USB2.0 Host Controller
[    8.780563] usb usb2: Manufacturer: Linux 5.13.9 fotg210_hcd
[    8.814550] usb usb2: SerialNumber: 69000000.usb
[    8.849543] hub 2-0:1.0: USB hub found
[    8.872305] hub 2-0:1.0: 1 port detected
[    8.901523] usbcore: registered new interface driver uas
[    8.935492] usbcore: registered new interface driver usb-storage
[    8.971780] usbcore: registered new interface driver ums-datafab
[    9.008053] usbcore: registered new interface driver ums-freecom
[    9.044330] usbcore: registered new interface driver ums-isd200
[    9.080091] usbcore: registered new interface driver ums-sddr09
[    9.115854] usbcore: registered new interface driver ums-sddr55
[    9.151619] usbcore: registered new interface driver ums-usbat
[    9.193985] rtc-ftrtc010 45000000.rtc: registered as rtc0
[    9.226519] rtc-ftrtc010 45000000.rtc: setting system clock to 1970-07-14T05:18:22 UTC (16780702)
[    9.282814] gemini-poweroff 4b000000.power-controller: Gemini poweroff driver registered
[    9.336389] usb 1-1: new high-speed USB device number 2 using fotg210-hcd
[    9.378656] ftwdt010-wdt 41000000.watchdog: FTWDT010 watchdog driver enabled
[    9.446765] NET: Registered protocol family 10
[    9.484993] Segment Routing with IPv6
[    9.507331] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    9.552541] 8021q: 802.1Q VLAN Support v1.8
[    9.578083] Loading compiled-in X.509 certificates
[    9.609202] Waiting 10 sec before mounting root device...
[    9.700753] usb 1-1: New USB device found, idVendor=058f, idProduct=6387, bcdDevice= 1.0b
[    9.749890] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[    9.792796] usb 1-1: Product: Mass Storage
[    9.817434] usb 1-1: Manufacturer: Generic
[    9.842085] usb 1-1: SerialNumber: A5F54302
[    9.874590] usb-storage 1-1:1.0: USB Mass Storage device detected
[    9.915918] scsi host0: usb-storage 1-1:1.0
[   10.973482] scsi 0:0:0:0: Direct-Access     Generic  Flash Disk       8.07 PQ: 0 ANSI: 4
[   11.034878] sd 0:0:0:0: [sda] 7864320 512-byte logical blocks: (4.03 GB/3.75 GiB)
[   11.081430] sd 0:0:0:0: [sda] Write Protect is off
[   11.110288] sd 0:0:0:0: [sda] Mode Sense: 23 00 00 00
[   11.141973] sd 0:0:0:0: [sda] Write cache: disabled, read cache: enabled, doesn't support DPO or FUA
[   11.208339]  sda: sda1
[   11.232734] sd 0:0:0:0: [sda] Attached SCSI removable disk
[   19.715886] EXT4-fs (sda1): mounted filesystem with ordered data mode. Opts: (null). Quota mode: disabled.
[   19.774127] VFS: Mounted root (ext4 filesystem) readonly on device 8:1.
[   19.815778] devtmpfs: mounted
[   19.842527] Freeing unused kernel memory: 1024K
[   19.869919] Run /sbin/init as init process
[   19.894531]   with arguments:
[   19.912366]     /sbin/init
[   19.928652]   with environment:
[   19.947523]     HOME=/
[   19.961712]     TERM=linux
[   20.231498] EXT4-fs (sda1): re-mounted. Opts: (null). Quota mode: disabled.
Starting syslogd: OK
Starting klogd: OK
Running sysctl: OK
Initializing random number generator: OK
Saving random seed: [   21.591112] random: dd: uninitialized urandom read (512 bytes read)
OK
Starting network: [   22.374037] gmac-gemini 60000000.ethernet: allocate 512 pages for queue
[   22.428880] gemini-ethernet-port 60008000.ethernet-port eth0: Link is Up - 1Gbps/Full - flow control off
[   22.485794] gemini-ethernet-port 60008000.ethernet-port eth0: link flow control: none
[   22.618480] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
udhcpc: started, v1.33.1
[   22.838341] random: mktemp: uninitialized urandom read (6 bytes read)
OK
Starting dropbear sshd: OK

Welcome to Buildroot
nas login:
</pre>

## Future-proofing the system

This working setup has two drawbacks: the root device is hardcoded into the kernel, and
upgrading the kernel requires reflashing a MTD partition.

To avoid the root device problem, we can identify the correct device with a filesystem
label, but this feature is not supported by the kernel; it is handled by the userspace
in initrd/initramfs.

We can avoid the first problem by using a kexec-enabled kernel. This way, the kernel/initramfs
written in the flash can act as a sophisticated bootloader that can load another more recent
kernel from disk.

## Credits

This project would never have happened without the outstanding open work from the [Buildroot]
project, from [OpenGemini], and from [Linus Walleij].

[Bus Pirate]: https://dangerousprototypes.com/docs/Bus_Pirate
[Buildroot]: https://buildroot.org
[Devicetree]: https://devicetree.org
[OpenGemini]: http://opengemini.free.fr
[Linus Walleij]: https://dflund.se/~triad/krad
