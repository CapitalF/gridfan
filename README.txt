gridfan README

--

ABOUT

Gridfan is a simple/stupid controller script for the NZXT Grid+ v2 fan controller. It's command-line only. There is no GUI.

Gridfan is written as a bash script, and badly at that. If someone wants to re-write it in a real programming language, that would be great. I pooped this thing out in an hour or three. Feel free to steal the "gridfan" name for yourself if you do that.


Gridfan can:
	Ping the controller to make sure it's alive.
	Show you what your fan speeds are in RPM.
	Set your fan speeds on a per-fan basis.

Gridfan can not:
	Automatically adjust fan speeds based on sensors/temperature.
	Tell you if your fans have died.

--

INSTRUCTIONS

First, plug the controller into your system. The white LED will light up when both the power and USB are connected. Then check the output of dmesg near the end. You should see a new device plugged in.

Check lsusb too. Mine shows up as a "04d8:00df Microchip Technology, Inc." device.

The kernel module cdc_acm recognizes this as a serial ACM device, so you will probably see a new device node right away (Probably /dev/ttyACM0)



Next, we need to secure the device file with a udev rule. Nobody but the superuser/admin should be able to read or write to the device. We use the group "staff" here to limit access, so make sure your user account is in the "staff" group. Or, use a different group that makes sense to you.

If your system doesn't have or use udev, just set the ownership and permissions on the device file manually and modify the GRID_DEV parameter in the control script.

Unfortunately, identifying this device as a Grid+ is somewhat difficult. NZXT developers were lazy slobs and didn't give the device a unique USB ID like they should have. This makes creating a good udev rule difficult. However, mine had a serial number which was useful in uniquely identifying it.

I only have my one Grid+ available, so I don't know if the serial number is actually unique to each device or it could be global across all devices. You will need to find your serial number and put it into the udev rule yourself.

To get the serial number, do this command:
	# NOTE: Replace /dev/ttyACM0 with your device file (it'll probably be /dev/ttyACM0)
	udevadm info -a -p $(udevadm info -q path -n /dev/ttyACM0) | egrep "ATTRS{serial}==\"[0-9]{10}\""

Now you should have your serial number. Put it in the udev rule below (replace the "00016xxxxx" thing).

Do this in a root shell on your system:

echo 'ACTION=="add", SUBSYSTEMS=="usb", ATTRS{product}=="MCP2200 USB Serial Port Emulator", ATTRS{serial}=="00016xxxxx", SUBSYSTEM=="tty", GROUP="staff" MODE="0660" SYMLINK+="GridPlus0"' > /etc/udev/rules.d/90-NZXT-Grid+_v2.rules

This should have created a new udev rule file: /etc/udev/rules.d/90-NZXT-Grid+_v2.rules

This rule creates a /dev/GridPlus0 symlink to the/dev/ttyACMx device, which we will use in place of ttyACMx.

Finally, you need to unplug/replug the device again so that udev will apply the new rule.

Do an 'ls /dev/GridPlus0' to make sure the device is there and that the udev rule worked. Note the permissions and ownership on the ttyACM file too. If your user account can't read and write to the device file, you won't be able to use the control script.

Assuming your controller is plugged in, the udev rule worked, and the Linux kernel loaded the driver for the serial device, you are good to start using the gridfan control script.

IMPORTANT: The first time you use the controller on a PC after boot, it needs to be "initialized". Run the "gridfan init" command to do this. You only need to initialize the controller once after it boots up.

--

NOTES

The fan ports on the device are addressed as such, if looking directly at the "Grid+" logo on the front:
1       4
2 GRID+ 5
3       6

You can actually read the fan port IDs from the board by peeking inside.

It would be a really good idea to write down a map of which fans on your PC correspond to which port on the controller.

The default controller configuration seems to be that all fans are set to 40% speed.

The controller does not have any persistent memory. Any time you unplug it's power, it loses it's old config and resets to defaults.

Any time a port is set from 0%/off to another speed, the controller first spins the fan up to 100% then sets the new speed. This does not happen if the fan speed is already set to something other than 0.

Trying to set speeds below 20% seems to have no effect. The controllers must have built-in ranges.

I don't know if you can actually set arbitrary speeds like 57% instead of 55% because I didn't try and didn't think it was important. I also have doubts that the controller supports it. The gridfan script only allows speeds of 0, and 20-100 in increments of 5. This should be good enough.

Gridfan can't (currently) show you any power usage info like CAM does. I didn't find that information very useful anyhow.

I never observed the controller arbitrarily sending data to the PC. It's simply a query-reply relationship.

If there is no fan plugged into a port on the controller, the controller reports a fan speed of 0.

IMPORTANT: If a fan stops moving, the controller continues to report the last known "good" speed instead of 0. This means that if a fan is suddenly stopped by a physical obstruction, the controller continues to report it as moving, which is wrong. If a fan slows down to a crawl before stopping, the controller will probably report the last known low speed, which should be a good indication that a fan is dying or has already died. I consider this a controller bug.

The controller needs to be initialized/sycned up with the PC each time it boots. I suspect this is needed by the serial UART controller as part of an auto-speed negotiation mechanism, but I'm not sure.


