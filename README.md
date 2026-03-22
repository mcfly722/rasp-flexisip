# rasp-flexisip
This is [Flexisip](https://gitlab.linphone.org/BC/public/flexisip) release builded for Raspberry PI (arm64).<br>

[Manual build process](manualBuild.md)

Compiled Flexisip deb package for arm64 you can find in [releases](https://github.com/mcfly722/rasp-flexisip/releases)<br>
Official documentation [link](https://wiki.linphone.org/xwiki/wiki/public/view/Flexisip/)

### Installation
```
sudo apt update

wget https://github.com/mcfly722/rasp-flexisip/releases/download/bc-flexisip_2.4.3/bc-flexisip_2.4.3-1_arm64.deb

sudo apt install ./bc-flexisip_2.4.3-1_arm64.deb
```
### Configure
```
mv /etc/flexisip/flexisip.conf /etc/flexisip/flexisip.conf.old
```
Replace users and their passwords
```
cat > /etc/flexisip/flexisip.conf << EOF
[server::proxy]
listen=tcp:5060
realm=flexisip.local
auth-enabled=yes

[b2bua-server::sip-bridge]
listen=tcp:5080
realm=flexisip.local

[users]
alice=password1
bob=password2

[log]
level=debug
file=/var/log/flexisip/flexisip.log
rotate=yes
max-size=10M
EOF
```