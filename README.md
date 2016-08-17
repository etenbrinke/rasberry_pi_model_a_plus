Setup my Rasberry PI model A+ setup as tomcat server
----------------------------------------------------

###Hardware
Rasberry PI model A+
WiFi-stick EDIMAX EW-7811Un
2A micro USB Power adapter (note you need 2A to allow the WiFi-stick to work stable)

###Raspbian OS
Used RASPBIAN JESSIE LITE installed on a 32Gb SD card
https://www.raspberrypi.org/downloads/raspbian/

###Setup Raspbian
Setup any config if neccesary (like your Locale, Timezone and Keyboard Layout)
```
$ sudo raspi-config
```
###Setup vi as default editor
Add to /etc/bash.bashrc
```
export EDITOR=vi
```
###Setup WiFi
Make sure you see your EDIMAX usb adapter
```
$ sudo lsusb
Bus 001 Device 003: ID 7392:7811 Edimax Technology Co., Ltd EW-7811Un 802.11n Wireless Adapter [Realtek RTL8188CUS]
```
Check your WiFi network (example interface wlan0)
```
$ sudo iwlist wlan0 scan
```
Change IP numbers to yours. Here we use security options WPA2-PSK [AES] and static IPs

/etc/network/interfaces :
```
auto lo
iface lo inet loopback

allow-hotplug wlan0
iface wlan0 inet manual
wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
iface my_wifi_network inet static
address 10.20.30.11
netmask 255.255.255.0
gateway 10.20.30.1
broadcast 10.20.30.255
wireless-power off
dns-nameservers 10.20.30.1 8.8.8.8 8.8.4.4
```
'wireless-power off' disables power management, if you're Pi should be connected 24x7 you might need this

You can also ping your gateway every minute by adding to your crontab:
```
*/1 * * * * ping -c 1 10.20.30.1 >/dev/null 2>&1
```

/etc/wpa_supplicant/wpa_supplicant.conf :
```
country=NL
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
  ssid="your_ssid"
  scan_ssid=1
  proto=RSN
  key_mgmt=WPA-PSK
  pairwise=CCMP
  group=CCMP
  psk="your_password"
  auth_alg=OPEN
  id_str="my_wifi_network"
}
```

Check your wlan0 adapter
```
$ sudo iwconfig
wlan0     IEEE 802.11bg  ESSID:"your_ssid"  Nickname:"<WIFI@REALTEK>"
          Mode:Managed  Frequency:2.417 GHz  Access Point: --:--:--:--:--:--
          Sensitivity:0/0
          Retry:off   RTS thr:off   Fragment thr:off
          Encryption key:****-****-****-****-****-****-****-****   Security mode:open
          Power Management:off
          Link Quality=100/100  Signal level=93/100  Noise level=0/100
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:0   Missed beacon:0
```

Check your wlan IP settings
```
$ sudo ifconfig wlan0
wlan0     Link encap:Ethernet  HWaddr --:--:--:--:--:--
          inet addr:10.20.30.11  Bcast:10.20.30.255  Mask:255.255.255.0
          inet6 addr: ----::---:----:----:----/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:74952 errors:0 dropped:157 overruns:0 frame:0
          TX packets:30704 errors:0 dropped:1 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:105105060 (100.2 MiB)  TX bytes:3308546 (3.1 MiB)
```

Because here we use static IP numbers above we don't need our dhcp server to involve, disable it with:
```
$ sudo update-rc.d dhcpcd disable
```

###Updating and upgrading Raspbian
your need your network to be working
```
$ sudo apt-get update
$ sudo apt-get dist-upgrade
```
###Install java
```
$ sudo apt-get install oracle-java8-jdk
$ java -version
```

###Install tomcat
```
$ sudo adduser \
  --system \
  --shell /bin/bash \
  --gecos 'Tomcat Java Servlet and JSP engine' \
  --group \
  --disabled-password \
  --home /home/tomcat \
  tomcat
  
$ mkdir -p ~/tmp
$ cd ~/tmp
$ wget http://www.us.apache.org/dist/tomcat/tomcat-8/v8.5.8/bin/apache-tomcat-8.5.8.tar.gz
$ tar xvzf ./apache-tomcat-8.5.8.tar.gz
$ sudo mv ~/tmp/apache-tomcat-8.5.8 /usr/share/tomcat
$ sudo chown -R tomcat:tomcat /usr/share/tomcat
$ sudo -s
$ chmod +x /usr/share/tomcat/bin/*.sh
```

Start tomcat
```
$ sudo /bin/su - tomcat -c /usr/share/tomcat/bin/startup.sh
```

To start Tomcat automatically after a reboot save this script in /etc/init.d/tomcat
```
#!/bin/bash

### BEGIN INIT INFO
# Provides:        tomcat
# Required-Start:  $network
# Required-Stop:   $network
# Default-Start:   2 3 4 5
# Default-Stop:    0 1 6
# Short-Description: Start/Stop Tomcat server
### END INIT INFO

PATH=/sbin:/bin:/usr/sbin:/usr/bin

start() {
 /bin/su - tomcat -c 'rm -rf /usr/share/tomcat/temp/*'
 /bin/su - tomcat -c 'rm -rf /usr/share/tomcat/work/*'
 /bin/su - tomcat -c /usr/share/tomcat/bin/startup.sh
}

stop() {
 /bin/su - tomcat -c /usr/share/tomcat/bin/shutdown.sh 
}

case $1 in
  start|stop) $1;;
  restart) stop; start;;
  *) echo "Run as $0 <start|stop|restart>"; exit 1;;
esac
```

change the permissions and correct symlinks automatically:
```
$ sudo chmod 755 /etc/init.d/tomcat
$ sudo update-rc.d tomcat defaults
```

###Test it
Start a browser in your network and open te URL: http://10.20.30.11:8080/

###Enable management interface
If this application server is 100% isolated from the internet, right?, and you want to open up the management interface you can add a admin account and assign manager and admin roles to it.
Create an admin account called for example "admin" who’s password is "tomcat" (choose a real password here) in the /usr/share/tomcat/conf/tomcat-users.xml
```
<role rolename="manager-gui"/>
<role rolename="admin-gui"/>
<role rolename="manager-status"/>
<role rolename="manager-script"/>
<user username="admin" password="admin" roles="admin-gui,manager-gui,manager-status,manager-script"/>
```
Default you can access the management interface only from the machine where your tomcat is running.
If you want to access it from another machine in the same network where your tomcat is running uncomment the restriction in /usr/share/tomcat/webapps/manager/META-INF/context.xml and /usr/share/tomcat/webapps/host-manager/META-INF/context.xml like:
```
<Context antiResourceLocking="false" privileged="true" >
<!--
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="10\.20\.30\.\d+" />
-->
</Context>
```
Restart tomcat
```
sudo /etc/init.d/tomcat restart
```
Now you can visit:

http://10.20.30.11:8080/manager/status

http://10.20.30.11:8080/manager/html

http://10.20.30.11:8080/host-manager/html

But also http://10.20.30.11:8080/manager/text will work in case you want to use it for deployments

###Deploy WAR with Cargo Maven

If you use maven to build your java project as WAR file you can deploy it to your tomcat server with the Cargo Maven plugin.
In this example tomcat is running at 10.20.30.11 and the machine we deploy from is somewhere in the same network.
    
Enable cargo maven plugin in your project pom.xml:

```
<plugin>
  <groupId>org.codehaus.cargo</groupId>
  <artifactId>cargo-maven2-plugin</artifactId>
  <version>1.6.1</version>
  <configuration>
    <container>
      <containerId>tomcat8x</containerId>
       <type>remote</type>
    </container>
    <configuration>
      <type>runtime</type>
      <properties>
        <cargo.hostname>10.20.30.11</cargo.hostname>
        <cargo.servlet.port>8080</cargo.servlet.port>
        <cargo.remote.username>admin</cargo.remote.username>
        <cargo.remote.password>tomcat</cargo.remote.password>
        <cargo.tomcat.manager.url>http://10.20.30.11:8080/manager/text</cargo.tomcat.manager.url>
      </properties>
    </configuration>
    <deployables>
      <deployable>
        <groupId>${project.groupId}</groupId>
        <artifactId>${project.artifactId}</artifactId>
        <type>war</type>
        <properties>
          <context>/myApp</context>
        </properties>
      </deployable>
    </deployables>
  </configuration>
</plugin>
```

Maven will deploy the WAR file to Tomcat server using “http://10.20.30.11:8080/manager/text”. For this you need to give the admin user the manager-script role as described before.
Now we can deploy to tomcat with:

```
mvn cargo:deploy
mvn cargo:undeploy
mvn cargo:redeploy
```
