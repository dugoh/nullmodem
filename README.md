This should implement a virtual nullmodem driver for linux as a kernel module. Maybe it does, maybe it doesn't. It has not been tested yet.

See http://stackoverflow.com/questions/52187/virtual-serial-port-for-linux
for its origins.

Introduction:

When the module is loaded, pairs of virtual COM ports are created that are connected to each other.
The name of the devices are /dev/nmpX, where X is the number of the COM port. "nmp" stands for "null modem port".

Two consecutive devices are linked with each other:

/dev/nmp0 <-> /dev/nmp1  
/dev/nmp2 <-> /dev/nmp3  


Features:
- line speed emulation

  Unlike pseudo terminals, this driver emulates serial line speed by using a timer.
  So if you set the line speed to 9600 baud, the througput will be about 960 cps.
  The driver takes mismatching configuration of the two ends of the virtual line into account.
  So if one end is configured for a different baud rate than the other end, each end will not
  receive any data from the other end.
  Likewise, if startbits/stopbits/databits don't match, no data will be received.
  Charsizes less than 8 are handled by clearing the appropriate upper bits.

- emulation of control lines
  - Supports TIOCMGET/TIOCMSET.
    This allows setting and reading the control lines (RTS/DTR etc.) via ioctl().
    When changed on one end of the virtual line, it affects the other end.
  - Supports TIOCMIWAIT.
    With this ioctl a user mode program can perform a blocking wait for control line changes.


Known problems / limitations:
- Flow control via XON/XOFF is not implemented.
- a 4k buffer is allocated for each virtual COM port (when opened). This may be a bit large.
- The timer that is used to emulate the line speed fires 20 times per second, even when no port is open
  (although it does not do much apart from checking every port whether it is open).
  This may put an unnecessary load on the system.


Installation:

Just unpack the tarball somewhere and run make in the nullmodem directory.
You will need to have "linux-headers" (>3.7) installed to compile the module.
A small shell script, called "reload", (re-)loads the module and sets permissions
of the /dev/nmp* devices.

Tweaking:

There are a few defines that can be tweaked:
- NULLMODEM_PAIRS: This defines how many virtual com port pairs will be created when the module is loaded.
  Currently, there will be 2 pairs created (4 devices).
- TIMER_INTERVAL: This determines the interval at which the timer fires.
  If you slow the timer down, more data will need to be transfered at each timer tick.
  Make sure the buffers are large enough. The timer rate and the highest baud rate determine the needed buffer size.
  Currently, the timer is set to 20Hz, i.e. every 50 milliseconds.
- TX_BUF_SIZE: This determines the size of the send buffers for the serial ports.
  These get filled by writes to the port and are drained by the timer.
  The timer also uses a buffer of this size to transfer data from one end of the virtual line to the other.
- NULLMODEM_MAJOR: this is the major device number. This is from the experimental range.
  Maybe someday it will be an officially assigned number.

In the makefile, you can turn on debug mode by un-commenting the "DEBUG = y" line.
The driver then prints diagnostic messages to the kernel log. You can watch this with a "tail -f /var/log/kern.log".
Some messages that produce a high volume of logging output are commented out in the code.
You can uncomment them if you want.

Notes/disclaimer:

- This module seems broken
- The code does not conform to any linux kernel coding standard.

If anyone of the linux kernel staff wants to include a nullmodem in the kernel, please do. FreeBSD has one since 4.4.
