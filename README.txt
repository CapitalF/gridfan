gridfan README

--

ABOUT

Gridfan is a simple controller script for the NZXT Grid+ v2 fan controller on GNU/Linux. It's command-line only; there is no GUI.

Gridfan can:
	Ping the controller to make sure it's alive.
	Show you what your fan speeds are in RPM.
	Set your fan power/speed on a per-fan basis.

Gridfan can not:
	Automatically adjust fan speeds based on sensors/temperature.
	Tell you if your fans have died.

This README file is user-oriented. If you are a developer interested in how gridfan or the Grid+ v2 works, please see the gridfan_dev_README.txt file.

--

INSTRUCTIONS

First, plug the controller into your system. The white LED will light up only when both the power and USB are connected.

Next you will want to confirm that the device is detected by Linux by tailing dmesg/journalctl.

Check lsusb too. Mine shows up as a "04d8:00df Microchip Technology, Inc." device.

The kernel module cdc_acm recognizes this as a serial ACM device, so you will probably see a new device node right away (/dev/ttyACM0).

Next, we need to secure the device file. Shutting down fans on a system could cause it to overheat, so only the superuser/admin should be able to read or write to the device. We use the group "staff" here to limit access, so make sure your user account is in the "staff" group. Or, use a different group that makes sense to you.

The best way to configure the device node on most modern systems is with a udev rule. If your system doesn't have or use udev, just set the ownership and permissions on the device file manually and modify the GRID_DEV parameter in the control script.

Unfortunately, identifying this device as a Grid+ is somewhat difficult. NZXT developers were lazy slobs and didn't give the device a unique USB ID like they should have. This makes creating a good udev rule difficult. However, The Grid+ has a serial number which can be used to uniquely identifying it.

To get the serial number, do this command:
	# NOTE: If your device file is something other than /dev/ttyACM0, you will need to change that here.
	udevadm info -a -p $(udevadm info -q path -n /dev/ttyACM0) | egrep "ATTRS{serial}==\"[0-9]{10}\""

Now you should have your serial number. Put it in the udev rule below (replace the "0001234567" thing).

Do this in a root shell on your system:

echo 'ACTION=="add", SUBSYSTEMS=="usb", ATTRS{product}=="MCP2200 USB Serial Port Emulator", ATTRS{serial}=="0001234567", SUBSYSTEM=="tty", GROUP="staff" MODE="0660" SYMLINK+="GridPlus0"' > /etc/udev/rules.d/90-NZXT-Grid+_v2.rules

This should have created a new udev rule file: /etc/udev/rules.d/90-NZXT-Grid+_v2.rules

This rule creates a /dev/GridPlus0 symlink to the/dev/ttyACMx device, which we will use in place of ttyACMx.

Finally, you need to unplug/replug the device again so that udev will apply the new rule.

Do an 'ls /dev/GridPlus0' to make sure the device is there and that the udev rule worked. Note the permissions and ownership on the ttyACM file too. If your user account can't read and write to the device file, you won't be able to use the control script.

Assuming your controller is plugged in, the udev rule worked, and the Linux kernel loaded the driver for the serial device, you should be ready to use the gridfan control script.

IMPORTANT: The first time you use the controller on a PC after boot, it needs to be initialized. Run the "gridfan init" command to do this. You should only need to initialize the controller once when it power up.

To see the valid command arguments and examples, do this:
	gridfan --help

--

NOTES

IMPORTANT: If a fan stops moving, the controller continues to report the last known "good" speed instead of 0. This means that if a fan is suddenly stopped by a physical obstruction, the controller continues to report it as moving, which is wrong. If a fan slows down to a crawl before stopping, the controller will probably report the last known low speed, which should be a good indication that a fan is dying or has already died. I consider this a controller bug.

Any time a port is set from 0%/off to another speed, or when the device powers up, the controller first spins the fan up to 100% then sets the new speed. This does not happen if the fan speed is already set to something other than 0.

The default controller configuration seems to be that all fans are set to 40% speed.

The controller does not have any persistent memory. Any time you unplug it's power, it loses it's old config and resets to defaults.

Trying to set speeds below 20% seems to have no effect. The controllers must have built-in ranges.

I don't know if you can actually set arbitrary speeds like 57% instead of 55% because I didn't try and didn't think it was important. I also have doubts that the controller supports it. The gridfan script only allows speeds of 0, and 20-100 in increments of 5.

Gridfan can't (currently) show you any power usage info like CAM does. I didn't find that information very useful anyhow.

If there is no fan plugged into a port on the controller, the controller reports a fan speed of 0.

The controller needs to be initialized/sycned up with the PC each time it boots. I suspect this is needed by the serial UART controller as part of an auto-speed negotiation mechanism, but I'm not sure.

The fan ports on the device are addressed as such, if looking directly at the "Grid+" logo on the front:
1       4
2 GRID+ 5
3       6

You can actually read the fan port IDs from the board by peeking inside.

It would be a really good idea to write down a table of which fans on your PC correspond to which port on the controller.

I have read reports that PWM fans can have issues with this controller because they require more than the default 40% power setting. Obviously, just set the power to such fans as high as is necessary to keep them alive.
	https://www.reddit.com/r/NZXT/comments/4k781h/grid_v2_cant_actually_control_the_fans/
	http://camfeedback.nzxt.com/forums/252256-bugs-issues/suggestions/11007621-grid-v2-spins-up-and-down

