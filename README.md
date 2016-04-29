# Setup Raspberry Pi Without Monitor or Keyboard from a Windows Machine
This is my step by step guidance to set up a Raspberry Pi.

## Install Raspbian

* Download Raspbian image from [here](http://www.raspberrypi.org/downloads/).
* Download and install [SD Formatter 4.0](https://www.sdcard.org/downloads/formatter_4/). Then use it to format SD card.
* Download and install [Win32DiskImager](http://sourceforge.net/projects/win32diskimager/). Then use it to write the Raspbian image into SD card.  
* Insert the SD card into Raspberry Pi, plug in the power cord and network cable, and then power on the Pi.

## Connect to Raspberry Pi through SSH

To setup the Raspberry Pi without a monitor or keyboard, you need to connect to the Pi through SSH.
To do that, you need to know the IP address of the Pi. You can find out the IP address from
your router's administrator page (usually, it can be accessed by browsing http://192.0.0.1 from any
machine inside your home network). If you cannot find out the IP address from the router's administrator page,
you can use [Advanced IP Scanner](http://www.advanced-ip-scanner.com/) to get the IP address. 


After you get the IP address, you can use [PuTTY](http://www.putty.org/) to connect to your Raspberry Pi through SSH.
If the SSH connection is successful, you will be greeted with the login prompt of your Raspberry Pi.
Since it is your first login, you just need to input `pi` as user name and `raspberry` as password.

## Raspi-config

After you log into the Pi, type the following command to open the Raspberry Pi configuration UI.
```
sudo raspi-config
```
Make the following configurations:

* Expand file system to ensure that all of the SD card storage is available to the OS.
* Choose Boot Options as `B1 Console`.
* Update your locale settings.
* Set the Memory Split (Advanced > Memory Split) to `16` since we won't be running a desktop. 
* Set your Hostname (Advanced > Hostname) if you don't like the default hostname `pi`.

Commit the changes and reboot your Pi with
```
sudo reboot
```

## Create a new user
## Setup WiFi through command line
## Mount USB Drive
## Share USB Drive
## Secure Raspberry Pi
## License

&copy; 2016 SanMuHe

This repository is licensed under the MIT license. See `LICENSE` for details.