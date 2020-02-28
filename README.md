# ArduPilot Blue - A beginner's guide
This is my regularly updated beginner's guide to setting up the BeagleBone Blue with Mirko Denecke's port of ArduPilot (https://github.com/mirkix/ardupilotblue). It is based in great part upon similar documents at Patrick Poirier's PocketPilot project (https://github.com/PocketPilot/PocketPilot). Many thanks to both these fine individuals, and others too @https://gitter.im/mirkix/BBBMINI, for their superb work!

Only necessary steps are shown, excepting that I install Git for the sake of convenience. Information found elsewhere may indicate steps that are no longer necessary due to software updates, or steps that only apply to other platforms (BBBMINI or PocketPilot, etc).

## Part 1 - Preparation
0) Before I begin, I want to stress that supplying the BeagleBone Blue with adequate power is a must. Typically, we'll be attaching quite a few navigation-related peripherals to it, and they'll not behave correctly without enough juice.

1) Go to https://rcn-ee.net/rootfs/bb.org/testing/ and select the directory named with the latest date. Then click on the stretch-console subdirectory. You'll see a number of files here. Download the file named something like 'bone-debian-V.V-console-armhf-20YY-MM-DD-1gb.img.xz'. This is what's known as the 'console' image. It's a very minimal distribution of Debian with only the bare essentials. Other images are available, including the 'IoT' image (IoT = Internet of Things), that include additional software and can make for a more comfortable experience should you be very new to Linux. They're available from the same site, but should NOT be used with this guide unless you understand the implications of doing so.

    Don't assume that the latest image is the best, or even that it works. Afterall, this is the 'testing' repository, so be warned. However, I will endeavour to keep abreast of what seems to be the latest functional image and mention it here.

    I'm currently using: https://rcn-ee.net/rootfs/bb.org/testing/2019-12-01/stretch-console/bone-debian-9.11-console-armhf-2019-12-01-1gb.img.xz
    
    *Please try using this precise image first before raising issues!*
    
    And quickly on the subject of editing text files in Linux: naturally, you can use your favourite text editor. Personally, I like nano, which, owing to the way these Debian images have been configured, is invoked by default if you use the `sudoedit` command.
    
2) Next you'll need to flash the image to a microSD card. Whether you are using Linux or Windows, I highly recommend a program called Etcher for this task (https://etcher.io/).

3) It should now be possible to boot up the BeagleBone Blue from the microSD card. It's beyond the scope of this document to detail all the ways of interacting with the BBBlue, but often it's accomplished by plugging in a Micro-USB cable and either using SSH (to 'debian@192.168.7.2', password 'temppwd') or establishing a serial link over a COM port (user 'debian', password 'temppwd)' in a program like Minicom or PuTTY. More information can be found here: https://beagleboard.org/blue

    BeagleBone drivers come with Windows 10, but Linux distributions can be hit-and-miss. If you're experiencing problems, try making a udev rule:

       sudo -s
       cat >/etc/udev/rules.d/73-beaglebone.rules <<EOF
       ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_interface", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="a6d0", DRIVER=="", RUN+="/sbin/modprobe -b ftdi_sio"
       ACTION=="add", SUBSYSTEM=="drivers", ENV{DEVPATH}=="/bus/usb-serial/drivers/ftdi_sio", ATTR{new_id}="0403 a6d0"
       ACTION=="add", KERNEL=="ttyUSB*", ATTRS{interface}=="BeagleBone", ATTRS{bInterfaceNumber}=="00", SYMLINK+="beaglebone-jtag"
       ACTION=="add", KERNEL=="ttyUSB*", ATTRS{interface}=="BeagleBone", ATTRS{bInterfaceNumber}=="01", SYMLINK+="beaglebone-serial"
       EOF
       udevadm control --reload-rules
       exit
4) Hopefully, you now find yourself logged in to the BBBlue and at the command prompt. We'll start by allowing the debian user to sudo without having to enter the password every (subsequent) time:

       echo "debian ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers.d/debian >/dev/null
    The next job is to update and install some software using an available internet connection, so it's time to set up connman for WiFi access. I do it the following way because it's easier to automate in a script later on. First, make a note of your router's SSID and WiFi password. Then type the following:

        sudo -s
        connmanctl services | grep '<your SSID>' | grep -Po 'wifi_[^ ]+'
    The response will be a hash that'll look something like 'wifi_38d279e099a8_4254487562142d4355434b_managed_psk'. If you see nothing, try it again - you probably made a typo.
    
    Now, using this hash, we're going to enter a file directly from the keyboard (stdin) using `cat`, one line at a time:
    
        cat >/var/lib/connman/wifi.config
        [service_<your hash>]
        Type = wifi
        Security = wpa2
        Name = <your SSID>
        Passphrase = <your WiFi password>
    Press 'Ctrl-D' to quit out to the prompt, and then type: `exit`
    
    A prominent green LED will have come on, signifying that WiFi is up. The BBBlue is connected to your router, and its IP address on your WiFi network can be found using:
    
       ip addr show wlan0
    If, for whatever reason, you find yourself unable to query the BBBlue directly, other ways to find its IP address are programs like nmap (`sudo nmap 192.168.0.0/24`) or by logging in to your router and looking there.
    
    Now try SSHing to the BBBlue using its WiFi IP address. Remember that 192.168.7.2 will still work as well.
    
    If you can't get WiFi to work with connman, or you simply don't want to use connman, you can use the following method. First, type: `sudo systemctl disable connman`. Then, with your SSID and WiFi password handy, edit /etc/network/interfaces to read:

       # The loopback network interface.
       auto lo
       iface lo inet loopback

       # WiFi w/ onboard device (dynamic IP).
       auto wlan0
       iface wlan0 inet dhcp
       wpa-ssid "<your SSID>"
       wpa-psk "<your WiFi password>"
       dns-nameservers 8.8.8.8 1.1.1.1
       
       # Ethernet/RNDIS gadget (g_ether).
       # Used by: /opt/scripts/boot/autoconfigure_usb0.sh
       iface usb0 inet static
       address 192.168.7.2
       netmask 255.255.255.252
       network 192.168.7.0
       gateway 192.168.7.1
    Now, reboot the BBBlue with: `sudo reboot`
    
    After logging back in, type: `sudo ifup wlan0`. The green LED should come on. You're connected.
    
    If you want the BBBlue to have a static IP (let's say 192.168.0.99), alter the '`# WiFi w/ onboard device (dynamic IP).`' section of /etc/network/interfaces to read:
    
       # WiFi w/ onboard device (static IP).
       auto wlan0
       iface wlan0 inet static
       wpa-ssid "<your SSID>"
       wpa-psk "<your WiFi password>"
       address 192.168.0.99  # <--- The desired static IP address of the BBBlue.
       netmask 255.255.255.0
       gateway 192.168.0.1  # <--- The address of your router.
       dns-nameservers 8.8.8.8 1.1.1.1
    If WiFi just isn't an option, you can tell the BBBlue that it'll be sharing your computer's internet connection by typing (at the BBBlue's command prompt):
    
       sudo /sbin/route add default gw 192.168.7.1
       echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" | sudo tee -a /etc/resolv.conf >/dev/null
    Incidentally, these changes can be made permanent by altering the '`# Ethernet/RNDIS gadget (g_ether).`' section of /etc/network/interfaces to:
    
       # Ethernet/RNDIS gadget (g_ether).
       # Used by: /opt/scripts/boot/autoconfigure_usb0.sh
       iface usb0 inet static
       address 192.168.7.2
       netmask 255.255.255.252
       network 192.168.7.0
       gateway 192.168.7.1
       post-up route add default gw 192.168.7.1
       dns-nameservers 8.8.8.8 1.1.1.1
   You can then tell your Linux computer to share with the BBBlue by typing (at the computer's command prompt):
    
       sudo sysctl net.ipv4.ip_forward=1
       sudo iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
       sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
       sudo iptables -A FORWARD -i enp0s20u7 -o eno1 -j ACCEPT
    On my Arch Linux desktop x64 PC, 'eno1' is the name of the Ethernet adapter connected to my router (and the connection I'll be sharing), and 'enp0s20u7' is the name assigned to the BBBlue. Use `ip link` to find the corresponding names for your devices.
    
    If you are using a Windows computer, you can tell it to share with the BBBlue by fast-forwarding to about 5:00 of this Derek Molloy video: http://derekmolloy.ie/beaglebone/getting-started-usb-network-adapter-on-the-beaglebone/. It should be noted that on recent Windows builds, you may also need to reset the Windows firewall first to get this to work. Crazy, I know.  
    
    If internet sharing isn't an option, another possibility is a USB-to-Ethernet dongle like this one: http://accessories.ap.dell.com/sna/productdetail.aspx?c=sg&l=en&s=bsd&cs=sgbsd1&sku=470-ABNL). One can plug this into the BBBlue and then connect it directly to the router. Note that, in my case, I had to supply extra power to the BBBlue via its 2s LiPo connector or jack plug for the dongle to work. The BBBlue enumerates the dongle as device 'usb2', and sets it up automatically (well, sometimes it's necessary to reboot in order for `ip link` to show that the device is up).
    
    Whichever way you go, you can verify you have internet by typing: `ping -c 3 google.com`

5) Update and install all required supporting software:

       sudo apt-get -y update
       sudo apt-get -y dist-upgrade
       sudo apt-get install -y cpufrequtils git
6) Update scripts: `cd /opt/scripts && git pull`
7) Specify real-time kernel 4_19: `sudo /opt/scripts/tools/update_kernel.sh --lts-4_19 --bone-rt-channel`
8) Specify device tree binary to be used at startup: `sudo sed -i 's/#dtb=/dtb=am335x-boneblue.dtb/g' /boot/uEnv.txt`
9) Specify device tree overlays: `sudo sed -i 's|#dtb_overlay=/lib/firmware/<file8>.dtbo|dtb_overlay=/lib/firmware/BB-I2C1-00A0.dtbo\n#dtb_overlay=/lib/firmware/BB-UART4-00A0.dtbo\n#dtb_overlay=/lib/firmware/BB-ADC-00A0.dtbo|g' /boot/uEnv.txt`
10) Specify U-Boot overlay: `sudo sed -i 's|uboot_overlay_pru=/lib/firmware/AM335X-PRU-RPROC-4-14-TI-00A0.dtbo|#uboot_overlay_pru=/lib/firmware/AM335X-PRU-RPROC-4-14-TI-00A0.dtbo|g' /boot/uEnv.txt`
11) (cont.) `sudo sed -i 's|#uboot_overlay_pru=/lib/firmware/AM335X-PRU-UIO-00A0.dtbo|uboot_overlay_pru=/lib/firmware/AM335X-PRU-UIO-00A0.dtbo|g' /boot/uEnv.txt`
12) Set clock frequency: `sudo sed -i 's/GOVERNOR="ondemand"/GOVERNOR="performance"/g' /etc/init.d/cpufrequtils`
13) Disable Bluetooth (optional): `sudo systemctl disable bb-wl18xx-bluetooth.service`
14) Maximize the microSD card's existing partition (optional): `sudo /opt/scripts/tools/grow_partition.sh`
15) Reboot now: `sudo reboot`

    *Please note that if you experience an RCOutputAioPRU.cpp:SIGBUS error, there's a couple of things to try. Wiping the eMMC boot sector with `sudo dd if=/dev/zero of=/dev/mmcblk1 bs=1M count=10` has been known to help, as has updating the bootlaoder with `sudo /opt/scripts/tools/developers/update_bootloader.sh`.*

## Part 2 - Putting ArduPilot on the BeagleBone Blue
16) When the BBBlue comes back up, we need to create a few text files. First, the ArduPilot environment configuration file, /etc/default/ardupilot:

	(Hint: type `sudoedit /etc/default/ardupilot`, and insert your own target IP address, e.g. 192.168.0.13)
        
        TELEM1="-C /dev/ttyO1"
        TELEM2="-A udp:<target IP address>:14550"
        GPS="-B /dev/ttyO2"
        RANGER="-F /dev/ttyO5"
    This is a pretty typical config. It breaks down like this:
    
    Switch -C maps ArduPilot's "Telem1" serial port (SERIAL1, default 57600) to the BBBlue's UART1. For example, I have a RFDesign 868x radio modem connected to UART1. It is the bidirectional data link with my drone. It sends various telemetry data to the base station, and receives commands and RTK differential corrections from the base station.
    
    Switch -A maps ArduPilot's "Console" serial port (SERIAL0, default 115200) to a protocol, target IP address and port number of one's choosing. For example, this allows me to have MAVLink data coming over WiFi for test purposes. Really useful, especially since it seems to be reliably auto-sensed by ground control station software like Mission Planner and QGroundControl.
    
    Switch -B maps ArduPilot's "GPS" serial port (SERIAL3, default 57600) to the BBBlue's UART2 (the UART confusingly marked 'GPS' on the board itself). For example, I have a u-blox NEO-M8P connected to UART2.
    
    Switch -F maps one of ArduPilot's "Unnamed" serial ports (SERIAL5, default 57600) to the BBBlue's UART5. In my case, this is for a laser rangefinder.
    
    Other possibilities exist, namely:
    
        Switch -A  -->  "Console", SERIAL0, default 115200
        Switch -B  -->  "GPS", SERIAL3, default 57600
        Switch -C  -->  "Telem1", SERIAL1, default 57600
        Switch -D  -->  "Telem2", SERIAL2, default 38400
        Switch -E  -->  Unnamed, SERIAL4, default 38400
        Switch -F  -->  Unnamed, SERIAL5, default 57600
    Consult the official ArduPilot documentation for more details on the various serial ports: http://ardupilot.org/plane/docs/parameters.html?highlight=parameters
    
17) Next, we'll create the ArduPilot systemd service files, one for ArduCopter, /lib/systemd/system/arducopter.service:
        
        [Unit]
        Description=ArduCopter Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arduplane.service ardurover.service antennatracker.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/arducopter $TELEM1 $TELEM2 $GPS $RANGER

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
    One for ArduPlane, /lib/systemd/system/arduplane.service:
        
        [Unit]
        Description=ArduPlane Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arducopter.service ardurover.service antennatracker.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/arduplane $TELEM1 $TELEM2 $GPS $RANGER

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target    
    And one for ArduRover, /lib/systemd/system/ardurover.service:
    
        [Unit]
        Description=ArduRover Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arducopter.service arduplane.service antennatracker.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/ardurover $TELEM1 $TELEM2 $GPS $RANGER

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
    While I'm here, how about AntennaTracker, too? Create /lib/systemd/system/antennatracker.service:
    
        [Unit]
        Description=AntennaTracker Service
        After=networking.service
        StartLimitIntervalSec=0
        Conflicts=arducopter.service arduplane.service ardurover.service

        [Service]
        EnvironmentFile=/etc/default/ardupilot
        ExecStartPre=/usr/bin/ardupilot/aphw
        ExecStart=/usr/bin/ardupilot/antennatracker $TELEM1 $TELEM2 $GPS $RANGER

        Restart=on-failure
        RestartSec=1

        [Install]
        WantedBy=multi-user.target
18) Pause for a moment to create a directory: `sudo mkdir -p /usr/bin/ardupilot`
    
    Then, carry on with creating what I call the ArduPilot hardware configuration file, /usr/bin/ardupilot/aphw, which is run by the services prior to running the ArduPilot executables:

        #!/bin/bash
        # aphw
        # ArduPilot hardware configuration.

        /bin/echo 80 >/sys/class/gpio/export
        /bin/echo out >/sys/class/gpio/gpio80/direction
        /bin/echo 1 >/sys/class/gpio/gpio80/value
        /bin/echo pruecapin_pu >/sys/devices/platform/ocp/ocp:P8_15_pinmux/state
    Lines 5 to 7 switch on power to the BBBlue's +5V servo rail - i.e. for when you're using servos. Not necessary for ESCs.
    
    Line 8 enables the PRU.
    
    Use `sudo chmod 0755 /usr/bin/ardupilot/aphw` to set permissions for this file.
    
19) Almost there! You must now obtain the latest ArduCopter, ArduPlane, etc. executables, built specifically for the BBBlue's Arm architecture, and place them in the /usr/bin/ardupilot directory. Mirko Denecke has them on his site here: http://bbbmini.org/download/blue/

    And I've built my own copies here in this repository: https://github.com/imfatant/test/blob/master/bin/

    Be sure to set their permissions with: `sudo chmod 0755 /usr/bin/ardupilot/a*`

    If you find that you need to build them from scratch yourself, do not be intimidated - this is not too difficult. Plus it means you'll be able to build your own customized ArduPilot software.

    Compiling them on the BBBlue itself is an option, but takes an absolute age. Patrick Poirier explains the process for the BBBMINI (based on a BeagleBone Black) on his site. Here's the BBBlue-specific procedure assuming you've followed all the steps so far, and are in the /home/debian directory:
    
        sudo apt-get install g++ make pkg-config python python-dev python-lxml python-pip
        sudo pip install future
        git clone https://github.com/ArduPilot/ardupilot
        cd ardupilot
        git branch -a  # <-- See all available branches.
        git checkout Copter-3.6  # <-- Select one of the ArduCopter branches.
        git submodule update --init --recursive
        ./waf configure --board=blue  # <-- BeagleBone Blue.
        ./waf
        sudo cp ./build/blue/bin/a* /usr/bin/ardupilot
    Copy the executable(s) from /home/debian to /usr/bin/ardupilot. Again, be sure to set their permissions.
    
    Patrick also provides instructions to cross-compile them on a powerful desktop x64 PC in Ubuntu, which is much, much faster. Here, I will run through the process of cross-compiling them in Arch Linux (which happens to be God's Own Linux Distro):
    ---+++ This method doesn't currently work because Arch Linux and Debian Stretch use different glibc versions. +++---
    
        sudo pacman -Syu
        gpg --recv-keys 79BE3E4300411886 38DBBDC86092693E 79C43DFBF1CF2187 13FCEF89DD9E3C4F 16792B4EA25340F8
        yay -S arm-linux-gnueabihf-glibc-headers arm-linux-gnueabihf-glibc arm-linux-gnueabihf-gcc  # <-- Note that I'm using yay instead of yaourt. Also, ensure you haven't set any C/C++ env variables.
        sudo ln -s pkg-config /usr/bin/arm-linux-gnueabihf-pkg-config
        sudo pacman -S --noconfirm python-pip
        sudo pip install future
        git clone https://github.com/ArduPilot/ardupilot
        cd ardupilot
        git config user.name <your username>
        sed -i 's/command -v yaourt/command -v yay/g' ./Tools/scripts/install-prereqs-arch.sh  # <-- Skip this if using yaourt.
        sed -i 's/yaourt -S --noconfirm --needed/yay -S --noconfirm/g' ./Tools/scripts/install-prereqs-arch.sh  # <-- Skip if using yaourt.
        git commit -a --allow-empty-message -m ''  # <-- The lazy option.
        ./Tools/scripts/install-prereqs-arch.sh
        git fetch --prune  # <-- Updates the repository.
        git branch -a  # <-- See all available branches.
        git checkout Copter-3.6  # <-- Select one of the ArduCopter branches.
        git submodule update --init --recursive
        ./waf configure --board=blue  # <-- BeagleBone Blue.
        ./waf
        scp ./build/blue/bin/a* debian@192.168.7.2:/home/debian  # <-- Finally, copy the built executable(s) over to the BBBlue.
    
20) To get ArduPilot going, choose which flavour you want and type ONE of these:

        sudo systemctl enable arducopter.service
        sudo systemctl enable arduplane.service
        sudo systemctl enable ardurover.service
        sudo systemctl enable antennatracker.service
    After you reboot, your ArduPilot should inflate automatically. Look for the flashing red LED!
    
    It'll help to familiarise yourself with `systemctl` (https://www.freedesktop.org/software/systemd/man/systemctl.html). Some useful example commands:
    
        sudo systemctl disable <name of service>
        sudo systemctl start <name of service>
        sudo systemctl stop <name of service>
	
## Part 3 - Connecting the peripherals
21) A basic minimum configuration is likely to include:
    - An R/C receiver.
    - A GPS receiver (with or without integrated compass).
    - A radio modem for a bidirectional data link, particularly at longer ranges.

    (The BBBlue's onboard WiFi is great for debugging and testing at close range if 2.4 GHz is available, but for anything more interesting, a dedicated bidirectional data link is recommended. Also bear in mind the type and placement of antennas that all these items use.)

    A good place to begin is the quickstart pinout diagram below (save the file and open it in any good image viewer for better resolution). I will refer to it in the paragraphs that follow.
    
    But first a few words about connectors, cables, and the tools you'll need, particularly if you're to fashion your own cables.
    
    I'm going to make a few strong recommendations because if you're new to this, you could waste a great deal of time, effort and money mucking about. The connector type mainly used is the JST-SH 1.0 mm pitch. You should buy some females, in 4 and 6-way sizes, and the crimp contacts to go with (buy lots so you can practise - you'll need to). Then, get a couple of metres of 28 or 30 AWG silicone covered wire, in at least five different colours (red, black, yellow, green, blue and white). *And*, it's very important to purchase the right tools for the job, namely, Engineer PA-06 wire strippers with the depth gauge attachment, and Engineer PA-09 'connector pliers' (they're Japanese - we tend to call them crimpers).
    
    By the way, the JST-SH is a horrible connector. It may be tiny, but that makes it hard to work with, plus it has a habit of working loose.
    
    I should also mention other connector types you'll come across (on the other end of the cables you'll be making up). The JST-GH 1.25 mm pitch is probably my favourite connector - it's sturdy and has an integrated catch mechanism. In the wild, you'll see 4, 5 and 6-way. Next, there's the Molex PicoBlade 1.25 mm pitch connector, also nice, available in receptacle (male) and plug housing (female) varieties - great for off-board connections. Grab some from 2-way through to 6-way. And of course, there's the venerable Mini-PV 2.54 mm connector (a.k.a. Harwin M20 / Futaba / Hitec / 'standard black R/C-style connector').
    
    Naturally, you'll need to buy the crimp contacts for all these (they're all different).
    
    DigiKey, Mouser, RS Components, Farnell, and so on, are the places to shop for these components. Try to make one BIG order because they charge a small fortune for shipping. As for information on all the rest of the bits and pieces you'll need, go here to Joshua Bardwell's FPV website: https://www.fpvknowitall.com/ultimate-fpv-shopping-list/
    
    So, back to connecting the peripherals:
    
![alt text](https://github.com/imfatant/test/blob/master/docs/BBBlue-ArduPilot.jpg)

- The R/C receiver: This can be powered off any +5V pin and a GND. The +5V servo rail, providing it's powered, will of course do. All that remains is to connect the receiver's SBUS OUT, DSM OUT or PPM OUT to one of the two SBUS pins marked on the diagram. The following receivers have been tested and are known to work:

	FrSky (https://www.frsky-rc.com/): R-XSR, XR4SB, X6R, X8R, R9 Slim and R9 Mini (both EU LBT 868 MHz and Universal 915 MHz firmwares).
	
	Spektrum (https://www.spektrumrc.com/): AR7700 DSMX with PPM/SRXL/Remote Rx.
	
	TBS (http://team-blacksheep.com/): 'Full' Crossfire with Nano Rx (Rx set to SBUS mode).
	
	Please contact me if you would like to add some information here about your own gear.

    Incidentally, you may hear a lot about the 'inverted SBUS' signal on the web, and perhaps find it confusing. To be clear: SBUS is simply a serial data communication protocol Futaba came up with, and which FrSky copied. It's inverted compared to the UART 'standard'. Fortunately, ArduPilot Blue expects this inverted SBUS signal, so there's no need for a signal inverter or a hack.

- The GPS receiver: Most people use the u-blox receivers, particularly the NEO-M8N and NEO-M8P. The former can be easily and cheaply obtained online from Chinese companies like HobbyKing, in a variety of usually disk-shaped packages. Conveniently, these packages contain the receiver itself, a very small ceramic patch antenna, and often include a compass chip. While the BBBlue already has an onboard compass (the AKM AK8963), ArduPilot can be configured to use this 'external' compass instead - something to consider if it's positioned in a less (electro)magnetically noisy location on the drone.

    The (much) more expensive alternative to the NEO-M8N is the NEO-M8P. This receiver supports a mode of operation known as 'RTK', or Real-Time Kinematic, which can achieve positional accuracies of a few centimetres in real-time. However, this kind of performance comes at a cost approximately ten times the price of the NEO-M8N, and that's before you add in the base station. I will devote a special section to the M8P later in the guide.
    
    I highly recommend starting out with an M8N, if only to get a feel for things. It's likely you'll have to chop off the connector(s) it comes with and break out the GPS lines to a 6-way JST-SH which you'll plug into the GPS serial UART on the BBBlue, and the external compass lines to a 4-way JST-SH which plugs into the I2C port. The niggle is that although you'll be powering the M8N with +5V, its GPS signal to the BBBlue must not exceed +3.3V. I believe most M8N modules step the voltage down to +3.3V internally, so you're OK. But you should check, if necessary with an oscilloscope. Failure to do so could damage the BBBlue.
    
    And things are evolving! You might want to check out the triple-band RTK-capable ZED-F9P receiver, again from u-blox. Interestingly, it seems that u-blox are making their own antenna for it, the ANN-MB. Personally though, I'd like a _really_ small, lightweight antenna, so if you know of one, please let me know.
    
The ArduPilot parameter settings files: /var/APM/{ArduCopter.stg,ArduPlane.stg,APMrover2.stg,AntennaTracker.stg}

UNDER CONSTRUCTION!

sudo apt-get install i2c-tools

sudo i2cdetect -r -y 0
sudo i2cdetect -r -y 1
sudo i2cdetect -r -y 2

    $ sudo i2cdetect -r -y 2
    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
    00:          -- -- -- -- -- -- -- -- -- 0c -- -- -- 
    10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    60: -- -- -- -- -- -- -- -- 68 -- -- -- -- -- -- -- 
    70: -- -- -- -- -- -- 76 --    
68 = InvenSense MPU-9250 IMU (onboard), 0c = AKM AK8963 compass (onboard), 76 = Bosch BMP280 barometer (onboard).

    $ sudo i2cdetect -r -y 1
    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
    00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
    10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- 1e -- 
    20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
    70: -- -- -- -- -- -- -- --
1e = Honeywell HMC5843 compass (external) - often comes integrated into the inexpensive u-blox NEO-M8N-based GPS modules.

## Getting started with Ground Control Station (GCS) software
22) Download either Mission Planner (http://firmware.ardupilot.org/Tools/MissionPlanner/MissionPlanner-latest.msi) for Windows or QGroundControl (http://qgroundcontrol.com/) for Linux & Windows. Both these programs will connect to streams of MAVLink data coming over the network (via UDP on port 14550, for example) or over COM ports. Some trivial configuration may be required, but both programs do a great job of auto-sensing and auto-connecting to traffic all by themselves.

    If you're having difficulty establishing a link, look at the following:
    - Ensure you've opened the necessary ports in the GCS computer's firewall. Perhaps even disable the firewall temporarily.
    - Be absolutely certain of the GCS computer's IP address, because if you happen to be 'dualing' Windows and Linux on the same machine, routers will sometimes assign different IPs to each of the OSes.
    - If you're getting a 'port is already open'-type error, turn off the GCS software's auto-connect feature, restart the program, and try again manually.
    - Remember that a server set to listen on the loopback address (127.0.0.1) won't see external nodes on your network.

    With the link established, you'll see the GCS program's artificial horizon moving as you turn and tilt the BBBlue. Now I suggest that you run ArduPlane specifically (as opposed to ArduCopter, etc) and get a servo moving in the 'Manual' flight mode. This is a convenient way to test that your receiver is working properly with ArduPilot Blue. Use QGoundControl's 'Parameters' page to set the following, and then reboot the BBBlue:

        Under 'Standard':
          FLTMODE1  -->  Manual
          FLTMODE2  -->  Manual
          FLTMODE3  -->  Manual
          FLTMODE4  -->  Manual
          FLTMODE5  -->  Manual
          FLTMODE6  -->  Manual
      
        Under 'Advanced':
          FLTMODE_CH ---> 5
    Notice that every flight mode switch position is set to Manual so that you're absolutely guaranteed a clean pass-through signal to your test servo as you move the transmitter's stick. I've also set the flight mode switching channel to 5. ArduCopter defaults to 5, but ArduPlane defaults to 8 (for various reasons that aren't interesting).

    The servo should be plugged into the bottom-most servomotor output header (oriented as on the pinout diagram).
    
    Power to the servo(s) must be supplied via the 2s LiPo connector, NOT the jack plug.

## Extras
99) Equipping your BBBlue-based drone with a Bluetooth speaker can be fun, providing that the Bluetooth RF transmissions don't interfere with any other systems. There's a bunch of info out there on BlueZ/PulseAudio/ALSA, but fortunately, it all boils down to something pretty simple.

    a) First, install the necessary software (whether using a console or IoT image):

           sudo apt-get install -y bluetooth pulseaudio pulseaudio-module-bluetooth alsa-utils
    b) Enable Bluetooth (if disabled): `sudo systemctl enable bb-wl18xx-bluetooth.service`
    
    c) Next, edit /etc/pulse/default.pa to contain the following lines (i.e. commented out):

           ### Automatically suspend sinks/sources that become idle for too long
           # load-module module-suspend-on-idle
    d) Then restart: `sudo reboot`
    
    e) When the BBBlue is back up, put your Bluetooth speaker into pairing mode, and do:
    
           bluetoothctl
           scan on
           agent on
           default-agent
           pair <Bluetooth speaker's MAC address>  # <--- e.g. AB:58:EC:5C:0C:03
           connect <Bluetooth speaker's MAC address>  # <--- Sometimes unnecessary.
           trust <Bluetooth speaker's MAC address>
           scan off
           exit

    f) Then finally (or after booting up):
    
           pulseaudio --start
           echo "connect <Bluetooth speaker's MAC address>" | bluetoothctl
           pactl list  # <--- Use this to check that your Bluetooth speaker has been picked up by PulseAudio.
           pacmd set-card-profile 0 a2dp_sink
           aplay /usr/share/sounds/alsa/Front_Center.wav
    That's all there is to it. By the way, if you're going to use a speech synthesizer, I recommend Festival.
