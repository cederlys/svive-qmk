Dumping the stock firmware
==========================

* Buy an ST-LINK v2

* Solder som leads to the relevant test points on the PCB.  See
  dump-st-link.png.  Connect BOOT to GND.  Connect the rest to the
  ST-LINK:

    Kbd	    ST-LINK v2

    VCC   - 3.3V
    GND   - GND
    SWCLK - SWCLK
    SWDIO - SWDIO

* Install OpenOCD

    apt-get install openocd

* Get some scripts:

    git clone git@github.com:gloryhzw/qmk_tool.git

* Start openocd:

    cd qmk_tool/firmware
    openocd -f stlink.cfg -f sonix.cfg

 * Follow the instructions in DumpTutorial.md, which I found on the
   SonixQMK Discord server.  It allegedy exists on some Wiki as well,
   but I can't find it.  Note that you need to edit the dump-memory.py
   in the middle of the process, after you have remapped the firmware
   flash.  Revert the change afterwards, in case you want to dump the
   firmware from some other keyboard in the future.

Reverse engineering the keyboard
================================

I've already done it for you.  The result is in svive.matrix.  Q.txt
may also be helpful if you need to locate the transistors, labeled Qnn
or QXnn on the PCB.

Compiling firmware
==================

* Create a venv for SviveQMK:

    python3 -m venv ~/.venv/svive-qmk

* Activate it:

    . ~/.venv/svive-qmk/bin/activate

* Install qmk:

    python3 -m pip install qmk

* Set it up:

    qmk setup -b sn32_master_openrgb cederlys/qmk_firmware

  This will create ~/qmk_firmware and install some things in the venv.

* Compile:

    qmk compile -kb svive/tritium_full -km default

  or, with swapped Fn and Win keys:

    qmk compile -kb svive/tritium_full -km ceder

Flashing
========

* Get the flasher:

    git clone git@github.com:SonixQMK/sonix-flasher.git

* Create and activate a new venv (you can't share it with qmk!):

    python3 -m venv ~/.venv/sonix-flasher
    . ~/.venv/sonix-flasher/bin/activate

* Install requirements:

    cd ~/rgit/sonix-flasher
    pip install wheel
    pip install -r requirements.txt

* Run it:

    cd ~/rgit/sonix-flasher
    sudo -s
    . ~ceder/.venv/sonix-flasher/bin/activate
    fbs run

* From stock firmware, use "Reboot to Bootloader [HFD]" to reach the
  bootloader.  From the QMK firmware, press Fn-ESC to reach the
  bootloader.  In all cases, you can also short the BOOT pin and GND
  while plugging in the keyboard.

* Select SN32F24x.  Select QMK offset 0x00.  This keyboard should not
  use a jumploader.

* To install QMK: Press "Flash QMK...", and select the firmware you
  compiled.

* To revert to stock firmware, press "Revert to Stock Firmware", and
  select the firmware you downloaded.  Included are five stock
  firmwares I've dumped:

    FULBLK00321250481.bin (Triton Full) (remap was needed)
    FULBLK00321250740.bin (Triton Full) (contains a real SN32F248BFG!)
    FULBLK00422090684.bin (Triton Full)
    FULBLK00422090686.bin (Triton Full)
    60BLKG00321200502.bin (Triton 60%) (remap was needed)

Controlling the LED:s
=====================

Install OpenRGB.

Add this udev rule:

echo 'SUBSYSTEMS=="usb|hidraw", ATTRS{idVendor}=="0c45", ATTRS{idProduct}=="8006", TAG+="uaccess", TAG+="Svive_Tritium_Full"' > /usr/lib/udev/rules.d/60-openrgb-svive.rules

Start openrgb.  Find "Settings" => "OpenRGB QMK Protocol".  Press
"Add".  Enter USB VID (0c45) and USB PID (8006).  Restart openrgb.

Now, you should find the keyboard in openrgb.
