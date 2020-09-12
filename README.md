# Setup Raspberry Pi Without Monitor or Keyboard from a Windows Machine

## Table of Contents

- [Install Raspbian](#install-raspbain)
- [Connect to Raspberry Pi through SSH](#connect-to-raspberry-pi-through-ssh)
- [Update and Upgrade Raspberry Pi](#update-and-upgrade-raspberry-pi)
- [Configure Raspberry Pi](#configure-raspberry-pi)
- [Secure Raspberry Pi](#secure-raspberry-pi)
- [Mount USB Flash Drive](#mount-usb-flash-drive)
- [Share USB Flash Drive](#share-usb-flash-drive)
- [References](#references)
- [License](#license)

## Install Raspbian

- Install Raspbian with [Raspberry Pi Imager](https://www.raspberrypi.org/documentation/installation/installing-images/).
- [Set up wireless networking headless](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md)
- [Enable SSH](https://www.raspberrypi.org/documentation/remote-access/ssh/README.md) on your Pi by placing a file name `ssh` (without any extension) onto the boot partition of the SD card.
- Insert the SD card into Raspberry Pi, plug in the power cord and network cable, and then power on the Pi.

## Connect to Raspberry Pi through SSH

First, you need to know the IP address of the Pi. You can find out the IP address from your router's administrator page (usually, it can be accessed by browsing http://192.0.0.1 from any machine inside your home network).
If you cannot find out the IP address from the router's administrator page, you can use [Advanced IP Scanner](http://www.advanced-ip-scanner.com/) to get the IP address.

After you get the IP address, you can use [PuTTY](http://www.putty.org/) to connect to your Raspberry Pi through SSH.
If the SSH connection is successful, you will be greeted with the login prompt of your Raspberry Pi.
Since it is your first login, you just need to input `pi` as user name and `raspberry` as password.

## Update and Upgrade Raspberry Pi

It is better to update Raspbian after your first login to your Pi.

```bash
sudo apt update
sudo apt full-upgrade
```

Per [here](https://www.raspberrypi.org/documentation/raspbian/updating.md) suggested, that `full-upgrade` is used in preference to a simple `upgrade`, as it also picks up any dependency changes that may have been made.

If your SD card is running out of space, your can free up some space with `sudo apt-get clean`. It will delete the downloaded package files (.deb files) from `/var/cache/apt/archives`.

## Configure Raspberry Pi

After you log into the Pi, type the following command to open the Raspberry Pi configuration UI.

```bash
sudo raspi-config
```

Make the following configurations:

- Expand file system to ensure that all of the SD card storage is available to the OS.
- Choose Boot Options as `B1 Console`.
- Update your locale settings.
- Set the Memory Split (Advanced > Memory Split) to `16` since we won't be running a desktop.
- Set your Hostname (Advanced > Hostname) if you don't like the default hostname `pi`.

Commit the changes and reboot your Pi with

```bash
sudo reboot
```

## Secure Raspberry Pi

Follow [this](https://www.raspberrypi.org/documentation/configuration/security.md) article from raspberrypi.org to secure your Raspberry Pi.

## Mount USB Flash Drive

You first need to install ntfs-3g driver to support NTFS format disk, type the following in bash:

```bash
sudo apt-get install ntfs-3g
```

Then plug the USB Flash Disk into your Pi and type the following:

```bash
sudo fdisk -l
```

You should see something like below at the output of the command:

```bash
/dev/sda1       92448 125173759 125081312 59.7G  7 HPFS/NTFS/exFAT
```

The `/dev/sda1` corresponds to the USB Flash Disk you just plug in.
If you have more than one USB Flash Disk plugged in, you might see `/dev/sdb1` and etc.

Before we can mount the drives, we need to create a directory to mount the drive:

```bash
sudo mkdir /mnt/usb
```

Now it is time to mount the USB Flask Drive:

```bash
sudo mount /dev/sda1 /mnt/usb
```

We also need to configure your Pi to automatically mount the UBS Flash Drive after every reboot

```bash
sudo cp /etc/fstab /etc/fstab.backup
sudo nano /etc/fstab
```

Add the below line into the `fstab` file

```bash
/dev/sda1       /mnt/usb        ntfs-3g rw,default        0       0
```

Save the `fstab` file and reboot your Pi

```bash
sudo reboot
```

## Share USB Flash Drive

We use a tool called `Samba` to make the USB Flash Drive plug into the Pi be accessible from other
computers inside the same ethernet.

First is to install the tool:

```bash
sudo apt-get install samba samba-common-bin
```

Second is to configure Samba

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
sudo nano /etc/samba/smb.conf
```

Type the below section into the configuration file

```bash
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

```bash
sudo /etc/init.d/samba restart
```

At this point we need to add in a user that can access the Pi’s samba shares.
We’re going to make an account with the username *backups* and the password *backups4ever*.
You can make your username and password whatever you wish.
To do so type the following commands:

```bash
sudo useradd backups -m -G users
sudo passwd backups
```

You’ll be prompted to type in the password twice to confirm.
After confirming the password, it’s time to add “backups” as a legitimate Samba user.
Enter the following command:

```bash
sudo smbpasswd -a backups
```

Enter the password for the backup account when prompted. You can now access the USB Flash
Drive on your Pi from any machine in your network with the username *backups* and the password *backups4ever*.

## Appendix - Creating a firewall by manually configuring `iptables` rules

If you don't want to use `ufw` as suggested in [Secure Raspberry Pi](#secure-raspberry-pi), you can set up your firewall by manually configuring the `iptables` rules.

Check your Pi's default firewall rules by entering the following command:

```bash
sudo iptables -L
```

Examine the output.
If you haven't implemented any firewall rules yet, you should see an empty rule set, as shown below:

```bash
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

Create a file to hold your firewall rules:

```bash
sudo nano /etc/iptables.firewall.rules
```

Now it's time to create some firewall rules. Copy and paste the rules shown below in to the iptables.firewall.rules file you just created.

```bash
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

Edit the rules as necessary.
By default, the rules will allow traffic to the following services and ports: HTTP (80), HTTPS (443), SSH (22), Samba (137,138,139,445), and ping.
All other ports will be blocked.
Save the changes to the firewall rules file and exit.

Activate the firewall rules by entering the following command:

```bash
sudo iptables-restore < /etc/iptables.firewall.rules
```

Recheck your Pi's firewall rules by entering the following command:

```bash
sudo iptables -L
```

Examine the output. The new rule set should look like the one shown below:

```bash
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

```bash
sudo nano /etc/network/if-pre-up.d/firewall
```

Copy and paste the following lines in to the file you just created and then save and exit:

```bash
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.firewall.rules
```

Set the script's permissions by entering the following command:

```bash
sudo chmod +x /etc/network/if-pre-up.d/firewall
```

Now, your firewall rules are in place and protecting your Pi.
Remember, you'll need to edit the firewall rules later if you install other software or services.

## References

- [Setting up a Raspberry Pi headless](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md)
- [How to Setup a Raspberry Pi Without a Monitor or Keyboard](http://www.circuitbasics.com/raspberry-pi-basics-setup-without-monitor-keyboard-headless-mode/)
- [How to set up a secure Raspberry Pi web server, mail server and Owncloud installation](https://www.pestmeester.nl/index.html)
- [How to Turn a Raspberry Pi into a Low-Power Network Storage Device](http://www.howtogeek.com/139433/how-to-turn-a-raspberry-pi-into-a-low-power-network-storage-device/)
- [Setting WiFi up via the command line](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)
- [Use Public Key Authentication with SSH Guide](https://www.linode.com/docs/security/use-public-key-authentication-with-ssh#windows-operating-system)

## License

&copy; 2016 SanMuHe

This repository is licensed under the MIT license. See `LICENSE` for details.
