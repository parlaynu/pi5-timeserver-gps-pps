# Customising BerryGPS

The utility [ubxtool](https://gpsd.io/ubxtool-examples.html) from [gpsd](https://gpsd.io/index.html)
can be used to customise the GPS configuration.

This document shows some commands that work for the BerryGPS receiver chip - u-blox M8C.

## Reference Material

Ubxtool examples:
* https://gpsd.io/ubxtool-examples.html

BerryGPS-IMU
* https://ozzmaker.com/berrygps-berrygps-imu-quick-start-guide/

uBLOX M8
* https://ozzmaker.com/wp-content/uploads/2016/08/CAM-M8-FW3_DataSheet_UBX-15031574.pdf
* https://ozzmaker.com/wp-content/uploads/2019/09/u-blox8-M8_ReceiverDescrProtSpec_UBX-13003221_Public.pdf


## Checking Configurations

### [Check Protocol Version](https://gpsd.io/ubxtool-examples.html#_protocol_version)

    ubxtool -p MON-VER localhost:2947:/dev/ttyAMA0

    UBX-MON-VER:
      swVersion ROM CORE 3.01 (107888)
      hwVersion 00080000
      extension FWVER=SPG 3.01
      extension PROTVER=18.00
      extension GPS;GLO;GAL;BDS
      extension SBAS;IMES;QZSS

Set protocol variable:

    export UBXOPTS="-P 18 -v 2"

### [Reset](https://gpsd.io/ubxtool-examples.html#_default_configuration)

This will clear all settings back to the initial setup

    ubxtool -p RESET localhost:2947:/dev/ttyAMA0

Use the binary protocol instead of NMEA:

    ubxtool -e BINARY localhost:2947:/dev/ttyAMA0
    ubxtool -d NMEA localhost:2947:/dev/ttyAMA0

### Check Constellations

    ubxtool -p MON-GNSS localhost:2947:/dev/ttyAMA0

    UBX-MON-GNSS:
       version 0 supported 0xf defaultGnss 0x3 enabled 0xb
       simultaneous 3 reserved1 0 0 0
         supported (GPS Glonass Beidou Galileo)
         defaultGnss (GPS Glonass)
         enabled (GPS Glonass Galileo)

### Check the Configuration

    ubxtool -p CFG-GNSS localhost:2947:/dev/ttyAMA0

    UBX-CFG-GNSS:
     msgVer 0  numTrkChHw 32 numTrkChUse 32 numConfigBlocks 7
      gnssId 0 TrkCh  8 maxTrCh 16 reserved 0 Flags x01010001
       GPS L1C/A enabled
      gnssId 1 TrkCh  1 maxTrCh  3 reserved 0 Flags x01010001
       SBAS L1C/A enabled
      gnssId 2 TrkCh  4 maxTrCh  8 reserved 0 Flags x01010001
       Galileo E1 enabled
      gnssId 3 TrkCh  8 maxTrCh 16 reserved 0 Flags x01010000
       BeiDou B1I 
      gnssId 4 TrkCh  0 maxTrCh  8 reserved 0 Flags x03010000
       IMES L1 
      gnssId 5 TrkCh  0 maxTrCh  3 reserved 0 Flags x05050001
       QZSS L1C/A enabled
      gnssId 6 TrkCh  8 maxTrCh 14 reserved 0 Flags x01010001
       GLONASS L1 enabled

## [Managing Constellations](https://gpsd.io/ubxtool-examples.html#_constellations)

### Enable and Disable Constellations

Enable Galileo:

    ubxtool -e Galileo localhost:2947:/dev/ttyAMA0

Disable/Enable GLONASS

    ubxtool -d GLONASS localhost:2947:/dev/ttyAMA0
    ubxtool -e GLONASS localhost:2947:/dev/ttyAMA0

Enable/Disable BEIDOU

    ubxtool -e BEIDOU,1 localhost:2947:/dev/ttyAMA0
    ubxtool -d BEIDOU localhost:2947:/dev/ttyAMA0

### Enable QZ SBAS

Enable QZSS L1SAIF:

    ubxtool -e GPS,5 localhost:2947:/dev/ttyAMA0

    ubxtool -p CFG-GNSS localhost:2947:/dev/ttyAMA0

    UBX-CFG-GNSS:
     msgVer 0  numTrkChHw 32 numTrkChUse 32 numConfigBlocks 7
      gnssId 0 TrkCh  8 maxTrCh 16 reserved 0 Flags x01010001
       GPS L1C/A enabled
      gnssId 1 TrkCh  1 maxTrCh  3 reserved 0 Flags x01010001
       SBAS L1C/A enabled
      gnssId 2 TrkCh  4 maxTrCh  8 reserved 0 Flags x01010000
       Galileo E1 
      gnssId 3 TrkCh  8 maxTrCh 16 reserved 0 Flags x01010000
       BeiDou B1I 
      gnssId 4 TrkCh  0 maxTrCh  8 reserved 0 Flags x03010000
       IMES L1 
      gnssId 5 TrkCh  0 maxTrCh  3 reserved 0 Flags x05050001
       QZSS L1C/A L1S enabled
      gnssId 6 TrkCh  8 maxTrCh 14 reserved 0 Flags x01010001
       GLONASS L1 enabled

Can’t find a way to disable this once it’s been enabled - needs to be reset.

## [Dynamic Platform Model](https://gpsd.io/ubxtool-examples.html#_dynamic_platform_model)

### Check Current

    ubxtool -p CFG-NAV5 localhost:2947:/dev/ttyAMA0

    UBX-CFG-NAV5:
     mask 0xffff dynModel 0 fixmode 3 fixedAlt 0 FixedAltVar 10000
     minElev 5 drLimit 0 pDop 250 tDop 250 pAcc 100 tAcc 350
     staticHoldThresh 0 dgpsTimeOut 60 cnoThreshNumSVs 0
     cnoThresh 0 res 0 staticHoldMaxDist 0 utcStandard 0
     reserved x0 0
       dynModel (Portable) fixMode (Auto 2D/3D) utcStandard (Default)
       mask (dyn minEl posFixMode drLim posMask timeMask staticHoldMask dgpsMask cnoThreshold utc)


### Change The Model

Stationary:

    ubxtool -p MODEL,2 localhost:2947:/dev/ttyAMA0

    ubxtool -p CFG-NAV5 localhost:2947:/dev/ttyAMA0

    UBX-CFG-NAV5:
     mask 0xffff dynModel 2 fixmode 3 fixedAlt 0 FixedAltVar 10000
     minElev 5 drLimit 0 pDop 250 tDop 250 pAcc 100 tAcc 350
     staticHoldThresh 0 dgpsTimeOut 60 cnoThreshNumSVs 0
     cnoThresh 0 res 0 staticHoldMaxDist 0 utcStandard 0
     reserved x0 0
       dynModel (Stationary) fixMode (Auto 2D/3D) utcStandard (Default)
       mask (dyn minEl posFixMode drLim posMask timeMask staticHoldMask dgpsMask cnoThreshold utc)


| Mode             | Code |
| ---------------- | -----|
| Portable         | 0    |
| Stationary       | 2    |
| Pedestrian       | 3    |
| Automotive       | 4    |
| Sea              | 5    |
| Airborne < 1g    | 6    |
| Airborne < 2g    | 7    |
| Airborne < 4g    | 8    |
| Wrist Worn Watch | 9    |


## [Measurement Rate](https://gpsd.io/ubxtool-examples.html#_rate_settings)

    ubxtool -p CFG-RATE localhost:2947:/dev/ttyAMA0

    UBX-CFG-RATE:
     measRate 1000 navRate 1 timeRef 1 (GPS)

## [Time Pulse](https://gpsd.io/ubxtool-examples.html#_timepulse)

Query current settings for TIMEPULSE pin:

    ubxtool -p CFG-TP5,0 localhost:2947:/dev/ttyAMA0

    UBX-CFG-TP5:
      tpIdx 0 version 1 reserved1 0
      antCableDelay 50 rfGroupDelay 0 freqPeriod 1000000 freqPeriodLock 1000000
      pulseLenRatio 0 pulseLenRatioLock 100000 userConfigDelay 0
      flags x77
        flags (Active lockGnssFreq lockedOtherSet isLength alignToTow RisingEdge)
        gridToGps (UTC) syncMode 0


Query current settings for TIMEPULSE2 pin:

    ubxtool -p CFG-TP5,1 localhost:2947:/dev/ttyAMA0

    UBX-CFG-TP5:
      tpIdx 1 version 1 reserved1 0
      antCableDelay 50 rfGroupDelay 0 freqPeriod 4 freqPeriodLock 1
      pulseLenRatio 125000 pulseLenRatioLock 100000 userConfigDelay 0
      flags x7e
        flags (lockGnssFreq lockedOtherSet isFreq isLength alignToTow RisingEdge)
        gridToGps (UTC) syncMode 0


