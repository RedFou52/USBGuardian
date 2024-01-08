# USBGuardian

:warning: Hi! The project is on hold from my side. Feel free to fork it and improve it if you wish. I am available for any questions (although it's been several years since I last looked at the code!)

## Major updates :mage:

- Works with Raspberry Pi 4.
- Use of ClamAV daemon for quicker scans.
- User **pi** is now **securite**.
- Default scan directory is now `/media/securite`.
- USB is mounted with Raspbian and no longuer with USBGuardian.
- Edit udev rules to work with Raspbian Buster
- NTFS is supported.
- Change of formating method `formatUSB.py`.


# How it works ? :sassy_man: 
You can read the offical documentation here :[Documentation](https://usbguardian.wordpress.com/documentation/)

Look at the user manual : [usbguardian-user-manual.pdf](https://usbguardian.files.wordpress.com/2018/02/usbguardian-user-manual.pdf)

# Prerequisite  :mag_right:
- Raspberry Pi 4 (2GB or +)
- SDCard (16GB or +)
- img Raspbian Buster Desktop
- Keyboard / Mouse
- Internet connection

# Install - Raspberry :strawberry:

For the moment, no installation script is available; this will surely be my next mission! In the installation section, I'll discuss Raspberry Pi configuration followed by the installation and configuration of USBGuardian.

*The Flash part of the SD card will not be explained here*.

## ----- Update
```bash
pi> sudo apt update
pi> sudo apt upgrade
pi> sudo raspi-config
```
Use **raspi-config** and then :
- Advanced Options --> A1 Expand Filesystem
This option allows you to check if Raspbian is using 100% of the available space.

## ----- Root passwd
```bash
pi $> sudo -i
root> passwd
```
## ----- SWAP
Let's add 2GB of SWAP
```bash
root>apt install vim
root> dphys-swapfile swapoff
root> vim /etc/dphys-swapfile
	Ø Edit : CONF_SWAPSIZE=2000
root> dphys-swapfile setup
root> dphys-swapfile swapon
```
## ----- New user
**securite** will replace the **pi** account.
```bash
root > useradd -m securite
root > passwd securite 
root > vim /etc/sudoers
	Ø Edit : securite ALL=(ALL:ALL) ALL
root > su securite
securite $> sudo usermod -a -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users,input,netdev,gpio,i2c,spi securite
securite $> groups securite
```
## ----- Securiring pi account
In this part we will delete the pi account. If the deletion doesn't work (Pi user is using a process),
Push the deletion back to the next reboot of the machine.

```bash
securite $> sudo pkill -u pi
securite $> sudo deluser -remove-home pi
securite $> sudo vim /etc/sudoers.d/010_pi-nopasswd
	Ø Edit : pi --> securite
```
## ----- SSH Configuration
Allows a secure SSH connection using public/private keys.
If you do not have keys available for an SSH connection, change the PasswordAuthentication no option to yes.

```bash
securite $> mkdir ~/.ssh
securite $> ssh-keygen -t ed25519
securite $> ssh-keygen -t rsa
securite $> vim ~/.ssh/autorized_keys 
	Ø Edit : Add your public keys in the file
securite $> sudo vim /etc/ssh/sshd_config
	Ø Edit:
		○ Port 2222 # Change default port to 2222
		○ PasswordAuthentication no # Disable password connection, only keys are allowed
		○ PermitRootLogin no # Remove root connection with ssh.
		○ Banner /etc/issue.net # Add a beautiful banner before auth.
securite $> sudo service ssh restart
```

## ----- Hostname
Modification of the machine name, here the name will be station01. If station01 is already in use, just add +1.
```bash
securite $> sudo vim /etc/hostname
	Ø Edit : station01
securite $> sudo vim /etc/hosts
	Ø Edit : remove raspberry and add station01
```
## ----- (Optionnal : ZSH)
Install a zsh terminal with a plugin for a better CLI experience.
```bash
securite $> sudo apt install zsh
securite $> sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## ----- Reboot
```bash
securite $> sudo reboot
```

# Install & Config - USBGuardian :shield:

## ----- Install & Config ClamAV
```bash
securite $> sudo apt install clamav clamav-daemon
securite $> sudo systemctl enable clamav-daemon
securite $> ls -l /var/log/clamav
```
After installing clamav, there should be a `freshclam.log` file in `/var/log/clamav` .  
If the file does not exist, create it with the following commands:  
```bash
securite $> sudo touch /var/log/clamav/freshclam.log
securite $> chmod 600 /var/log/clamav/freshclam.log
securite $> chown clamav /var/log/clamav/freshclam.log
```
Run a first update of the virus database.  
```bash
securite $> sudo freshclam
```

If you encounter the following error: `ERROR : Problem with internal logger …`
Execute the commands in this order:  
```bash
securite $> sudo /etc/init.d/clamav-freshclam stop
securite $> sudo freshclam
securite $> sudo /etc/init.d/clamav-freshclam start
```

Modification of the clamav-daemon conf:  
```bash
securite $> sudo vim /etc/systemctl/system/clamav-daemon.service.d/extend.conf
	Ø Edit : ExecStartPre=/bin/mkdir -p /run/clamav
securite $> sudo systemctl daemon-reload
securite $> sudo service clamav-daemin start
```
Automatic update of the virus database:
```bash
securite $> sudo vim /etc/cron.daily/freshclam
	Ø Edit :
		#!/bin/sh
		/etc/init.d/clamav-freshclam stop
		/usr/bin/freshclam -v >> /var/log/clamav/freshclam.log
		/etc/init.d/clamav-freshclam start
```

## ----- Install USBGuardian 
```bash
securite $> cd 
securite $> git clone https://github.com/AlrikRr.USGBuardian.git
securite $> cd USBGuardian
securite $> sudo cp -r USBGuardian-core /opt/USBGuardian
securite $> sudo chmod +x -R /opt/USBGuardian/scripts
```

## ----- QT5
QQt5 will help you to compile the GUI. If you want to modify the GUI, I recommend you to install the `qtcreator` package:  
```bash
securite $> sudo apt install qt5-default qtcreator
securite $> cd USBGuardian-GUI
```
Then, you have to compile the application to have an executable binary:  
```bash
securite $> cd USBGuardian-GUI
securite $> qmake USBGUardian.pro
securite $> make
```

Then run the USBGuardian binary to run the GUI:  
```bash
securite $> cd USBGuardian-GUI
securite $> ./USBGuardian
```

**If the GUI is modified, you have to re-compile the application !**  

## ----- udev rules for USB
Add a UDEV rule that will detect if a USB key is plugged in.  
On detection of a "block" USB key the udev rule insertUSB.rules will execute the insertUSB.service.  
```bash
securite $> sudo cp ~/USBGuardian/udev/insertUSB.rules /etc/udev/rules.d/insertUSB.rules
```
Reload the udev rules with the command :  
```bash
securite $> sudo udevadm control --reload
```

## ----- insertUSB.service
Creation of the service that will perform our analysis.  
```bash
securite $> sudo cp ~/USBGuardian/service/insertUSB.service /etc/systemd/system/insertUSB.service
securite $> sudo systemctl enable insertUSB.service
```
## ----- Automount

Go to the Raspbian file explorer:  
	- Edit --> Preferences --> Removable media management --> DUnckeck "Display options..."  
	
## ----- Permissions

Make sure that your securite user and clamav user have the same rights:  
**/media/securite/**:  
```bash
securite $> sudo chown clamav:clamav -R /media/securite
securite $> sudo chmod 760 -R /media/securite
```

**/opt/USBGuardian/logs** :  
```bash
securite $> sudo chown securite -R /opt/USBGuardian/logs
securite $> sudo chmod 760 -R /opt/USBGuardian/logs
```

# To-do :mechanical_arm:

- [x] fix permission errors `/logs` & `/media/securite`  
- [ ] Install script  
- [x] Fix **formatUSB.py**  
- [x] Problem of duplicates on the display of found viruses  
- [ ] Use Subprocessing everywhere (python)
- [ ] On boot, execute `systemctl daemon-reload` for clamav-daemon.
