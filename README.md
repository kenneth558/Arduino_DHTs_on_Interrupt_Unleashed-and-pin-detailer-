      Arduino DHTs on Interrupts Unleashed          (previously named Arduino_DHTs_on_Interrupt_steroids)

Files you need to place in your sketch directory:

Arduino_DHTs_on_Interrupts_Unleashed.ino   

ISRs.h

misc_maskportitems.h

structs.h

** External 5VDC power supply for your sensors may be required: Save yourself some frustration and use one while getting your system stable, THEN try without it.  Otherwise you'll not really know if the lack of an external power supply is the cause of instabilites that may occur**

**NOTE WINDOWS USERS: If the host computer is MS Windows, it requires a CR line ending besides the LF that Linux requires.  This sketch contains a prorammed feature to accommodate MS Windows line endings if you install a high-value resistor (like 1 Mohm) between pins A0 and LED_BUILTIN.  On the UNO these pins are labeled A0 and 13.  Refer to the pin detail printout when this sketch first starts if you need that info for other boards. **

I DO NOT YET HAVE THIS SKETCH FUNCTIONING ON THE LEONARDO.  I suspect my ISR cycling logic causes problems where there is only a single ISR.


The main purpose of the interrupt is to avoid the time waste of polling.  This project was developed because other DHT libraries that advertised interrupt useage still unnecessarily rely on polling as well, making their claimed use of interrupts virtually meaningless, not to mention their sad (and now unnecessary) limitation of only a single DHT device per board.  You can read all the sensors you want to with proper application of the interrupt code posted here.  Connect them all up and go - the setup() code searches every digital pin and detects every device for you.  The sensors found on PCI pins will start streaming their data after the forced rest period, and readings will then become available.  The last 5 readings (determined by the confidence level variable) from every device are stored along with the age of each sensor's most recent reading.  Devices connected to pins NOT supported by PCIs will still be shown in the startup print, but this code won't operate them - you'll have to use legacy code to read those sensors.  Just make sure that, if the legacy code uses interrupts, it doesn't disable interrupts for too long so as to interfere with this code.  Empiric experimentation should enable you to adequately assess compatibility.

Optionally, add your main functionality to void loop() if you want more than the super-simple demo that is only intended to be a proof-of-concept.  This project is not a formal library, just some ISR code, ISR launcher code in setup(), DHT device detection and ISR pin change detection code, and a main loop() hello world demo with rudimentary examples of how RH and temperatures are read from the data structures.  

If you just want to see which pins of your board are served by PC ISRs and which ISR it is, this sketch is exactly what you need as well.  It is not hindered by the incomplete OEM definitions that support digitalPinToPCICRbit() and digitalPinToPCMSKbit() functions, as is the case with the official Mega 2560 IDE environment ISR1.  Screen shots of that functionality during board startup are posted here for the boards I had at hand.  Each Arduino type has a different amount of available RAM, and some boards are thusly limited to less than the maximum number of devices.  Note that LED_BUILTIN and any other LEDs will preclude their pins from being used for DHT devices because they prevent voltage pull-up.

Due to its lack of code beauty, the only claims I'll stand behind right now is that this project is PROOF OF CONCEPT ONLY.  There is a known 9 sensor limitation on Arduinos with limited memory size, including the UNO and Nano, even though those boards have more PCI pins than other boards.  If you go beyond the sensor limit on these boards, the sketch will eventually fail.  I've used nearly all the program memory as well in those boards, so little is left for your sketch in void loop() unless you clean out functionalities you decide you can do without. The Mega, with its increased memory size, can handle all its PCI pins (except D0) being populated for a total of 17 sensors.  That board actually has 18 PCI pins, but Pin D0 was not tested with a sensor because I needed it for serial communications.  If you ensure no serial communications occur, the Mega will probably handle the full complement of 18 sensors.  

IMPORTANT NOTE:  This C++ sketch version has a known limitation of one sensor per ISR minimum.  This is due to imperfect ISR cycling logic, and I'm leaving it this way in the C++ sketch.  The ASM version (not for free) will address these and the other limitations, but I don't intend to embark on that version until I deal with other priorites I have.  I am willing to make the ASM version a priority if YOU express a serious interest in it by filling out an issue report in the "Issues" tab above.  Serial communications on the Leonardo will get fixed soon, but you may just as well fix that yourself.

This project utilizes two different interrupt service routine types:  one "Pin Change Interrupt" pin is required for every DHT device connected serviced by an PC ISR code segment, and an additional over-all process-monitoring/watchdog code segment that resides in "Timer/Counter Compare Match A" interrupt code. (This code does not use any of the "2-wire Serial Interface", "External Interrupt", "Watchdog", "Timer/Counter Overflow", etc. types of interrupts.)

The pin change interrupt service routines come into play after the compare-match interrupt code triggers a device.  In response, a device will stream its data back while the pin change ISR code stores the timestamp of each falling edge until the final one.  Then the ISR sends the device into the wait state.  Note that storing timestamps in their raw unsigned long type is one memory useage vs speed compromise.  To gain a few more bytes of variable RAM at the expense of some extra execution time in the Pin Change ISRs, a person could move the translation function to the Pin Change ISRs.  For now, I am deciding against providing that option because it would have to be run-time determined every ISR call and thus add excess detection and decision execution time to a project like this meant for general consumption.

The compare-match code serves the other purposes, including translating timestamps into bits then into RH/temperature data, invalidating non-responding devices, etc.  Only a portion of it is executed every millisecond when it is launched, so the math won't equate as you might hope - translating timestamps to bits will require one ISR execution ( a whole millisecond ) for every bit and for every device.  In other words, if all 3 ISRs have incoming data streams, 3 time 42 = 126.  That is 126 ms time windows AT LEAST for timestamp translation alone.   This "small bite each pass" technique is done so that main code of your making will be allowed to run as well as can be allowed.

Note that a typical Arduino board may have 3 Pin Change Interrupts with Pin Change Interrupt Service Routines, so if the pins are chosen to do so, 3 devices may ALL be feeding their data streams back to the board simultaneously.  When those three devices are finished with their data streams, the next device in line on each ISR will get triggered, assuming their resting times are completed.  Note this project can actually handle the 4 Pin Change interrupt streams that some devices are capable of, so the example of 3 is just a typical scenario rather than its true limit.

The compare-match interrupt code routine (ISR) is triggered every millisecond, and the pin change ISRs are triggered every one hundred microseconds, more or less, and MUST NOT BE DELAYED BY ANOTHER PIN CHANGE ISR NOR BY LEAVING INTERRUPTS TURNED OFF INSIDE THE COMPARE-MATCH ISR.  Therefore, the pin change ISRs must take as little execution time as possible, and the compare-match ISR must keep interrupts enabled as much as possible during its execution while ensuring its execution time is WELL less than a millisecond.  This project was written under the assumption that the PC interrupts are higher priority than the compare-match interrupt, but that assumption may not really be necessary since interrupts are kept enabled everywhere that atomic writes of shared memory locations are not at stake.

This is C++, so it is free.  An assembly code version awaits while I hope to see an interest demonstrated by [a] potential customer/employer[s].  Please let me know if that is you!

I merely got this functional, not beautiful.  It is NOT intended for first-project newbies who need handholding creating their own void loop() based on my demo loop().  That said, feel free to try anyway.  Please ensure your main code does not disable interrupts for more than roughly a microsecond at a time, since a four microsecond delay is all it takes to corrupt the data stream.  

If you use DHT devices in an inhabited structure thermostat application, be aware that the DHT11 is notorious for frustratingly inconsistent manufactured quality affecting its operation, so have plenty of spares, make it easy to swap between them, and attempt to source from the best manufacturers you can find.  The thermostat application is exactly why I delved into the DHT11.  In the process of interfacing to my wall thermostat, I burnt it out and had to quickly (it was wintertime) use an Arduino on my bench as my thermostat.  I learned that I needed to ensure at least three consecutive readings were consistent and that a reasonable temperature margin was allowed (1 C is fine) or the furnace would get cycled recklessly.  I can now read and control my thermostat to my heart's content via Internet.
