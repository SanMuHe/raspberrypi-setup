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

It is always a good idea to delete the default user name `pi` and create a new user name.
After log into your Pi with PuTTY, type
```
groups
```
You will see a list output similar to the one below (Note: yours may be different to mine, so
pay attention to your list)
```
pi adm dialout cdrom sudo audio video plugdev games users input netdev gpio i2c spi
```
Now you can create a new user by typing the following but remember to use your list of 
groups (minus the first 'pi' item) and replace USERNAME with the username you want to create.

```
sudo useradd -m -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users,input,netdev,gpio,i2c,spi USERNAME
```

Next set a password for the new user:

```
sudo passwd USERNAME
```

Complete the prompts as they appear and restart Pi
```
sudo reboot
```

## Setup WiFi via command line

Plug your WiFi dongle into one of the USB ports of the Raspberry Pi and wait several minutes for the driver to be installed.
Then type the following command to scan all available WiFi networks.
```
sudo iwlist wlan0 scan
```
Look out something like `ESSID:"testing"` and `IE: IEEE 802.11i/WPA2 Version 1`. 
The former one is the name of the WiFi network and the later one is the authentication used.

Open the `wpa-supplicant` configuration file:
```
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```
At the bottom of the file add the following:
```
network={
    ssid="The_ESSID_from_earlier"
    psk="Your_wifi_password"
}
```
Save the file by pressing **Ctrl+X** then **Y**, then finally press **Enter**.

At this point, `wpa-supplicant` will normally notice a change has occurred within a few seconds, 
and it will try and connect to the network. If it does not, either manually restart the interface 
with `sudo ifdown wlan0` and `sudo ifup wlan0`, or reboot your Raspberry Pi with `sudo reboot`. 

You can verify if it has successfully connected using `ifconfig wlan0`. 
If the `inet addr` field has an address beside it, the Pi has connected to the network. 
If not, check your password and ESSID are correct. 

## Mount USB Flash Drive

You first need to install ntfs-3g driver to support NTFS format disk, type the following in bash:
```
sudo apt-get install ntfs-3g
```
Then plug the USB Flash Disk into your Pi and type the following:
```
sudo fdisk -l
```
You should see something like below at the output of the command:
```
/dev/sda1       92448 125173759 125081312 59.7G  7 HPFS/NTFS/exFAT
```
The `/dev/sda1` corresponds to the USB Flash Disk you just plug in. 
If you have more than one USB Flash Disk plugged in, you might see `/dev/sdb1` and etc.

Before we can mount the drives, we need to create a directory to mount the drive:
```
sudo mkdir /mnt/usb
```
Now it is time to mount the USB Flask Drive:
```
sudo mount /dev/sda1 /mnt/usb
```
We also need to configure your Pi to automatically mount the UBS Flash Drive after every reboot
```
sudo cp /etc/fstab /etc/fstab.backup
sudo nano /etc/fstab
```
Add the below line into the `fstab` file
```
/dev/sda1       /mnt/usb        ntfs-3g rw,default        0       0
```
Save the `fstab` file and reboot your Pi
```
sudo reboot
```

## Share USB Flash Drive 

We use a tool called `Samba` to make the USB Flash Drive plug into the Pi be accessible from other
computers inside the same ethernet. 

First is to install the tool:
```
sudo apt-get install samba samba-common-bin
```

Second is to configure Samba
```
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
sudo nano /etc/samba/smb.conf
```
Type the below section into the configuration file
```
[pi-usb]
    comment = USB Flash Dirve at my Raspberry Pi
    path=/mnt/usb
    valid users=@users
    force group=users
    browseable=yes
    writeable=yes
    only guest=no
    create mask=0777
    directory mask=0777
    public=no
```
Save and close the file and restart the `Samba` daemons:
```
sudo /etc/init.d/samba restart
```
At this point we need to add in a user that can access the Pi’s samba shares. 
We’re going to make an account with the username *backups* and the password *backups4ever*. 
You can make your username and password whatever you wish. 
To do so type the following commands:
```
sudo useradd backups -m -G users
sudo passwd backups
```
You’ll be prompted to type in the password twice to confirm. 
After confirming the password, it’s time to add “backups” as a legitimate Samba user. 
Enter the following command:
```
sudo smbpasswd -a backups
```
Enter the password for the backup account when prompted. You can now access the USB Flash
Drive on your Pi from any machine in your network with the username *backups* and the password *backups4ever*.

## Secure Raspberry Pi
## References

* [How to Setup a Raspberry Pi Without a Monitor or Keyboard](http://www.circuitbasics.com/raspberry-pi-basics-setup-without-monitor-keyboard-headless-mode/)
* [How to set up a secure Raspberry Pi web server, mail server and Owncloud installation](https://www.pestmeester.nl/index.html)
* [How to Turn a Raspberry Pi into a Low-Power Network Storage Device](http://www.howtogeek.com/139433/how-to-turn-a-raspberry-pi-into-a-low-power-network-storage-device/)
* [Setting WiFi up via the command line](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)

## License

&copy; 2016 SanMuHe

This repository is licensed under the MIT license. See `LICENSE` for details.