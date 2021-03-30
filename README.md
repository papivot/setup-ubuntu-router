
# If UI needs to be enabled - 

```
sudo apt-get install tasksel
sudo apt-get install gdm3
sudo tasksel
# pick the Desktop minimal
sudo reboot
```

# Setup Router on Ubuntu

Set up a server with multiple NICs. Example - 

ens160 is WAN
ens192 is LAN01
ens224 is LAN02
...

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
sudo netplan apply
```

## Modify IP tables and make it persistant

```shell
sudo apt-get install iptables-persistent
sudo iptables -t nat -A POSTROUTING -o ens160 -j MASQUERADE
sudo iptables -A FORWARD -i ens192  -o ens160 -j ACCEPT
sudo iptables -A FORWARD -i ens160  -o ens192 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i ens224  -o ens160 -j ACCEPT
sudo iptables -A FORWARD -i ens160  -o ens224 -m state --state RELATED,ESTABLISHED -j ACCEPT
# enable multicast routing
iptables -A INPUT   -s 224.0.0.0/4 -j ACCEPT
iptables -A FORWARD -s 224.0.0.0/4 -d 224.0.0.0/4 -j ACCEPT
iptables -A OUTPUT  -d 224.0.0.0/4 -j ACCEPT
...
sudo su - 
iptables-save > /etc/iptables/rules.v4

ip route add 224.0.0.0/4 dev ens192 #on the private nw

```

## Enable IP forwarding 

```shell 
sudo vim /etc/sysctl.conf 
# Modify file to uncomment net.ipv4.ip_forward=1
sudo reboot
```

## Install DNS/DHCP using DNSMASQ

Values fot the configuration file /etc/dnsmasq.conf

```shell
except-interface=ens192  # Public interface that you do not need to serve DNS/DHCP
bind-interfaces
dhcp-option=3,192.168.100.1 # E.g. DHCP on the 1st private interface
dhcp-range=192.168.100.5,192.168.100.15,255.255.255.0,12h # E.g. DHCP rage to serve
domain=env1.lab.local # E.g. DHCP domain
# Add more...
```
