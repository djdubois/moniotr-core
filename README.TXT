==========================
Mon(IoT)r IoT TESTBED v1.0
==========================

PREREQUISITES

This installation guide assumes the following:

- You run an Ubuntu Server 20.04 Linux distribution
- You have a working network configuration with at least two
  network interfaces, one for the Internet, and one for the
  IoT devices
- The network interface for the IoT devices must be NATTed to the
  Internet using iptables Masquerade rules
- The network interface for the IoT devices must have a DHCP
  server running assigning local IPv4 address to them
- Mon(IoT)r requires the use of the ISC DHCP SERVER as a DHCP
  server. DNAMASQ or others are not supported at this time.
- You can find example configuration files in the examples directory.
  Examples include: isc-dhcp-server configuration, iptables-save,
  sysctl, etc.


INSTALLATION STEPS

1) Copy the content of this directory to any location and than change directory to that location. 
   We recommend /opt/moniotr

2) Install the following Ubuntu dependences:
apt install python3 python3-pip isc-dhcp-server tcpdump bridge-utils

3) Install the following python dependence and modification (please do NOT use a different version of mitmproxy):
pip3 install mitmproxy==4.0.4
cp lib/server.py /usr/local/lib/python3.8/dist-packages/mitmproxy/proxy/

4) Configure app-armor in such a way that dhcpd and tcpdump are excluded:
mkdir -p /etc/apparmor.d/disable
ln -s /etc/apparmor.d/usr.sbin.dhcpd /etc/apparmor.d/disable/usr.sbin.dhcpd
ln -s /etc/apparmor.d/usr.sbin.tcpdump /etc/apparmor.d/disable/
service apparmor restart

5) Configure isc-dhcp-server for the network interfaces that will host the monitored devices.

For each monitored network, add to dhcpd.conf the following lines inside the "subnet" block:

on commit {
    set clip = binary-to-ascii(10, 8, ".", leased-address);
    set clhw = binary-to-ascii(16, 8, ":", substring(hardware, 1, 6));
    execute("/opt/moniotr/bin/setup-device", clip, clhw);
}

If Mon(IoT)r was not installed in /opt/moniotr, replace /opt/moniotr with the installation directory.

Note: this assumes you have already configured a DHCP server on your monitored interfaces. If not,
have a look at examples/dhcp/dhcpd.conf as inspiration for your /etc/dhcp/dhcpd.conf

Finally, restart isc-dhcp-server:
service isc-dhcp-server restart

6) Add root privileges to isc-dhcp-server for executing setup-device by using the "visudo" command as root.
Add to the end of the file the following line:

dhcpd ALL=(ALL) NOPASSWD: ALL

Note: this may lower the security of your system. Non-root execution is not supported yet, but will be
      added in the future.

7) Copy etc/moniotr.conf.example to etc/moniotr.conf and edit its values according to the local configuration
of the system. You can find explanations inside the file.

8) Copy etc/devices.txt.example to etc/devices.txt, this file should contain a list of mac addresses followed
by the device names (feel free to edit the example).

9) Configure IP forward by typing this command: sysctl -w net.ipv4.ip_forward=1

Then, make the change persistent by adding/uncommenting the following line in /etc/sysctl.conf
net.ipv4.ip_forward=1

Note: if you had a working NAT configuration, this step is not necessary because it is needed as part
of a NAT configuration.

10) Configure firewall using iptables and save it using iptables-persistent
apt-get install iptables-persistent

Note: if you had a working NAT configuration, this step is not necessary because it is needed as part
of a NAT configuration. In any case, iptables-persistent will save your NAT configuration and apply
it every time the system boots.

11) You can start Mon(IoT)r using:

/opt/moniotr/bin/moniotr-ctrl start

The same command can be added to /etc/rc.local to activate Mon(IoT)r at startup.

If you have no rc.local configured, follow this guide to activate it: 
https://linuxmedium.com/how-to-enable-etc-rc-local-with-systemd-on-ubuntu-20-04/

==================================
OPTIONAL: Configuring DNS override
==================================

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

7) Start Mon(IoT)r using:
/opt/moniotr/bin/moniotr-ctrl start

NOTE: DNS override will replace whatever DNS is specified on the DHCP configuration (using /opt/moniotr/etc/resolv.conf as default).
      DNS override is incompatible with policy-based routing (WAN_INTERFACES), for example when using Mon(IoT)r with VPN.



===============================================================
OPTIONAL ADDITIONAL STEPS: Adding regex support to DNS override
===============================================================

8) Check if file /opt/moniotr/bin/dnsmasq.x86_64 works by typing
 /opt/moniotr/bin/dnsmasq.x86_64 --version

If there is an error you have to download dnsmasq, add the dnsmasq patch, and recompile by
following this tutorial: https://github.com/lixingcong/dnsmasq-regex

9) Stop Mon(IoT)r using:
/opt/moniotr/bin/moniotr-ctrl stop

10) Edit /etc/moniotr.conf and decomment the DNSMASQ option, adding the full path of
   the executable for the regex version of dnsmasq.
   If the step 1 was successful you can use:

   DNSMASQ="$MONIOTR_DIR/bin/dnsmasq.x86_64""

   Otherwise, use the path of the newly compiled dnsmasq version.

11) Start Mon(IoT)r using:
/opt/moniotr/bin/moniotr-ctrl start

12) Examples of regex to add as destinations using the "dns-override add" command are:

   :^n\.netnet\.net$: -> for a destination that is exactly 'n.netnet.net'
   :.com: -> for a destination that contains 'com' preceded by any character.
             For example "a.como" "acomo"

   The regex much be between two colons ':' to be evaluated as a regex.
   Regex will have priority over all other destination rules.

   NOTE: use '' when using special characters such as * to prevent bash from
                expanding them.
