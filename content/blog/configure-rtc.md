+++
title = "Configuration of DS1374 RTC on Raspberry Pi CM4"
date = "2025-04-19"
draft = false

[taxonomies]
tags = ["raspberry pi", "linux",]

[extra]
lang = "en"
+++

# What is an RTC?
A real-time clock (RTC) is used to keep accurate track of the current time and date, even when the main system power is turned off or the device is in a low-power state. A Raspberry Pi unlike many computers does not include a built in RTC. It relies on network time synchronization (NTP) to set the correct time after boot. This means when disconnected from the internet the Pi will not be able to maintain accurate time after a reboot or power cycle. 

We can add this functionality by interfacing an RTC to the Pi. In this example we're using the DS1374 RTC from Maxim.

# Setting up the RTC
The DS1374 connects using I2C. Typically, this would involve the following connections:
- VCC to 3.3V (Pin 1)
- GND to Ground (Pin 6)
- SDA to SDA1 (Pin 3, GPIO 2)
- SCL to SCL1 (Pin 5, GPIO 3)

Enable the I2C interface on the CM4 by editing `/boot/firmware/config.txt`:
```text
dtparam=i2c_arm=on
```
Reboot.

Next install the I2C tools using:
```bash
$ sudo apt-get update
$ sudo apt-get install i2c-tools
```
Check for the RTC
```bash
$ sudo i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- UU -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --
```
You should see an entry at 0x68 (the typical address for DS1374/DS1307/DS3231). If not, reboot the system.

Configure the device tree overlay by editing /boot/firmware/config.txt:
```text
dtoverlay=i2c-rtc,ds1374
```

The device tree overlay file should be located at /boot/firmware/overlays. If not, edit it there:
```bash
$ cat /boot/firmware/overlays/i2c-rtc-ds1374.dts
/dts-v1/;
/plugin/;

/{
        compatible = "brcm,bcm2835";
        fragment@0 {
                target = <&i2cbus>;
                __dormant__ {
                        #address-cells = <1>;
                        #size-cells = <0>;

                        ds1374: ds1374@68 {
                                compatible = "maxim,ds1374";
                                reg = <0x68>;
                        };
                };
        };

        frag100: fragment@100 {
                target = <&i2c_arm>;
                i2cbus: __overlay__ {
                        status = "okay";
                };
        };

        __overrides__ {
                ds1374 = <0>, "+0";

                i2c0 = <&frag100>, "target:0=",<&i2c0>;
                i2c_csi_dsi = <&frag100>, "target:0=",<&i2c_csi_dsi>;

                addr = <&ds1374>, "reg:0";
                wakeup-source = <&ds1374>, "wakeup-source?";
        };
};
```

# Remove Fake Hardware Clock

The Raspberry Pi OS uses a "fake" hardware clock by default. Remove it to avoid conflicts:
```bash
$ sudo apt remove fake-hwclock
$ sudo update-rc.d -f fake-hwclock remove
$ sudo systemctl disable fake-hwclock
```

# Synchronize and Set System Time
Set the current system time to the RTC:
```bash
$ sudo hwclock -w
```

On subsequent boots, the system should read the time from the RTC automatically. You can check the RTC time with:
```bash
$ sudo hwclock -r
```