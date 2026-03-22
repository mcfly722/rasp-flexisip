# rasp-flexisip
This is [Flexisip](https://gitlab.linphone.org/BC/public/flexisip) release builded for Raspberry PI (arm64).<br>

[Manual build process](manualBuild.md)

Compiled Flexisip deb package for arm64 you can find in [releases](https://github.com/mcfly722/rasp-flexisip/releases)<br>
Official documentation [link](https://wiki.linphone.org/xwiki/wiki/public/view/Flexisip/)<br>
Configuration [reference for 2.4.3](https://wiki.linphone.org/xwiki/wiki/public/view/Flexisip/A.%20Configuration%20Reference%20Guide/2.4.3/) version<br>

### Install flexisip service
```
sudo apt update

wget https://github.com/mcfly722/rasp-flexisip/releases/download/bc-flexisip_2.4.3/bc-flexisip_2.4.3-1_arm64.deb

sudo apt install ./bc-flexisip_2.4.3-1_arm64.deb
```

### Install libhiredis.so
```
sudo apt install libhiredis-dev

sudo ln -s /usr/lib/aarch64-linux-gnu/libhiredis.so.0.14 /usr/lib/aarch64-linux-gnu/libhiredis.so.1.1.0
sudo ldconfig

```
### Configure
```
# save original config
mv /etc/flexisip/flexisip.conf /etc/flexisip/flexisip.conf.old

# create empty users db
cat > /etc/flexisip/userdb << EOF
version:1

EOF

chmod 600 /etc/flexisip/userdb
```
# Service config
```
cat > /etc/flexisip/flexisip.conf << EOF
[global]
require-peer-certificate=false
log-level=debug
user-errors-logs=true

[module::Registrar]
enabled=true
reg-domains=c0a80009.nip.io

[presence-server]
long-term-enabled=true

[module::Authentication]
enabled=true
available-algorithms=MD5
db-implementation=file
file-path=/etc/flexisip/userdb
auth-domains=c0a80009.nip.io
realm=c0a80009.nip.io
EOF
```

### Add users
```
export username="user1"
export domain="c0a80009.nip.io"
export password="secret"

echo "${username}@${domain} clrtxt:${password} ;" >> /etc/flexisip/userdb
```

### Restart service
```
systemctl restart flexisip-proxy flexisip-presence
```