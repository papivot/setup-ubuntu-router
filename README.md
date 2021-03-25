
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
...
sudo su - 
iptables-save > /etc/iptables/rules.v4

```

## Enable IP forwarding 

```shell 
sudo vim /etc/sysctl.conf 
# Modify file to uncomment net.ipv4.ip_forward=1
sudo reboot
```

## Install DNS/DHCP using DNSMASQ

```shell
sudo apt-get install dnsmasq
vi /etc/dnsmasq.conf
uncomment bind-interface
sudo systemctl restart dnsmasq
# Add more...
```
