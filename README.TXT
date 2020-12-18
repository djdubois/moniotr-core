=====================
Mon(IoT)r IoT TESTBED
=====================

0) You can find example configuration files for netplan, hostapd (in case of use Wi-Fi access point),
isc-dhcp-server, iptables-save, and sysctl in the examples directory (or under examples/nuc, if using Intel NUC).

1) Copy the content of this directory to any location, for example on /opt/moniotr

2) Install the following Ubuntu dependences:
apt install python3 python3-pip isc-dhcp-server tcpdump bridge-utils

3) Install the following python dependence and modification (please do NOT use a different version of mitmproxy):
pip3 install mitmproxy==4.0.4
cp lib/server.py /usr/local/lib/python3.6/dist-packages/mitmproxy/proxy/

4) Configure app-armor in such a way that dhcpd and tcpdump are excluded:
mkdir /etc/apparmor.d/disable
ln -s /etc/apparmor.d/usr.sbin.dhcpd /etc/apparmor.d/disable/usr.sbin.dhcpd
ln -s /etc/apparmor.d/usr.sbin.tcpdump /etc/apparmor.d/disable/
service apparmor restart

5) Configure netplan and isc-dhcp-server for the network interfaces that will host the monitored devices.

For each monitored network, add to dhcpd.conf the following lines inside the "subnet" block:

on commit {
    set clip = binary-to-ascii(10, 8, ".", leased-address);
    set clhw = binary-to-ascii(16, 8, ":", substring(hardware, 1, 6));
    execute("/opt/moniotr/bin/setup-device", clip, clhw);
}

If Mon(IoT)r was not installed in /opt/moniotr, replace /opt/moniotr with the installation directory.

Finally, restart isc-dhcp-server:
service isc-dhcp-server restart

6) Add root privileges to isc-dhcp-server for executing setup-device by using the "visudo" command as root.
Add to the end of the file the following line:

dhcpd ALL=(ALL) NOPASSWD: ALL

7) Copy etc/moniotr.conf.example to etc/moniotr.conf and edit its values according to the local configuration
of the system. You can find explanations inside the file.

8) Configure IP forward by typing this command: sysctl -w net.ipv4.ip_forward=1

Then, make the change persistent by adding/uncommenting the following line in /etc/sysctl.conf
net.ipv4.ip_forward=1

9) Configure firewall using iptables and save it using iptables-persistent
apt-get install iptables-persistent

As an alternative use the rules.v4 file (in examples/iptables), copy it to /etc/iptables, and load it with:
iptables-restore < /etc/iptables/rules.v4

10) You can start Mon(IoT)r using:

/opt/moniotr/bin/moniotr-ctrl start

The same command can be added to /etc/rc.local to activate Mon(IoT)r at startup
If Mon(IoT)r was not installed in /opt/moniotr, replace /opt/moniotr with the installation directory.

====================================
APPENDIX 1. Configuring DNS override
====================================

1) Stop Mon(IoT)r (if active) using:
/opt/moniotr/bin/moniotr-ctrl stop

2) Edit moniotr.conf and enable the DNS_OVERRIDE configuration option:
DNS_OVERRIDE="true"

3) Install dnsmasq and ipset
apt-get install dnsmasq ipset

4) Disable systemwide dnsmasq
systemctl stop dnsmasq
systemctl disable dnsmasq
systemctl mask dnsmasq

5) Configure ip forwarding for localhost using this command: sysctl -w net.ipv4.conf.all.route_localnet=1

Then, make the change persistent by adding/uncommenting the following line in /etc/sysctl.conf
net.ipv4.conf.all.route_localnet=1

6) Edit the file /opt/moniotr/etc/resolv.conf with the default DNS to be used when DNS_OVERRIDE is enabled

7) (optional) If upgrading from an older version, remove bind9 and related files
apt-get purge bind9
rm -rf /opt/moniotr/var

8) Start Mon(IoT)r using:
/opt/moniotr/bin/moniotr-ctrl start

NOTE: DNS override will replace whatever DNS is specified on the DHCP configuration (using /opt/moniotr/etc/resolv.conf as default).
      DNS override is incompatible with policy-based routing (WAN_INTERFACES), for example when using Mon(IoT)r with VPN.

================================================
APPENDIX 2. Adding regex support to DNS override
================================================

1) Check if file /opt/moniotr/bin/dnsmasq.x86_64 works by typing
 /opt/moniotr/bin/dnsmasq.x86_64 --version

If there is an error you have to download dnsmasq, add the dnsmasq patch, and recompile by
following this tutorial: https://github.com/lixingcong/dnsmasq-regex

2) Stop Mon(IoT)r using:
/opt/moniotr/bin/moniotr-ctrl stop

3) Edit /etc/moniotr.conf and decomment the DNSMASQ option, adding the full path of
   the executable for the regex version of dnsmasq.
   If the step 1 was successful you can use:

   DNSMASQ="$MONIOTR_DIR/bin/dnsmasq.x86_64""

   Otherwise, use the path of the newly compiled dnsmasq version.

4) Start Mon(IoT)r using:
/opt/moniotr/bin/moniotr-ctrl start

5) Examples of regex to add as destinations using the "dns-override add" command are:

   :^n\.netnet\.net$: -> for a destination that is exactly 'n.netnet.net'
   :.com: -> for a destination that contains 'com' preceded by any character.
             For example "a.como" "acomo"

   The regex much be between two colons ':' to be evaluated as a regex.
   Regex will have priority over all other destination rules.

   NOTE: use '' when using special characters such as * to prevent bash from
                expanding them.
