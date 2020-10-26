# Setup Router on Ubuntu

Eaxmple - 
ens160 is WAN

enx192 is LAN01

ens224 is LAN02
...

## Disable systemd-resolved on Ubuntu
```shell
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
# If /etc/resolv.conf is a symbolic link to ../run/systemd/resolve/stub-resolv.conf
sudo rm /etc/resolv.conf
```

## Modify Networking
Change /etc/netplan/01...cfg to something similar
```yaml


``

```shell
sudo netplan apply
```

## Modify IP tables

```shell
iptables -t nat -A POSTROUTING -o ens160 -j MASQUERADE
iptables -A FORWARD -i ens192  -o ens160 -j ACCEPT
iptables -A FORWARD -i ens160  -o ens192 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i ens224  -o ens160 -j ACCEPT
iptables -A FORWARD -i ens160  -o ens224 -m state --state RELATED,ESTABLISHED -j ACCEPT
...
```

## Enable IP forwarding 

```shell 
sudo vim /etc/sysctl.conf 
# Modify file to uncomment net.ipv4.ip_forward=1
sudo reboot
```

```

