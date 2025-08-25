![Tribes 2 Server Setup Guide](https://cdn.discordapp.com/attachments/521797012014759970/678460505597149214/TacoServer.png)

# Tribes 2 Server Setup Guide by ChocoTaco
## A guide for setting up a Tribes 2 server via vps, ssh, and vnc

---

Discord: [Tribes 2 Discord](https://discord.gg/Y4muNvF)

Tacoserver: [Tacoserver Github](https://github.com/ChocoTaco1/TacoServer)

---
### Target Environment
 - This guide is for Debian 12 (Bookworm)
 - This could probably work on other distros varying packages and commands

---
### Choosing a host
 - Any VPS host could probably run a t2 server depending on how much you want to pay...
 - Vultr is probably to easiest to use - https://www.vultr.com/
 - Linode probably a great option too (Now Akamai) - https://www.linode.com/
 - GCE (Google Compute Engine) being probably the more difficult option - https://cloud.google.com/compute/
 - I'm sure there's others

---
### Connecting to your vps
 - SSH as root into your vps (Your root password can usually be found on your host vps website): https://www.linode.com/docs/guides/connect-to-server-over-ssh-on-windows/
 - In Linux use

 		ssh root@ip.address -L 5901:localhost:5901

---
### Setting up Debian
 - Once youre in... make a user and set a password. Change `t2server` as whatever username you want...

		adduser t2server

 - Then...

		usermod -aG sudo t2server


 - Update packages

		sudo apt update && sudo apt upgrade

---
# Log out as root and log back in as t2server or your newly created user. This is very important...
---

 - Install desktop

		sudo apt install xfce4 xfce4-goodies tightvncserver xfonts-base firefox-esr synaptic file-roller git winetricks htop curl zenity

 - Start vnc server, make password...8 characters

		vncserver

 - Kill vnc server

		vncserver -kill :1

 - Open vnc config file

		nano ~/.vnc/xstartup

 - Copy this inside xstartup at the end, `ctrl-o` save, `ctrl-x` to exit. This says to start our desktop when we start our vnc

		#!/bin/bash
		xrdb $HOME/.Xresources
		startxfce4 &

 - Repair permissions

		sudo chmod +x ~/.vnc/xstartup

---

### Installing WINE
- Install the wine repo by following the directions here: https://wiki.winehq.org/Debian


		sudo dpkg --add-architecture i386 &&
		sudo mkdir -pm755 /etc/apt/keyrings &&
		sudo wget -O /etc/apt/keyrings/winehq-archive.key https://dl.winehq.org/wine-builds/winehq.key &&
		sudo wget -NP /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/debian/dists/bookworm/winehq-bookworm.sources &&
		sudo apt update

 - This will install wine-development, typically theres no issues.

		sudo apt install --install-recommends winehq-stable

 - If you want the development branch install `winehq-devel`

### Get all the files
 - Download:

		wget https://www.the-construct.net/downloads/tribes2/tribes2gsi.exe &&
		wget https://www.tribesnext.com/files/TribesNext_rc2a.exe &&
		wget https://tribes2stats.com/files/mods/classic_v152.zip &&
		git clone https://github.com/ChocoTaco1/TacoServer &&
		git clone https://github.com/ChocoTaco1/TacoMaps &&
		wget https://www.dropbox.com/s/bvh9631a4mtuisf/msvcrt-ruby190.zip &&
		wget https://www.dropbox.com/s/tt4utbqkzh791y8/SetPerfCounter.zip

---

### Start your VNC Server
 - Change `1800x950` to be whatever resolution you want

		vncserver -geometry 1800x950

 - Use Real VNC for windows - https://www.realvnc.com/en/
 - Use Reminna for linux - https://remmina.org/
 - Setup to use `localhost:5901` while youre logged into the server using ssh. Enter your 8 character vnc password. You should be greeted with a Debian desktop.

---

### Setup T2
 - Install T2 in wine (Typically installs to `/home/t2server/.wine/drive_c/Dynamix`).

		wine Tribes2gsi.exe

 - Install Tribesnext_rc2a

		wine TribesNext_rc2a.exe

 - Update Classic to 1.5.2 (1.0 is included in Tribes2gsi) Just delete classic folder in Gamedata and extract classic 1.5.2 there
 - Extract TacoServer to Classic (Overwriting any existing files)
 - Extract Tacomaps to Classic/maps/
 - Extract msvcrt-ruby190 to Gamedata (Overwriting any existing files)
 - Extract Perf Counter to Classic/scripts/autoexec (This locks server to HighPerformanceCounter=0)
---

 - Optional: Download Loop's sha1 fix (Extract Server.dll and wine-injector to GameData Folder, put t2csri_serverside_looped.cs in Gamedata/Classic/scripts/autoexec)

		wget https://www.dropbox.com/scl/fi/gu5eixt8q078fnweahov3/t2-auth-faster.zip?rlkey=gy4u4789ylfhvr1iqi2zp30bz&st=ka7s1r74

 - Loop's fix requires vcrun22 to work...

		winetricks -q --force vcrun2022

---

### Firewall (Optional)
 - Install iptables

		sudo apt install iptables iptables-persistent

 - Make sure iptable ports are open

		sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
		sudo iptables -I INPUT -p udp --dport 80 -j ACCEPT
		sudo iptables -I INPUT -p tcp --dport 28000 -j ACCEPT
		sudo iptables -I INPUT -p udp --dport 28000 -j ACCEPT

- Also make sure ports are open within the host service youre using with their firewall. Make sure to open port 22 for your ssh port as well.

---

### Starting the Server
 - Start t2 server (without loop's fix)

		taskset -c 0 wineconsole Tribes2.exe -dedicated -mod Classic

 - Start t2 server (with loop's fix)

		taskset -c 0 wine wine_injector.exe Server.dll Tribes2.exe 28000 -dedicated -mod Classic

 - `taskset -c 0` is locking the server process to one cpu only

---

### Using a bash script
 - Ideally you can start your server with a bash script `ie: sh startserver.sh`

		#!/bin/sh
		BASEDIR=/home/t2server/.wine/drive_c/Dynamix/Tribes2/GameData/
		cd ${BASEDIR}

		while true; do
			echo "Waiting 30 seconds for all wine processes to close..."
			sleep 30
			wineserver -k9 && killall wine && kill $(lsof -t -i:28000) && kill $(lsof -t -i:80) && killall winedevice.exe
			echo "Removing dsos..."
			find ${BASEDIR} -name \*.dso -execdir /bin/rm {} \;
			echo "Starting Tribes2 server..."
			WINEDEBUG=-all,-fixme taskset -c 0 wineconsole Tribes2.exe -dedicated -mod Classic
		done

 - Update the start command with whatever you choose to use regarding loop's fix

 - You can create a shortcut to this start server script by right-clicking on the desktop and choosing create a launcher

		Name: Start Server | Command: sh /home/t2server/startserver.sh | Click "Start in Terminal"

---

### Other Things
 - For security you can use an ssh key to login

 - Make sure to use Vultr Firewall or something to lock down all ports except 80, 28000, and 22 (ssh)

 - Adding an ftp server can also help with file management, granted you locked down the ports

 - Higher Priorty: To allow your user to set a higher priority use add at the end of... (I wouldnt recommend this on shared cpus)

		sudo nano /etc/security/limits.conf

		@t2server        -          nice          -20

	New startup would be something with `nice -n -5` added...

		nice -n -5 taskset -c 0 wineconsole Tribes2.exe -dedicated -mod Classic

---

### Post Preview Patch
  - Updating a server to the newly released preview:

		Visit Thread: https://tribesnext.com/forum/discussion/4430/preview-qol-fixes-update
		Back up your server T2 folder
		Download the latest release
		Use wine `TribesNEXT_XXXXXXXXX_preview.exe` to initiate the installer for wine
		Or in windows execute the exe
		Install to your server T2 GameData folder
		If you're using Loops fix or anything like that, remove it from the launch parameters

  - To take advantage of extra bandwidth set,

		$pref::Net::PacketRateToClient = 64;
		$pref::Net::PacketSize = 1000;


  - To prevent the new UE box from popping up set,

		$pref::Engine::ExitOnException = true;


  - If you don't want the linux icon showing up use

		$Host::Linux = 0;


  - Loadingscreen Safeguards:

		To ensure clients dont get stuck on the loading screen these changes need to be made
		`https://github.com/ChocoTaco1/TacoServer/commit/7cd1cb8815e19990ab5a0cf632aedc82726333ed`

---
---

# Success!
## If everything is setup correctly your server should show up on the master server within a few minutes
Good Luck!