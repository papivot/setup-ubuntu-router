
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

## Disable systemd-resolved on Ubuntu (Optional)
```console
foo@bar:~$ sudo systemctl disable systemd-resolved
foo@bar:~$ sudo systemctl stop systemd-resolved
# If /etc/resolv.conf is a symbolic link to ../run/systemd/resolve/stub-resolv.conf
foo@bar:~$ sudo rm /etc/resolv.conf
```

## Modify Networking
Change /etc/netplan/01...cfg to something similar

```yaml
network:
  ethernets:
    ens160:
      addresses:
      - 192.168.1.51/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
        - 192.168.1.8 # IP address of DNS servers
        - 192.168.1.1
        search:
        - lab.local
    ens192:
      addresses:
      - 192.168.10.1/23
      nameservers:
        addresses:
        - 192.168.1.8
        - 192.168.1.1
        search:
        - lab.local
    ens224:
      addresses:
      - 192.168.12.1/23
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

## Modify IP tables, making it persistant and configuring IP forwarding

```console
## enable iptables for NAT and forwarding
foo@bar:~$ iptables -t nat -A POSTROUTING -o ${WAN_NIC} -j MASQUERADE
foo@bar:~$ iptables -A FORWARD -i ${LAN1_NIC}  -o ${WAN_NIC}  -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${WAN_NIC}   -o ${LAN1_NIC} -m state --state RELATED,ESTABLISHED -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${LAN2_NIC}  -o ${WAN_NIC}  -j ACCEPT
foo@bar:~$ iptables -A FORWARD -i ${WAN_NIC}   -o ${LAN2_NIC} -m state --state RELATED,ESTABLISHED -j ACCEPT
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
# If UI needs to be enabled - 

```console
foo@bar:~$ sudo apt-get install tasksel gdm3
foo@bar:~$ sudo tasksel install ubuntu-desktop-minimal
foo@bar:~$ sudo reboot
```

