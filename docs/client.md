# RaspberryPi5 Time Client Setup

These instructions setup the Pi4 to synchronise its time with and NTP and/or PTP server.

## OS Installation

Use the [RaspberryPi Imager](https://www.raspberrypi.com/software/) to install the OS on the your SD card.
Follow the [getting started](https://www.raspberrypi.com/documentation/computers/getting-started.html) instructions
to install and configure your operating system.

I've used the OS Lite (64 bit) option as shown in the image below. Others should work as well, but might
be some slight differences.

![OS Selection](os-choice.png "OS Selection")

Follow the instructions to the point where your Pi is on your network (wired connection) and you can log in 
remotely (or via the console).

## NTP Client

Install `chrony` as ntp client:

    apt install chrony

Add this to `/etc/chrony/chrony.conf`, replacing the IP address with the address of the server you previously
created:

    server 192.168.1.37 iburst

Check the results:

    root@pi-sofa:/etc/chrony# chronyc sources

    MS Name/IP address         Stratum Poll Reach LastRx Last sample               
    ===============================================================================
    ^* 192.168.1.37                  1   6    17     5  +4367ns[+7959ns] +/-  115us
    ^- ntp3.ds.network               2   6    17     3    +15ms[  +15ms] +/-  177ms
    ^- pauseq4vntp2.datamossa.io     2   6    17     4  -2040us[-2040us] +/-   40ms
    ^- time.cloudflare.com           3   6    17     4  +1470us[+1470us] +/-   13ms
    ^- 159-196-3-239.9fc403.mel>     2   6    17     5   +242us[ +242us] +/-   23ms

This shows that server just added is the currently selected time source. It might take a minute or two to be 
recognised.

## PTP Client with Software Timestamping

The RaspberryPi 4B doesn't have hardware timestamping on the eth0 so the configuration below uses software
timestamping. Hardware timestamping should be just a simple change to the configuration as will be obvious
below.

This runs linuxptp in time receiver mode to synchronise from a time transmitter. The ptp daemons write the
time to `chrony` shared memory to use as a refclock source.

First edit `/etc/chrony/chrony.conf` configuration to make sure that the shared memory segment is present for
`ptp4l`:

    refclock SHM 2 refid PTP precision 1e-7 stratum 1

Restart and check the results:

    systemctl restart chrony

    root@pi-sofa:/etc/chrony# chronyc sources

    MS Name/IP address         Stratum Poll Reach LastRx Last sample               
    ===============================================================================
    #? PTP                           0   4     0     -     +0ns[   +0ns] +/-    0ns
    ^* 192.168.1.37                  1   6     7     1  +5195ns[  -12us] +/-  118us
    ^- ntp1.ds.network               3   6     7     2  -2340us[-2358us] +/-   18ms
    ^- 167-179-162-50.a7b3a2.bn>     1   6     7     1   +534us[ +517us] +/-   25ms
    ^- time.blockbluemedia.com       4   6     7     2  -2511us[-2528us] +/-   41ms
    ^- 220-158-215-20.broadband>     2   6     7     2  -3609us[-3626us] +/-   86ms

From the above, you can see that `chrony` knows about the PTP refclock but there's no data coming
just yet.

Now, install the `linuxptp` package:

    apt install linuxptp

And edit the configuration file `/etc/linuxptp/ptp4l.conf`, changing the settings below. Leave
all other settings alone.

    slaveOnly               1
    clock_servo             ntpshm
    ntpshm_segment          2
    time_stamping           software
    summary_interval        10

Start the server:

    systemctl enable ptp4l@eth0
    systemctl start ptp4l@eth0

And to check, run the below command. It will take a minute or two before the PTP refclock
is the selected source. As you can see, even with software timestamping, it's much more
accurate than NTP on the local network.

    root@pi-sofa:/etc/chrony# chronyc sources

    MS Name/IP address         Stratum Poll Reach LastRx Last sample               
    ===============================================================================
    #* PTP                           1   4    77    19    -15ns[ -102ns] +/-  563ns
    ^- 192.168.1.37                  1   6    37    35    -23us[  -23us] +/-  119us
    ^- x.ns.gin.ntt.net              2   6    37    35  -2403us[-2403us] +/-  111ms
    ^- ntp2.ds.network               2   6    37    34    +14ms[  +14ms] +/-  177ms
    ^- pauseq4vntp1.datamossa.io     3   6    37    34  -3222us[-3223us] +/-   94ms
    ^- time.cloudflare.com           3   6    37    36  -3812us[-3813us] +/-   12ms

You can check for the PTP network messages using tcpdump. First install the utility:

    apt install tcpdump

And look for packets from the server:

    tcpdump -n -i eth0 host 192.168.1.37

You should see PTP messages, some of which are like this:

    13:57:17.245636 IP 192.168.1.37.319 > 224.0.1.129.319: PTPv2, v1 compat : no, msg type : sync msg, length : 44, domain : 0, reserved1 : 0, Flags [two step], NS correction : 0, sub NS correction : 0, reserved2 : 0, clock identity : 0xd83addfffeac163e, port id : 1, seq id : 6541, control : 0 (Sync), log message interval : 0, originTimeStamp : 0 seconds, 0 nanoseconds
    13:57:17.245685 IP 192.168.1.37.320 > 224.0.1.129.320: PTPv2, v1 compat : no, msg type : follow up msg, length : 44, domain : 0, reserved1 : 0, Flags [none], NS correction : 0, sub NS correction : 0, reserved2 : 0, clock identity : 0xd83addfffeac163e, port id : 1, seq id : 6541, control : 2 (Follow_Up), log message interval : 0, preciseOriginTimeStamp : 1705719474 seconds, 245606966 nanoseconds

And, NTP messages that look like this:

    13:57:18.085311 IP 192.168.1.31.43538 > 192.168.1.37.123: NTPv4, Client, length 48
    13:57:18.085517 IP 192.168.1.37.123 > 192.168.1.31.43538: NTPv4, Server, length 48

