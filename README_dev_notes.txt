gridfan devellopment README

--

This file includes observational notes and lessons learned from having written gridfan.

--

Gridfan is written as a bash script, which is "non-ideal" for this kind of application. If someone wants to re-write it in a real programming language, that would be great. I pooped this thing out in an hour or three. Feel free to steal the "gridfan" name for yourself if you do that.

--

Here's an accounting for how CAM communicates with the Grid+ v2 over the serial port:

	NOTE: I did most of this debugging under CAM version 3.0.4 unless otherwise noted.

	The CAM application starts up.

	CAM first sets the baud to 9600 and starts sending a "c0" ping every 200ms. 9600 is the Windows default baud setting, so I'm not sure if CAM is manually specifying 9600, or if it just tries the default baud on the first attempt.

	Sometimes at 9600 baud the Grid will send back an "18", but often just fails to reply. Since we are at the wrong baud anyway, this data is probably just mangled and meaningless.

	CAM then cycles it's baud to 256000 and continues to try and send it's "c0" ping at the same interval as before.

	Nothing ever comes back from the Grid at 256000 baud.

	It looks like CAM makes 20 ping attempts at each baud setting before it moves on to the next baud.

	CAM then cycles it's baud to 4800 and continues to try and ping the Grid.

	NOTE: CAM 3.5.30 seems to go directly to baud 4800, so it skips the baud scanning. Not sure when this change was made.

	Now that the baud is 4800, we will often get back a single "02" from the Grid on the first ping from CAM. I suspect 02 is an error code and is the result of CAM sending garbage at the Grid in the wrong baud.

	CAM then pings again, and this time the Grid will send a "21", which is apparently a success message.

	Once CAM gets back a "21" from the "c0" ping, it cycles through a series of four byte codes, three of which are GET messages (read info), and one of which is a SET message (write info).

	The order of these sent commands is always "85", "8a", "84" and "44" and CAM sends these command codes at a regular scan interval. The interval is about 80ms with a small variance. It seems like CAM sends a command and sleeps internally for this 80ms before it tries to read the data off of the controller. This means that with four command codes and six different fan IDs, it takes CAM about 2 seconds to perform a complete scan cycle (80ms x 24 codes = 1920ms + variance). This means that it takes CAM up to 2 full seconds before a fan speed change is committed to the controller, or new RPM data is read. Note that SET code 44 is only sent on changes, not every time, so a full cycle can have anywhere from 18 to 24 commands in it.

	Note that this scan interval process seems completely unnecessary based on my tests. The Grid controller often replies to a given command within 10-20ms and my gridfan script can do a complete 6-fan command get/set within 40ms. CAM just likes to be slow for fun I guess, or maybe they know something I don't know.

	For GET commands 85, 8a, and 84, the command is a two-byte code consisting of the command type and the fan ID. So, for GET command 85, CAM sends "85 01" for fan ID 1, and "85 06" for fan ID 6. CAM sends a GET for each fan ID in order, from 01 to 06, at it's regular 80-90ms interval.

	Command 85 is a GET command and has a single byte argument of the fan ID. I'm not sure what data is being returned but it has a fairly small resolution and variance. Maybe it's amperage? I don't know.

	Command 8a is a GET command that retrieves the fan speed in RPM.

	Command 84 is a GET command and has a single byte argument of the fan ID. I'm not sure what data is being returned but it's pretty wild and fluctuates even when no fan is plugged into the port.

	Command 44 is a SET command. This command is a 7-byte code in a format like "44 XX c0 00 00 YY YY", where XX is the fan ID code (01-06) and YY YY is the desired fan port voltage converted from decimal to hex. For example, a normal 3-pin fan connector at 100% power is 12V, which is code "0c 00", and 50% power is 6V at "07 00".

	The controller reply for the 85, 8a, and 84 GET commands is always a 5-byte code in a format of "c0 00 00 XX XX" where XX XX is some return data which might be the fan RPM speed or some power-related data.

	The controller reply for the 44 SET command is always a 1-byte return code of "01", which is probably just a success message. On a few occasions I've seen the Grid send back an "02", usually after sending an invalid command or other garbage during testing.

	NOTE: In CAM 3.5.30 and possibly earlier versions, and possibly later versions, CAM incorrectly will try to set fan speeds with codes "02 xx" and "03 xx", which the Grid controller does not support. The lowest supported speed setting for the Grid, other than 0, is 20%, which uses code "04 00". The NZXT devs apparently don't even know how their own hardware works.

--

Command Codes Summary
	c0 = Ping/GET-status
	84 = GET unknown. May be voltage related.
	85 = GET unknown. Amperage?
	8a = GET RPM/speed
	44 = SET speed

--

