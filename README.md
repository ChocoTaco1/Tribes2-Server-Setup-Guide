![Tribes2 Server Setup Guide](https://cdn.discordapp.com/attachments/521797012014759970/678460505597149214/TacoServer.png)

# Tribes2 Server Setup Guide by ChocoTaco
## A guide for setting up a Tribes 2 server via vps, ssh, and vnc

---

Discord: [Tribes 2 Discord](https://discord.gg/Y4muNvF)

### Target Environment
 - This guide is for Debian 12 (Bookworm)
 - This could probably work on other distros varying packages and commands

### Choosing a host
 - Any VPS host could probably run a t2 server depending on how much you want to pay...
 - Vultr is probably to easiest to use - https://www.vultr.com/
 - Linode probably a great option too (Now Akamai) - https://www.linode.com/
 - GCE (Google Compute Engine) being probably the more difficult option - https://cloud.google.com/compute/
 - I'm sure there's others

### Connecting...
 - SSH as root into your vps (Your ssh password can usually be found on your host vps website)
 - In windows use puTTy (ssh tunneling for vnc: https://helpdeskgeek.com/how-to/tunnel-vnc-over-ssh/)
 - In Linux use ssh root@ip.address -L 5901:localhost:5901

### Setting up Debian
 - Once youre in...Make user and set a Password. "t2server" as whatever username you want
		adduser t2server

		usermod -aG sudo t2server

 - This is for winetricks later
		echo "deb http://deb.debian.org/debian bookworm contrib" > /etc/apt/sources.list

		sudo apt update && sudo apt upgrade

# Log out as root and log back in as t2server or your newly created user. This is very important...

 - Install desktop
		sudo apt install xfce4 xfce4-goodies tightvncserver xfonts-base firefox-esr synaptic file-roller git winetricks htop curl zenity

 - Start vnc server, make password...8 characters
		vncserver

 - Kill vnc server
		vncserver -kill :1

 - Open vnc config file
		nano ~/.vnc/xstartup

 - Copy this inside xstartup, ctrl-o save. This says to start our desktop when we start our vnc
		#!/bin/bash
		xrdb $HOME/.Xresources
		startxfce4 &

//Repair permissions
sudo chmod +x ~/.vnc/xstartup

//Installing WINE
//Install the wine repo from: https://wiki.winehq.org/Debian
sudo dpkg --add-architecture i386
sudo mkdir -pm755 /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/winehq-archive.key https://dl.winehq.org/wine-builds/winehq.key
sudo wget -NP /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/debian/dists/bookworm/winehq-bookworm.sources
//This install wine-development, typically theres no issues. If you have issues install "winehq-stable".
sudo apt install --install-recommends winehq-devel

//Get files
wget https://www.the-construct.net/downloads/tribes2/tribes2gsi.exe &&
wget https://www.tribesnext.com/files/TribesNext_rc2a.exe &&
wget https://tribes2stats.com/files/mods/classic_v152.zip &&
git clone https://github.com/ChocoTaco1/TacoServer &&
git clone https://github.com/ChocoTaco1/TacoMaps &&
wget https://www.dropbox.com/s/bvh9631a4mtuisf/msvcrt-ruby190.zip &&
wget https://www.dropbox.com/s/tt4utbqkzh791y8/SetPerfCounter.zip

//Start our VNC Server (1800x950 can be whatever you want)
vncserver -geometry 1800x950
//VNC into server (In windows use real vnc/In linux use Reminna). Setup to use localhost:5901 while youre logged into the server using ssh. Enter your 8 character vnc password
//You should be greeted with a Debian desktop

//Install T2 in wine (Installs to /home/t2server/.wine/drive_c/Dynamix)
wine Tribes2gsi.exe
//Install Tribesnext_rc2a
wine TribesNext_rc2a.exe
//Update Classic to 1.5.2 (1.0 is included in Tribes2gsi) Just delete classic folder in Gamedata and extract classic 1.5.2 there
//Extract TacoServer to Classic (Overwriting any existing files)
//Extract Tacomaps to Classic/maps/
//Extract msvcrt-ruby190 to Gamedata (Overwriting any existing files)
//Extract Perf Counter to Classic/scripts/autoexec (This locks server to HighPerformanceCounter=0)

//Optional: Download Loops sha1 fix (Extract Server.dll and wine-injector to GameData Folder, put t2csri_serverside_looped.cs in Gamedata/Classic/scripts/autoexec)
wget https://cdn.discordapp.com/attachments/1154920105097040023/1154923875562422382/t2-auth-faster.zip?ex=6518b0ad&is=65175f2d&hm=5be90f772b1c0a0046331ffce8350a9c686ba242dcaa7a2350ebd433798f81cc&
//This requires vcrun22 to work so...
winetricks -q --force vcrun2022

//Firewall (Optional)
//Install IPTables
sudo apt install iptables iptables-persistent
//Make sure ports are open
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p udp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 28000 -j ACCEPT
sudo iptables -I INPUT -p udp --dport 28000 -j ACCEPT
//Also make sure ports are open within the host service youre using with their firewall. Make sure to open port 22 for your ssh port as well.

//Start t2 server (without loops fix)
taskset -c 0 wineconsole Tribes2.exe -dedicated -mod Classic
//Start t2 server (with loops fix)
taskset -c 0 wine wine_injector.exe Server.dll Tribes2.exe 28000 -dedicated -mod Classic
//"taskset -c 0" is locking the server process to one thread

//Ideally you can start your server with a bash script ie: sh startserver.sh
//Update the start command with whatever you choose to use with or without loops fix
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

//For security you can use an ssh key to login, lockdown ssh thru firewall, lock ssh to your home ip, and move ssh ports around
//Adding an ftp server can also help with file management, granted you locked down the ports
//If everything is setup correctly, your server should show up on the main server within a few minutes

//Other things that can be done...
//Higher Priorty: To allow your user to set a higher priority use...
sudo nano /etc/security/limits.conf
//At the end add
@t2server        -          nice          -20
//New startup would be something with "nice -n -5" added...
//nice -n -5 taskset -c 0 wineconsole Tribes2.exe -dedicated -mod Classic
