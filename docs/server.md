# RaspberryPi5 Time Server Setup

These instructions setup the Pi5 as an NTP and PTP time server with the GPS/PPS as the refclock
source.

Once you have completed these instructions, you can move on to configuring the client to make sure
that it is working.

## BerryGPS-IMU

The BettryGPS-IMU needs headers soldered onto it before you can use it with the Pi. There are some
instructions [here](https://ozzmaker.com/berrygps-berrygps-imu-quick-start-guide/) on how to do that.

Solder the 5x2 female header that comes with the kit as per the instructions.

Additional to the BerryGPS instructions, you also need a way to get the PPS signal from the board to the GPIO 
headers. I've done this by adding a 9x1 male header to the board and using a jumper cable to attach the PPS pin 
to GPIO18/pin 12. This can be seen in the photo in the top-level readme file.

All the instructions that follow assume you have done the above.

Note that the PPS connection is essential; without it you won't get a useful time signal from your GPS.

Make sure your GPS antenna is placed somewhere that has a clear view of the sky so it can receive satellite
signals.

## OS Installation

Use the [RaspberryPi Imager](https://www.raspberrypi.com/software/) to install the OS on the your SD card.
Follow the [getting started](https://www.raspberrypi.com/documentation/computers/getting-started.html) instructions
to install and configure your operating system.

I've used the OS Lite (64 bit) option as shown in the image below. Others should work as well, but might
be some slight differences.

![OS Selection](os-choice.png "OS Selection")

Follow the instructions to the point where your Pi is on your network (wired connection) and you can log in 
remotely (or via the console).

## OS Configuration

There are two main configuration settings to change here:

* enable UART0
* configure PPS on GPIO18

UART0 is the serial port connection to the BerryGPS-IMU HAT. This needs to be enabled to receive GPS data
from the board. There is a lot of discussion around about UARTs and the RaspberryPi. Older versions of the
Pi only had one hardware UART which was used by the console by default so you needed to do a bit of shuffling
to get things in the right in place. The Pi5 has a lot of UARTs though and a dedicated one for the console so
it's a lot simpler.

Edit the `/boot/config.txt` file and add these two lines at the end:

    dtparam=uart0=on
    dtoverlay=pps-gpio,gpiopin=18

The first line enables UART0 for serial communication with the GPS, and the second installs the PPS overlay
and configures it to get the PPS signal from GPIO18.

Reboot and log back in.

First, to test the GPS is communicating over the serial port, run the following command and you should see
output similar to below:

    root@pi-gps:/home/pi# cat /dev/ttyAMA0

    ?ű?$GNTXT,01,01,01,More than 100 frame errors, UART RX was disabled*70
    
    $GNTXT,01,01,01,NMEA unknown msg*46
    
    $GNRMC,225528.00,A,3815.67084,S,14431.42040,E,2.201,,180124,,,A*75
    
    $GNVTG,,T,,M,2.201,N,4.076,K,A*39

To test the PPS signal, first install the `pps-tools` package:

    sudo apt install pps-tools

To test, run the following:

    root@pi-gps:/home/pi# ppstest /dev/pps0

    trying PPS source "/dev/pps0"
    found PPS source "/dev/pps0"
    ok, found 1 source(s), now start fetching data...
    source 0 - assert 1705618412.998999387, sequence: 88 - clear  0.000000000, sequence: 0
    source 0 - assert 1705618413.999075604, sequence: 89 - clear  0.000000000, sequence: 0
    source 0 - assert 1705618414.999147712, sequence: 90 - clear  0.000000000, sequence: 0

## GPSd

[GPSd](https://gpsd.io/) is a daemon that can communicate with a lot of different types of GPSes to deal
with their quirks and present a standard interface to applications. It is used here to read the
GPS NMEA stream and the PPS signal and converts them into useful information for `chronyd` to use as
a reference clock.

Install the software:

    apt install gpsd gpsd-clients

Configure the daemon by editing the `/etc/default/gpsd` file and setting the lines below:

    DEVICES="/dev/ttyAMA0 /dev/pps0"
    GPSD_OPTIONS="-n"
    USBAUTO="false"

Start the daemon:

    systemctl enable gpsd
    systemctl start gpsd

Run a test:

    cgps

You should see an output similar to below:

    ┌─}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}}┐┌─aaaaaaaaaaaaaaaaaSeen 24/Used  7┐
    │ Time:        2024-01-20T00:44:24.000Z (18)││GNSS   PRN  Elev   Azim   SNR Use│
    │ Latitude:         XX.XXXXXXXX S           ││GP 10   10  28.0  317.0  27.0  Y │
    │ Longitude:       YYY.YYYYYYYY E           ││GP 15   15  40.0   92.0  30.0  Y │
    │ Alt (HAE, MSL):     31.184,     34.962 m  ││GP 23   23  67.0  317.0  26.0  Y │
    │ Speed:             0.15 km/h              ││GP 29   29  48.0   50.0  17.0  Y │
    │ Track (true, var):    30.2,  11.6     deg ││GL  7   71  24.0    2.0  36.0  Y │
    │ Climb:           -18.00 m/min             ││GL 11   75  11.0  304.0  22.0  Y │
    │ Status:         3D FIX (16 secs)          ││QZ  2  194  29.0    n/a  34.0  Y │
    │ Long Err  (XDOP, EPX):  0.89, +/- 13.3 m  ││GP 13   13  20.0  122.0  24.0  N │
    │ Lat Err   (YDOP, EPY):  1.76, +/- 26.3 m  ││GP 16   16  28.0  240.0   0.0  N │
    │ Alt Err   (VDOP, EPV):  2.18, +/- 27.0 m  ││GP 18   18  67.0  178.0  14.0  N │
    │ 2D Err    (HDOP, CEP):  1.73, +/- 40.2 m  ││GP 26   26  31.0  275.0   0.0  N │
    │ 3D Err    (PDOP, SEP):  2.78, +/- 53.2 m  ││GP 27   27  12.0  225.0   0.0 uN │
    │ Time Err  (TDOP):       1.74              ││SB128   41  14.0  289.0   0.0  N │
    │ Geo Err   (GDOP):       3.28              ││SB129   42  45.0  353.0   0.0  N │
    │ ECEF X, VX:   -4083638.850 m    0.020 m/s ││SB137   50  46.0    1.0   0.0  N │
    │ ECEF Y, VY:    2910293.490 m    0.040 m/s ││GL  5   69  26.0  146.0   0.0  N │
    │ ECEF Z, VZ:   -3928245.860 m   -0.020 m/s ││GL  6   70  62.0   57.0  23.0  N │
    │ Speed Err (EPS):       +/-  2.1 km/h      ││GL 12   76  20.0  247.0   0.0  N │
    │ Track Err (EPD):        n/a               ││GL 13   77   7.0  204.0   0.0  N │
    │ Time offset:            1.017777314 s     ││GL 20   84  24.0   92.0   0.0  N │
    │ Grid Square:            QF21gr27          ││GL 21   85  58.0  163.0   0.0  N │
    └─aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa┘└More...eeeeeeeeeeeeeeeeeeeeeeeeee┘


## Chrony

You can use a number of different time servers here; I use chrony because the setup was a bit smoother
than other options I tried. See the `gpsd` documentation for instructions for setting up different options.

Install chrony:

    apt install chrony

This also uninstalls `systemd-timesyncd`.

Detailed configuration instructions are available on the GPSd [website](https://gpsd.io/gpsd-time-service-howto.html#_feeding_chrony_from_gpsd).

Edit the file `/etc/chrony/chrony.conf`:

    # serve time to the network
    allow
    
    # refclock from gpsd
    refclock SOCK /run/chrony.ttyAMA0.sock refid GPS precision 1e-1 offset 0.9999 delay 0.2
    refclock SOCK /run/chrony.pps0.sock refid PPS precision 1e-7
    
    # sync the real-time clock
    rtcsync

This adds the refclocks and sets the daemon as an NTP time source for the network. The `rtcsync` directive might
already be in the configuration file.

NOTE: An important thing to keep in mind here is that PPS refclock above isn't a raw PPS signal. It is PPS corrected
time from the GPS. GPSd takes care of this so we don't need to deal with raw PPS signals here in chrony. Relying on
GPSd to deal with this also means we can get both the PPS corrected time and actual GPS data from the same GPS.

Restart `chrony` to get the updated configuration:

    systemctl restart chrony

Check that chrony is running with:

    root@pi-gps:/etc/chrony# chronyc sources
    MS Name/IP address         Stratum Poll Reach LastRx Last sample               
    ===============================================================================
    #? GPS                           0   4     0     -     +0ns[   +0ns] +/-    0ns
    #? PPS                           0   4     0     -     +0ns[   +0ns] +/-    0ns
    ^? 159-196-3-239.9fc403.mel>     2   6     1     1  +3233us[+3233us] +/-   25ms
    ^? 220-158-215-20.broadband>     2   6     1     1  -3425us[-3425us] +/-   91ms
    ^? time.blockbluemedia.com       4   6     1     1  -1335us[-1335us] +/-   58ms
    ^? time.cloudflare.com           3   6     1     1   -649us[ -649us] +/- 9111us

At the moment, there is no GPS or PPS time signal being received. This is because when `gpsd` was started, `chronyd`
wasn't running so the linux sockets couldn't be found. To fix this, restart `gpsd`:

    systemctl restart gpsd

Check chrony again, and after a minute or two, you will see that the PPS refclock has been selected as
the time source:

    root@pi-gps:/etc/chrony# chronyc sources
    MS Name/IP address         Stratum Poll Reach LastRx Last sample               
    ===============================================================================
    #x GPS                           0   4     7    12  -1000ms[ -998ms] +/-  200ms
    #* PPS                           0   4   377    22   -309ns[ -377ns] +/-  101ns
    ^- 159-196-3-239.9fc403.mel>     2   6    77    12  +1593us[+3400us] +/-   24ms
    ^- 220-158-215-20.broadband>     2   6    77    13  -2657us[ -848us] +/-   91ms
    ^- time.blockbluemedia.com       4   6    77    13  -1318us[ +490us] +/-   58ms
    ^- time.cloudflare.com           3   6    77    14  -1713us[ -171us] +/-   10ms

Remember that if you restart `chrony` for any reason, you will also need to restart `gpsd`.

## PTP Server

There isn't much documentation around about how to serve linux system time with PTP. It seems that the 
typical use case is syncing linux system time from a PTP source. I did find 
[this](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/deployment_guide/s0-serving_ntp_time_with_ptp)
which was enough to get started.

To make this work, we need to do three things:

* configure ptp4l to be a time transmitter (always)
* boost the priority so it is likely to win any elections
* push the system time to the NICs hardware clock

First, install the necessary OS packages:

    apt install linuxptp

The server `ptp4l` is used to control the hardware timestamping and participate in elections and so on. We configure
it to always behave like a time transmitter, and we boost it's priority so it will win elections:

Edit `/etc/linuxptp/ptp4l.conf` and change these values, leaving the rest as is:

    priority1               127
    masterOnly              1

Start the service:

    systemctl enable ptp4l@eth0
    systemctl start ptp4l@eth0

To synchronise the system clock to the PTP hardware clock, the server `phc2sys` is used. Despite the name,
it can also sync system time to the PHC but we need a custom service file to make that work.

Create a new systemd service file:

    cd /lib/systemd/system
    cp phc2sys\@.service sys2phc-eth0.service

Then edit `sys2phc-eth0.service` lines to match this:

    Requires=ptp4l@eth0.service
    After=ptp4l@eth0.service

    phc2sys -w -c eth0 -s CLOCK_REALTIME

You can add the '-q' flag to the `phc2sys` command to stop it sending a log message every second.

Start the service:

    systemctl daemon-reload
    systemctl enable sys2phc-eth0
    systemctl start sys2phc-eth0

This should now be setting the hardware clock on eth0 to match the system time. There should be a way to verify
this, but I'm not sure how.

