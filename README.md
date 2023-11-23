Dumping the stock firmware
==========================

* Buy an ST-LINK v2

* Solder som leads to the relevant test points on the PCB.  See
  dump-st-link.png.  Connect BOOT to GND.  Connect the rest to the
  ST-LINK:

```
    Kbd	    ST-LINK v2

    VCC   - 3.3V
    GND   - GND
    SWCLK - SWCLK
    SWDIO - SWDIO
```

* Install OpenOCD

    ```apt-get install openocd```

* Get some scripts:

    ```git clone git@github.com:gloryhzw/qmk_tool.git```

* Start openocd:

```
    cd qmk_tool/firmware
    openocd -f stlink.cfg -f sonix.cfg
```

 * Follow the instructions in DumpTutorial.md, which I found on the
   SonixQMK Discord server.  It allegedy exists on some Wiki as well,
   but I can't find it.  Note that you need to edit the dump-memory.py
   in the middle of the process, after you have remapped the firmware
   flash.  Revert the change afterwards, in case you want to dump the
   firmware from some other keyboard in the future.  For you
   convenience, here is a patch that performs the necessary edits:

```
--- dump-memory.py~	2023-08-24 23:22:38.810453463 +0200
+++ dump-memory.py	2023-11-20 00:00:37.107395829 +0100
@@ -30,8 +30,8 @@
     outf = open(args.file, "wb")
 
     with Telnet(host, port) as tn:
-        tn.read_until(b"> ")
-        tn.write(b"reset halt\n")
+        # tn.read_until(b"> ")
+        # tn.write(b"reset halt\n")
 
         for addr in range(args.start, args.start + args.size, 4):
             tn.read_until(b"> ")
```

Reverse engineering the keyboard
================================

I've already done it for you.  The result is in svive.matrix.  Q.txt
may also be helpful if you need to locate the transistors, labeled Qnn
or QXnn on the PCB.

To see the pinout of the MCU, run

    git clone git@github.com:SonixQMK/Mechanical-Keyboard-Database.git

and look at docs/MCU chip/SN32F240B_V1.9_EN.pdf.

These keyboards contain a HFD2201KBA or SN32F248B MCU.  They are
compatible.

Disassembling the keyboard
==========================

* Remove 3 screws from the underside of the keyboard.

* Gently remove the top cover by pulling the front and back away from
  the rest of the keyboard.

* Remove at least the following keycaps to get access to several small
  screws:

```
    F3 F4 F6 F7 F10 F11 ScrLk Pause
    TAB E R U I Å ^ DEL END
    CapsLock D J Ä
    LeftCtrl Fn Space AltGr Menu RightCtrl Down Right
    Numpad keys: * - + Enter 0 .
```

* Remove the screws.

* Gently lift the PCB a short bit.  Remove the USB cable from the
  base.  Lift the PCB and turn it over.

  You now have access to the bottom of the PCB.  If all you needed was
  to unbrick it, short the BOOT and GND pins (see dump-st-link.png)
  and plug in the USB cable.

  However, if you want access to the other side of the PCB, continue below:

* Remove all the keycaps and switches.  (Use the supplied tool.  The
  switches are hotpluggable, so you can simply lift them.)

* Remove 8 screws from the PCB.

* Gently remove the PCB from the steel plate.

Compiling firmware
==================

* Create a venv for SviveQMK:

```
    python3 -m venv ~/.venv/svive-qmk
```
* Activate it:

```
    . ~/.venv/svive-qmk/bin/activate
```

* Install qmk:

```
    python3 -m pip install qmk
```

* Set it up:

```
    qmk setup -b svive-tritium-full-1 cederlys/qmk_firmware
```

  This will create `~/qmk_firmware` and install some things in the venv.

* Compile:

```
    qmk compile -kb svive/tritium_full -km default
```

  or, with swapped Fn and Win keys:

    qmk compile -kb svive/tritium_full -km ceder

Flashing
========

* Get the flasher:

```
    git clone git@github.com:SonixQMK/sonix-flasher.git
```

* Create and activate a new venv (you can't share it with qmk!):

```
    python3 -m venv ~/.venv/sonix-flasher
    . ~/.venv/sonix-flasher/bin/activate
```

* Install this patch.  Otherwise, it won't compile on a modern system.

```
diff --git a/requirements.txt b/requirements.txt
index 93ea466..c35de51 100644
--- a/requirements.txt
+++ b/requirements.txt
@@ -1,7 +1,7 @@
 altgraph==0.17
 fbs==0.8.6
 future==0.18.2
-hidapi==0.9.0.post2
+hidapi==0.14.0
 macholib==1.14
 pefile==2019.4.18
 PyInstaller==3.4
```

* Install requirements:

```
    cd ~/rgit/sonix-flasher
    pip install wheel
    pip install -r requirements.txt
```

* Run it:

```
    cd ~/rgit/sonix-flasher
    sudo -s
    . ~ceder/.venv/sonix-flasher/bin/activate
    fbs run
```

* From stock firmware, use "Reboot to Bootloader [HFD]" to reach the
  bootloader.  From the QMK firmware, press Fn-ESC to reach the
  bootloader.  In all cases, you can also short the BOOT pin and GND
  while plugging in the keyboard.

* Select SN32F24x.  Select QMK offset 0x00.  This keyboard should not
  use a jumploader.

* To install QMK: Press "Flash QMK...", and select the firmware you
  compiled.

* To revert to stock firmware, press "Revert to Stock Firmware", and
  select the firmware you downloaded.

Controlling the LED:s
=====================

Install OpenRGB.

Add this udev rule:

```
echo 'SUBSYSTEMS=="usb|hidraw", ATTRS{idVendor}=="0c45", ATTRS{idProduct}=="8006", TAG+="uaccess", TAG+="Svive_Tritium_Full"' > /usr/lib/udev/rules.d/60-openrgb-svive.rules
```

Start openrgb.  Find `Settings` => `OpenRGB QMK Protocol`.  Press
`Add`.  Enter USB VID (0c45) and USB PID (8006).  Restart openrgb.

Now, you should find the keyboard in openrgb.
