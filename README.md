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
### Issue root certificate
```
# Generate Root CA private key (2048 bits)
openssl genrsa -out ~/rootCA.key 2048

# Create Root CA configuration file
cat > ~/rootca.cnf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
x509_extensions = v3_ca

[ dn ]
C = US
ST = State
L = City
O = Organization
CN = Flexisip Root CA

[ v3_ca ]
basicConstraints = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF


# Generate Root CA certificate (valid for 10 years)
openssl req -x509 -new -nodes -key ~/rootCA.key -sha256 -days 3650 \
  -config ~/rootca.cnf \
  -extensions v3_ca \
  -out ~/rootCA.crt
```

### Issue server certificate
```
# Generate server private key
openssl genrsa -out ~/server.key 2048

# Create certificate signing request (CSR) configuration file
cat > ~/server.cnf << EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[dn]
C = US
ST = State
L = City
O = Organization
CN = c0a80009.nip.io

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = c0a80009.nip.io
DNS.2 = *.nip.io
IP.1 = 192.168.0.9
EOF

# Generate CSR
openssl req -new -key ~/server.key -out ~/server.csr -config ~/server.cnf

# Create extension file for signing
cat > ~/server_ext.cnf << EOF
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = c0a80009.nip.io
DNS.2 = *.nip.io
IP.1 = 192.168.0.9
EOF

# Sign the server certificate with Root CA (valid for 1 year)
openssl x509 -req -in ~/server.csr \
  -CA ~/rootCA.crt \
  -CAkey ~/rootCA.key \
  -CAcreateserial \
  -out ~/server.crt \
  -days 3640 \
  -sha256 \
  -extfile ~/server_ext.cnf
```
#### Verify certs
```
# Verify Root CA certificate
openssl x509 -in ~/rootCA.crt -text -noout

# Verify server certificate
openssl x509 -in ~/server.crt -text -noout

# Verify server certificate against Root CA
openssl verify -CAfile ~/rootCA.crt ~/server.crt

# Check SAN entries in server certificate
openssl x509 -in ~/server.crt -text -noout | grep -A 1 "Subject Alternative Name"
```
#### Change permissions on keys
```
sudo chmod 600 ~/rootCA.key
sudo chmod 600 ~/server.key
sudo chmod 644 ~/server.crt
sudo chmod 644 ~/rootCA.crt
```
#### Move certs to /etc/flexisip/tls
```
sudo mkdir -p /etc/flexisip/tls
chmod 600 /etc/flexisip/tls

cp ~/server.crt /etc/flexisip/tls
cp ~/server.key /etc/flexisip/tls
cp ~/rootCA.crt /etc/flexisip/tls
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
transports=sips:*;tls-verify-incoming=0
tls-certificates-dir=/etc/flexisip/tls
tls-certificates-file=/etc/flexisip/tls/server.crt
tls-certificates-private-key=/etc/flexisip/tls/server.key
tls-certificates-ca-file=/etc/flexisip/tls/rootCA.crt

[module::Registrar]
enabled=true
reg-domains=*

[module::Router]
enabled=true

[presence-server]
long-term-enabled=true

[module::Authentication]
enabled=true
available-algorithms=MD5
db-implementation=file
file-path=/etc/flexisip/userdb
auth-domains=*
realm=voip
```
### Automate users creation
```
export ENCRYPTION_SECRET="<SPECIFY HERE STRONG PASSWORD WHICH WOULD BE USED AS USER PASSWORDS HASHING SALT>"
export DOMAIN="<SPECIFY HERE YOUR DOMAIN>"
export REALM="voip"

for i in {1..100}; do
	export username="${i}"
	export secret=$(echo -n "${i}:${ENCRYPTION_SECRET}" | sha256sum | awk '{print $1}' | xxd -r -p | base64 -w 0 | cut -c1-7)
	export md5=$(echo -n "${username}:${REALM}:${secret}" | md5sum | awk '{print $1}')
	echo "${username}@${DOMAIN} md5:${md5} ;" | sudo tee -a /etc/flexisip/userdb
done


# get passwords of first 100 users
for i in {1..100}; do
	export username="${i}"
	export secret=$(echo -n "${i}:${ENCRYPTION_SECRET}" | sha256sum | awk '{print $1}' | xxd -r -p | base64 -w 0 | cut -c1-7)
	echo "${username} ${secret}"
done
```
Now all users added to <b>/etc/flexisip/userdb</b> file

### Restart service
```
systemctl restart flexisip-proxy flexisip-presence
```
### To debug troubles use [global:log-directory]
```
tail -f /var/opt/belledonne-communications/log/flexisip/flexisip-proxy.log
```
