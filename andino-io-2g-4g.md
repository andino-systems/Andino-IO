----------
### Setup PPP for the 2G/EDGE SomCom800L or 4G on the SimCom7600E Modem

For the SimCom 7600 PPP is working but not recommended because it is a high bandwidth module.

[Please follow this instructions to use the NDIS interface for the 7600E](https://techship.com/faq/how-to-integrate-simcom-sim7500-sim7600-series-linux-ndis-driver-without-rebuilding-kernel/)


Open the Housing and insert the Micro SIM.
![Andino Lora - Insert SIM Card](andino-io-gsm-inside.png)

Remove PIN from SIM!

The Modem is connected to the internal UART of the Raspberry Pi.  
The Reset Line of the Modem is connected to GPIO 17.

#### Useful scripts
see at top to download.

    ./init.sh   	initialize the Ports to reset Modem
    ./stop.sh   	Stop (hold in reset) the Modem
    ./start.sh  	Start the Modem (release reset)
    ./restart.sh	Stop and Start again
    
    ./dial.sh   	Init the DialIn
    ./hangup.sh 	Shutdown PPP
    
    ./log.sh		Tail the log
    
#### Useful commands

	ifconfig
	ping 8.8.8.8
	sudo route add default dev ppp0
	sudo ifconfig eth0 down
	sudo ifconfig eth0 up

#### Prepare the Debian

    sudo nano /boot/cmdline.txt
    remove console...
    dwc_otg.lpm_enable=0   >>console=serial0,115200 console=tty1<<   root=/dev/mmcblk0
    
    sudo nano /boot/config.txt
    Add to the end:
    
    >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
    enable_uart=1
    dtoverlay=pi3-disable-bt-overlay
    dtoverlay=pi3-miniuart-bt
    <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
    
    reboot
    
Install PPP and some Tools
    
    sudo apt-get install ppp screen elinks minicom
    
    Test Modem: 
    sudo minicom --setup
    +-----------------------------------+
    | A -Serial Device  : /dev/ttyAMA0  |
    | B - Lockfile Location : /var/lock |
    | C -   Callin Program  :           |
    | D -  Callout Program  :           |
    | E -Bps/Par/Bits   : 115200 8N1    |
    | F - Hardware Flow Control : No    |
    | G - Software Flow Control : No    |
    |                                   |
    |Change which setting?              |
    +-----------------------------------+
    
    minicom
    at
    OK
    ati
    SIM800 R14.18
    OK
    # Show Error as text
    AT+CMEE=2
    # SIM Ready?
    AT+CPIN?
    +CPIN: READY
    # Network available?
    AT+COPS?
    +COPS: 0,0,"D1"
    # Network quality?
    AT+CSQ
    +CSQ: 4,0
    # Set APN
    AT+CGDCONT=1,"IP","internet.telekom","",0,0


#### Setup ppp
"internet.telekom" is my APN and *must* be changed to the providers APN

    
sudo nano /etc/ppp/peers/rnet
 
    # >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
    #imis/internet is the apn for idea connection
    connect "/usr/sbin/chat -v -f /etc/chatscripts/gprs -T internet.telekom"
    
    # For SIM800 use /dev/ttyAMA0 as the communication port
    # For SIM7600E use /dev/ttyUSB2 as the communication port
    /dev/ttyAMA0
    
    # Baudrate
    115200
    
    # Assumes that your IP address is allocated dynamically by the ISP.
    noipdefault
    
    # Try to get the name server addresses from the ISP.
    usepeerdns
    
    # Use this connection as the default route to the internet.
    defaultroute
    
    # Makes PPPD "dial again" when the connection is lost.
    persist
    
    # Do not ask the remote to authenticate.
    noauth
    
    # No hardware flow control on the serial link with GSM Modem
    nocrtscts
    
    # No modem control lines with GSM Modem
    local
    #<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
    
sudo nano /etc/chatscripts/gprs
    
    #>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
    # You can use this script unmodified to connect to cellular networks.
    # The APN is specified in the peers file as the argument of the -T command
    # line option of chat(8).
    
    # For details about the AT commands involved please consult the relevant
    # standard: 3GPP TS 27.007 - AT command set for User Equipment (UE).
    # (http://www.3gpp.org/ftp/Specs/html-info/27007.htm)
    
    ABORT   BUSY
    ABORT   VOICE
    ABORT   "NO CARRIER"
    ABORT   "NO DIALTONE"
    ABORT   "NO DIAL TONE"
    ABORT   "NO ANSWER"
    ABORT   "DELAYED"
    ABORT   "ERROR"
    
    # cease if the modem is not attached to the network yet
    ABORT   "+CGATT: 0"
    
    ""  AT
    TIMEOUT 12
    OK  ATH
    OK  ATE1
    
    # +CPIN provides the SIM card PIN
    #OK "AT+CPIN=1234"
    
    # +CFUN may allow to configure the handset to limit operations to
    # GPRS/EDGE/UMTS/etc to save power, but the arguments are not standard
    # except for 1 which means "full functionality".
    #OK AT+CFUN=1
    
    OK  AT+CGDCONT=1,"IP","\T","",0,0
    OK  ATD*99#
    TIMEOUT 22
    CONNECT ""
    #<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<


#### Check the connection

    dial:
    
    sudo pon rnet
    
    check:
    tail -n 30 /var/log/syslog
    ifconfig
    
    set route:
    sudo route add default dev ppp0
    ping 8.8.8.8
    
    hang off:
    sudo poff rnet


#### Start while booting

    sudo nano /etc/network/interfaces
    >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
    # interfaces(5) file used by ifup(8) and ifdown(8)
    
    # Please note that this file is written to be used with dhcpcd
    # For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'
    
    # Include files from /etc/network/interfaces.d:
    
    auto rnet
    iface rnet inet ppp
    provider rnet
      
    source-directory /etc/network/interfaces.d
    <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
    
    sudo nano /etc/rc.local
    >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
    ...
    ...
    
    printf "Restart SIM800L\n"
    sudo echo "21" > /sys/class/gpio/export
    sudo echo "out" > /sys/class/gpio/gpio21/direction
    sudo echo "1" > /sys/class/gpio/gpio21/value
    sudo echo "0" > /sys/class/gpio/gpio21/value
    sudo echo "1" > /sys/class/gpio/gpio21/value
    
    exit 0
    <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
