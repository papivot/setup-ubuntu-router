
# Setup Router on Ubuntu (RUN THIS ENTIRE AS ROOT)

## Install pre-reqs

```console
foo@bar:~$ apt-get update
foo@bar:~$ apt-get install jq yq dnsmasq iptables-persistent -y
```

Set up a server with multiple NICs. Example - 

```console
foo@bar:~$ export WAN_NIC=ens160 # Note that this is the NIC used to talk to the outside world.
foo@bar:~$ export LAN1_NIC=ens192 # Used for Private Network #1. ens... need to be validated with correct MAC address
foo@bar:~$ export LAN2_NIC=ens224 # Used for Private Network #2. ens... need to be validated with correct MAC address
...
```

## Modify Networking
Change /etc/netplan/01...cfg to something similar

```yaml
network:
  ethernets:
    ${WAN_NIC}:
      addresses:
      - 192.168.1.51/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
        - 192.168.1.8 # IP address of DNS servers
        - 192.168.1.1
        search:
        - lab.local
    ${LAN1_NIC}:
      addresses:
      - 192.168.100.1/23
      nameservers:
        addresses:
        - 192.168.1.8
        - 192.168.1.1
        search:
        - lab.local
    ${LAN2_NIC}:
      addresses:
      - 192.168.102.1/23
      nameservers:
        addresses:
        - 192.168.1.8
        - 192.168.1.1
        search:
        - lab.local
  version: 2
```

```console
foo@bar:~$ netplan apply
```

## If planning to use multiple VLANs on one of the private NIC 
In this example LAN1_NIC - ens224 will be the native VLAN and VLAN 102 and 104 will be cofigured on the interface. Make sure that the Switch that this interface connects on allows all VLANs (4095). 

```console
foo@bar:~$ sudo apt-get install vlan
foo@bar:~$ sudo modprobe 8021q
foo@bar:~$ sudo su -c 'echo "8021q" >> /etc/modules'
```


```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens192:
      addresses:
      - 10.197.107.62/24
      gateway4: 10.197.107.253
      nameservers:
        addresses:
        - 10.142.7.21
        - 10.142.7.22
        search:
        - eng.vmware.com
    ${LAN1_NIC}:
      addresses:
      - 192.168.100.1/23
      nameservers:
        addresses:
        - 10.142.7.21
        - 10.142.7.22
        search:
        - eng.vmware.com
        - lab.local
  vlans:
    vlan102:
      id: 102
      link: ${LAN1_NIC} 
      addresses:
      - 192.168.102.1/23
      nameservers:
        addresses:
        - 10.142.7.21
        - 10.142.7.22
        search:
        - eng.vmware.com
        - lab.local
    vlan104:
      id: 104
      link: ${LAN1_NIC}
      addresses:
      - 192.168.104.1/23
      nameservers:
        addresses:
        - 10.142.7.21
        - 10.142.7.22
        search:
        - eng.vmware.com
        - lab.local
  version: 2
```

```console
foo@bar:~$ netplan apply
```

## Modify IP tables, making it persistant and configuring IP forwarding

```console
## enable iptables for NAT and forwarding
foo@bar:~$ iptables -t nat -A POSTROUTING -o ${WAN_NIC} -j MASQUERADE
foo@bar:~$ iptables -A FORWARD -i ${LAN1_NIC}  -o ${WAN_NIC}  -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${WAN_NIC}   -o ${LAN1_NIC} -m state --state RELATED,ESTABLISHED -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${LAN2_NIC}  -o ${WAN_NIC}  -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${WAN_NIC}   -o ${LAN2_NIC} -m state --state RELATED,ESTABLISHED -j ACCEPT

## If using VLANS (102 and 104 are VLAN IDs)
foo@bar:~$ iptables -A FORWARD -i ${LAN1_NIC}.102  -o ${WAN_NIC}  -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${WAN_NIC}  -o ${LAN1_NIC}.102 -m state --state RELATED,ESTABLISHED -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${LAN1_NIC}.104  -o ${WAN_NIC}  -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${WAN_NIC}  -o ${LAN1_NIC}.104 -m state --state RELATED,ESTABLISHED -j ACCEPT

## Additional DNAT rules to expose internal IP addresses to the outside world 
## E.g. Elasticsearch running on port 9200 on private IP 192.168.104.105
foo@bar:~$ iptables -t nat -A PREROUTING -p tcp --dport 9200 -j DNAT --to-destination 192.168.104.105:9200
...
...
## enable multicast routing
foo@bar:~$ iptables -A INPUT   -s 224.0.0.0/4 -j ACCEPT
foo@bar:~$ iptables -A FORWARD -s 224.0.0.0/4 -d 224.0.0.0/4 -j ACCEPT
foo@bar:~$ iptables -A OUTPUT  -d 224.0.0.0/4 -j ACCEPT
...
## persist the changes
foo@bar:~$ iptables-save > /etc/iptables/rules.v4

foo@bar:~$ ip route add 224.0.0.0/4 dev ${LAN1_NIC} #on the private nw
```

```console 
## enable IP forwarding
foo@bar:~$ cp -p /etc/sysctl.conf  /etc/sysctl.conf.bck
foo@bar:~$ sed '/net.ipv4.ip_forward=1/s/^#//' /tmp/sysctl.conf
```

```console
## disable firewall
foo@bar:~$ sudo ufw disable
```


## Disable systemd-resolved on Ubuntu (Optional)
```console
foo@bar:~$ sudo systemctl disable systemd-resolved
foo@bar:~$ sudo systemctl stop systemd-resolved
# If /etc/resolv.conf is a symbolic link to ../run/systemd/resolve/stub-resolv.conf
foo@bar:~$ sudo rm /etc/resolv.conf
```

## Setup DNS/DHCP using DNSMASQ

Values fot the configuration file /etc/dnsmasq.conf

```console
except-interface=${WAN_NIC}  # Public interface that you do not need to serve DNS/DHCP
bind-interfaces
dhcp-option=3,192.168.100.1 # E.g. DHCP on the 1st private interface
dhcp-range=192.168.100.5,192.168.100.15,255.255.255.0,12h # E.g. DHCP rage to serve
domain=env1.lab.local # E.g. DHCP domain
# Add more...
```

```console
foo@bar:~$ reboot
```
---



---
# If UI needs to be enabled - 

```console
foo@bar:~$ sudo apt-get install tasksel gdm3
foo@bar:~$ sudo tasksel install ubuntu-desktop-minimal
foo@bar:~$ sudo reboot
```
---

## Installing DHCP server  isc-dhcp-server
```
sudo apt install isc-dhcp-server
```

edit `sudo vi /etc/default/isc-dhcp-server` and modify `INTERFACESv4=""` to service the required interfaces. For e.g. `INTERFACESv4="vlan102 vlan104"`

edit `/etc/dhcp/dhcpd.conf` and modify the following - 

```
option domain-name "env1.lab.test";
option domain-name-servers 192.168.100.1;

authoritative;

subnet 192.168.102.0 netmask 255.255.254.0 {
  range 192.168.103.128 192.168.103.192;
  option domain-name-servers 192.168.100.1;
  option domain-name "env1.lab.test";
  option subnet-mask 255.255.254.0;
  option routers 192.168.102.1;
  option broadcast-address 192.168.103.255;
  option ntp-servers ntp.vmware.com;
  default-lease-time 600;
  max-lease-time 7200;
}

subnet 192.168.104.0 netmask 255.255.254.0 {
  range 192.168.105.128 192.168.105.192;
  option domain-name-servers 192.168.100.1;
  option domain-name "env1.lab.test";
  option subnet-mask 255.255.254.0;
  option routers 192.168.104.1;
  option broadcast-address 192.168.105.255;
  option ntp-servers ntp.vmware.com;
  default-lease-time 600;
  max-lease-time 7200;
}
```

restart DHCP service -
```
sudo systemctl restart isc-dhcp-server.service
```
