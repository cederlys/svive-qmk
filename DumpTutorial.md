# Tutorial for dumping keyboard firmware of 240(B) chips
## Needed Tools/software
* Stlink-v2 (or some other debug thingy with a swd interface and openocd support)
* Python 3
* Openocd
* Telnet for troubleshooting + remapping if necessary
* A clone of [this](https://github.com/smp4488/dk63) repo
* Some form of hex inspector is useful

## Getting the chip connected
* Make sure the usb connection is disconnected
* Find gnd, vcc, swdio and swclk connections on the board and solder wires to them
	* Datasheets for the chips can be found [here](https://github.com/SonixQMK/Mechanical-Keyboard-Database/tree/main/docs)
* Find a connection to the boot pin (no need for a wire)
* Leave the st-link unplugged for now
* Connect vcc to the 3.3V of the stlink and gnd, swdio and swclk to the right other pins
* Now while connecting the boot pin to ground: plug in the stlink
* Start openocd with the vs11k09a-1.cfg from the cloned repo and verify that the chip is recognized

## General infos about dumping
The arm cortex has a read protection so a normal flash read via openocd is not possible.
The solutions to this is using a ldr instruction that is part of the bootloader to get access to the data in the flash.

Read more about it [here](https://blog.includesecurity.com/2015/11/firmware-dumping-technique-for-an-arm-cortex-m0-soc/)

The usually used ldr for these chips is at 0x1FFF0118 in memory, but may vary with newer bootloader versions. You will have to find a different ldr if 0x1FFF0118 does not work for you.

## Dumping the bootloader (Not the firmware yet!)
* The bootloader starts at 0x1FFF0000 in memory
* Leave openocd running in the background
* cd into the *dump* directory of the repo and run the following command:\
```python3 dump-memory.py 0x1FFF0000 0x1000 bootrom.bin --openocd localhost:4444 --ldr-gadget 0x1FFF0118 --reg1 r4 --reg2 r0```
* This should now have created a bootrom.bin in the dump folder

## Checking if the bootrom dump is valid
* Yeah that's kinda difficult
* One thing thats usually there is ```ÓxÈxÁúØR``` (```D3 78 C8 78 C1 FA D8 52```) about at 0x00000100
* The bootrom is not that important anyway, but useful for the next step ;)

## Dumping the firmware
* In theory this is done by using the dumpscript again (but starting at 0x00000000):\
```python3 dump-memory.py 0x00000000 0x10000 firmware.bin --openocd localhost:4444 --ldr-gadget 0x1FFF0118 --reg1 r4 --reg2 r0```
* Now you may be lucky and it's actually the firmware
* Or the start of the dumped file matches the bootrom.bin exactly....then it is the bootloader again and you have to do the **remap** to get the firmware

## Checking if the firmware dump is valid
* There are a couple of key properties:
	* **The binary ends with ```55 55 AA AA```**
	* The binary has the string ```S.O.N.i.X...U.S.B. .D.E.V.I.C.E...D.E.M.O``` somewhere at about 0x00007EC0
	* The binary might have ```«FTF]F¬B``` (```AB 46 54 46 5D 46 AC 42```) at about 0x000000D0
* If you found these properties you probably have a valid dump
* **>>You did it!<<**
* *Please* submit the dump [here](https://github.com/SonixQMK/Mechanical-Keyboard-Database/tree/main/stockFWs)

## Doing the remap
* If you just got the bootrom again when dumping at 0x00000000 then you need to remap the firmware flash to 0x00000000 before dumping.
* This step a bit trickier than the others.
* To remap the flash the value **0xA5A50001** needs to be written to the address **0x40060024**
* So a str operation to write stuff is required: **0x1FFF011A** might be a valid one (```str	r4, [r1, #0]```)
* So do the following to remap:
	* Open telnet with ```telnet localhost 4444```
	* It should show that is is connected to openocd
	* Execute the following commands:
```
reg r4 0xA5A50001	# Sets the value to write
reg r1 0x40060024	# Sets the adress to write to
reg pc 0x1FFF011A	# Sets the program counter to our str operation
step				# Executes the operation
```
* Now exit telnet
* Comment out ```tn.write(b"reset halt\n")``` in line 37 in the dump script (I do not know if this is actually necessary, I did not have to do the remap :P)
* Run the command for dumping again:\
```python3 dump-memory.py 0x00000000 0x10000 firmware-remap.bin --openocd localhost:4444 --ldr-gadget 0x1FFF0118 --reg1 r4 --reg2 r0```
* Hope that your firmware is valid now :D

## Troubleshooting
**Ask in the Sonix Keyboard Hacking Discord**

### Finding a ldr operation (only neccesary if the provided one does not work for you)
* Basically try and error.
* In telnet set all registers to 0x0 with ```reg r0 0; reg r1 0;``` etc...
* Set the programm counter with ```reg pc <address>```\
Start with about 0x1FFF0100
* See if any of the registers changed (just ```reg``` to see a list)
* You generally want an instruction that changes one register to += 8 and another register with to random value.
* The register with the += 8 is probably the input address register of the load instruction, the other register the returned value
* Repeat with other addresses for the pc until you found an intruction that 'looks right' and try dumping with the dumpscript (adjust ```--ldr-gadget <pc address> --reg1 <return value register> --reg2 <input address register>```)

### Finding a str operation
* This one is way easier as long as you have a valid bootrom dumped
* Make sure arm-none-eabi-binutils (probably only linux sry) is installed
	* You might be able to use a different disassembler, but none worked properly for me
* Run\
```arm-none-eabi-objdump -D -bbinary -marm bootrom.bin -Mforce-thumb > bootrom-dis.s```
* Search for a str operation in the disassebled code and note its address offsetted by 0x1FFF0000
* Use the offsetted address for remapping the flash

**Thanks for reading, hope it helps :)**
