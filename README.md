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

We need a few setups to improve the security of your Pi.

### Key-pair authentication
By default, password authentication is used to login to your Pi via SSH. A more secure way
is to use key-pair authentication. Just follow instructions in the 
[Use Public Key Authentication with SSH Guide](https://www.linode.com/docs/security/use-public-key-authentication-with-ssh#windows-operating-system)
to set up PuTTY to use key-pair authentication to login to your Pi.

### Disabling SSH Password Authentication and Root Login
Now it's time to make some changes to the default SSH configuration. 
First, you'll disable password authentication to require all users connecting via SSH 
to use key authentication. Next, you'll disable root login to prevent the root user 
from logging in via SSH. These steps are optional, but are strongly recommended.

Open and edit the SSH configuration file:
```
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

Change the PasswordAuthentication setting to "no":
```
PasswordAuthentication no
```
Change the PermitRootLogin setting to no:
```
PermitRootLogin no
```
Save the changes to the SSH configuration file and exit.

Restart the SSH service to load the new configuration. Enter the following command:
```
sudo service ssh restart
```

### Creating a Firewall
You also need to set up a firewall to limit and block unwanted inbound traffic to your Pi.

Check your Pi's default firewall rules by entering the following command:
```
sudo iptables -L
```
Examine the output. If you haven't implemented any firewall rules yet, you should see an 
empty ruleset, as shown below:
```
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
Create a file to hold your firewall rules:
```
sudo nano /etc/iptables.firewall.rules
```
Now it's time to create some firewall rules. Copy and paste the rules shown below in to the iptables.firewall.rules file you just created.
```
*filter

# Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT -d 127.0.0.0/8 -j REJECT

# Accept all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow all outbound traffic - you can modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

# Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

#  Allow SSH connections
#  The -dport number should be the same port number you set in sshd_config
-A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT

# Allow Samba access
-A INPUT -s 192.168.1.0/24 -m state --state NEW -p tcp --dport 137 -j ACCEPT
-A INPUT -s 192.168.1.0/24 -m state --state NEW -p tcp --dport 138 -j ACCEPT
-A INPUT -s 192.168.1.0/24 -m state --state NEW -p tcp --dport 139 -j ACCEPT
-A INPUT -s 192.168.1.0/24 -m state --state NEW -p tcp --dport 445 -j ACCEPT

#  Allow ping
-A INPUT -p icmp --icmp-type echo-request -j ACCEPT

#  Log iptables denied calls
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

#  Drop all other inbound - default deny unless explicitly allowed policy
-A INPUT -j DROP
-A FORWARD -j DROP

COMMIT
```
Edit the rules as necessary. By default, the rules will allow traffic to the following services 
and ports: HTTP (80), HTTPS (443), SSH (22), Samba (137,138,139,445), and ping. All other ports will be blocked.
Save the changes to the firewall rules file and exit.

Activate the firewall rules by entering the following command:
```
sudo iptables-restore < /etc/iptables.firewall.rules
```
Recheck your Pi's firewall rules by entering the following command:
```
sudo iptables -L
```
Examine the output. The new rule set should look like the one shown below:
```
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
fail2ban-ssh  tcp  --  anywhere             anywhere             multiport dports ssh
ACCEPT     all  --  anywhere             anywhere
REJECT     all  --  anywhere             loopback/8           reject-with icmp-port-unreachable
ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https
ACCEPT     tcp  --  192.168.1.0/24       anywhere             state NEW tcp dpt:netbios-ns
ACCEPT     tcp  --  192.168.1.0/24       anywhere             state NEW tcp dpt:netbios-dgm
ACCEPT     tcp  --  192.168.1.0/24       anywhere             state NEW tcp dpt:netbios-ssn
ACCEPT     tcp  --  192.168.1.0/24       anywhere             state NEW tcp dpt:microsoft-ds
ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:ssh
ACCEPT     icmp --  anywhere             anywhere             icmp echo-request
LOG        all  --  anywhere             anywhere             limit: avg 5/min burst 5 LOG level debug prefix "iptables denied: "
DROP       all  --  anywhere             anywhere

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DROP       all  --  anywhere             anywhere

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere
```
Now you need to ensure that the firewall rules are activated every time you restart your Pi.

Start by creating a new script with the following command:
```
sudo nano /etc/network/if-pre-up.d/firewall
```
Copy and paste the following lines in to the file you just created and then save and exit:
```
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.firewall.rules
```

Set the script's permissions by entering the following command:
```
sudo chmod +x /etc/network/if-pre-up.d/firewall
```
Now, your firewall rules are in place and protecting your Pi. 
Remember, you'll need to edit the firewall rules later if you install other software or services.

### Installing and Configuring Fail2Ban
Fail2Ban is an application that prevents dictionary attacks on your server. 
When Fail2Ban detects multiple failed login attempts from the same IP address, 
it creates temporary firewall rules that block traffic from the attacker's IP address. 
Attempted logins can be monitored on a variety of protocols, including SSH, HTTP, and SMTP. 
By default, Fail2Ban monitors SSH only.

Type the following to install Fail2Ban on your Pi:
```
sudo apt-get install fail2ban
```
Fail2Ban is now installed and running on your Pi. 
It will monitor your log files for failed login attempts. 
After an IP address has exceeded the maximum number of authentication attempts, 
it will be blocked at the network level and the event will be logged in 
`/var/log/fail2ban.log`.

## References

* [How to Setup a Raspberry Pi Without a Monitor or Keyboard](http://www.circuitbasics.com/raspberry-pi-basics-setup-without-monitor-keyboard-headless-mode/)
* [How to set up a secure Raspberry Pi web server, mail server and Owncloud installation](https://www.pestmeester.nl/index.html)
* [How to Turn a Raspberry Pi into a Low-Power Network Storage Device](http://www.howtogeek.com/139433/how-to-turn-a-raspberry-pi-into-a-low-power-network-storage-device/)
* [Setting WiFi up via the command line](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)
* [Use Public Key Authentication with SSH Guide](https://www.linode.com/docs/security/use-public-key-authentication-with-ssh#windows-operating-system)

## License

&copy; 2016 SanMuHe

This repository is licensed under the MIT license. See `LICENSE` for details.