
# If UI needs to be enabled - 

```console
sudo apt-get install tasksel gdm3
sudo tasksel install ubuntu-desktop-minimal
sudo reboot
```

# Setup Router on Ubuntu (RUN THIS ENTIRE AS ROOT)

## Install pre-reqs

```console
$ apt-get update
$ apt-get install jq yq dnsmasq iptables-persistent -y
```

Set up a server with multiple NICs. Example - 

```shell
export WAN_NIC=ens160 # Note that this is the NIC used to talk to the outside world.
export LAN1_NIC=ens192 # Used for Private Network #1. ens... need to be validated with correct MAC address
export LAN2_NIC=ens224 # Used for Private Network #2. ens... need to be validated with correct MAC address
...
```

## Disable systemd-resolved on Ubuntu (Option 1)
```shell
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
# If /etc/resolv.conf is a symbolic link to ../run/systemd/resolve/stub-resolv.conf
sudo rm /etc/resolv.conf
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

```shell
# netplan apply
```

## Modify IP tables and make it persistant

```shell
# iptables -t nat -A POSTROUTING -o ${WAN_NIC} -j MASQUERADE
# iptables -A FORWARD -i ${LAN1_NIC}  -o ${WAN_NIC}  -j ACCEPT
# iptables -A FORWARD -i ${WAN_NIC}   -o ${LAN1_NIC} -m state --state RELATED,ESTABLISHED -j ACCEPT
# iptables -A FORWARD -i ${LAN2_NIC}  -o ${WAN_NIC}  -j ACCEPT
# iptables -A FORWARD -i ${WAN_NIC}   -o ${LAN2_NIC} -m state --state RELATED,ESTABLISHED -j ACCEPT
...
...
## enable multicast routing
# iptables -A INPUT   -s 224.0.0.0/4 -j ACCEPT
# iptables -A FORWARD -s 224.0.0.0/4 -d 224.0.0.0/4 -j ACCEPT
# iptables -A OUTPUT  -d 224.0.0.0/4 -j ACCEPT
...
# iptables-save > /etc/iptables/rules.v4

# ip route add 224.0.0.0/4 dev ${LAN1_NIC} #on the private nw
```

## Enable IP forwarding 

```shell 
# cp -p /etc/sysctl.conf  /etc/sysctl.conf.bck
# sed '/net.ipv4.ip_forward=1/s/^#//' /tmp/sysctl.conf
```

## Install DNS/DHCP using DNSMASQ

Values fot the configuration file /etc/dnsmasq.conf

```shell
except-interface=${WAN_NIC}  # Public interface that you do not need to serve DNS/DHCP
bind-interfaces
dhcp-option=3,192.168.100.1 # E.g. DHCP on the 1st private interface
dhcp-range=192.168.100.5,192.168.100.15,255.255.255.0,12h # E.g. DHCP rage to serve
domain=env1.lab.local # E.g. DHCP domain
# Add more...
```

```shell
# reboot
```
