# setup-ubuntu-router

## Disable systemd-resolved on Ubuntu
```shell
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
# If /etc/resolv.conf is a symbolic link to ../run/systemd/resolve/stub-resolv.conf
sudo rm /etc/resolv.conf
```
