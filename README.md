This implements a virtual nullmodem driver for linux.

See http://stackoverflow.com/questions/52187/virtual-serial-port-for-linux for its origins.


Introduction:

Pairs of virtual serial ports are created that are connected to each other. The name of the devices are /dev/nmpX, where X is the number of the serial port. "nmp" stands for "null modem port".

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
- a 4k buffer is allocated for each virtual serial port (when opened). This may be a bit large.
- The timer that is used to emulate the line speed fires 20 times per second, even when no port is open
  (although it does not do much apart from checking every port whether it is open).
  This may put an unnecessary load on the system.


Installation:

Clone the repo and run make in the nullmodem directory.
You will need to have "linux-headers" (>3.7) installed to compile the module.
A small shell script, called "reload", (re-)loads the module and sets permissions
of the /dev/nmp* devices.

You can also add it as a driver in the kernel.

```sh
cd src/linux # or wherever your sources live
mkdir -p drivers/nullmodem/
get -nc -P drivers/nullmodem/ https://github.com/dugoh/nullmodem/raw/master/nullmodem.c
echo 'obj-$(CONFIG_NULLMODEM) += nullmodem.o' >drivers/nullmodem/Makefile
printf "config NULLMODEM\n\ttristate \"Virtual nullmodem driver\"\n\thelp\n\t  Creates pairs of virtual COM ports that are connected to each other.\n\t  The name of the devices are /dev/nmpX, where X is the number of the COM port.\n" >drivers/nullmodem/Kconfig
echo 'obj-$(CONFIG_NULLMODEM)\t\t+= nullmodem/' >> drivers/Makefile
sed -i~ -e's/endmenu/source "drivers\/nullmodem\/Kconfig"\n\nendmenu/' drivers/Kconfig
echo CONFIG_NULLMODEM=y >>.config
```

Tweaking:
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
- The code does not conform to any linux kernel coding standard.
- It has not been tested extensively.
- It does work on a stock Slackware64 14.2 box.

If anyone of the linux kernel staff wants to include a nullmodem in the kernel, please do. FreeBSD has one since 4.4.
