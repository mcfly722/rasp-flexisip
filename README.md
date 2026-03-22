# rasp-flexisip
This is [Flexisip](https://gitlab.linphone.org/BC/public/flexisip) release builded for Raspberry PI (arm64).<br>
### Build process
Build requires Raspberry Pi OS.
#### 1. Install lxc
```
sudo apt update
sudo apt install lxc lxc-templates -y
```
#### 2. Install lxd service and configure it
```
sudo apt install lxd -y

cat > lxd-init-config.yaml << EOF
config:
  images.auto_update_interval: "0"
networks:
- config:
    ipv4.address: 192.168.2.1/24
    ipv4.nat: "true"
    ipv6.address: none
  description: ""
  name: lxdbr0
  type: ""
  project: default
storage_pools:
- config: {}
  description: ""
  name: default
  driver: dir
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      network: lxdbr0
      type: nic
    root:
      path: /
      pool: default
      type: disk
  name: default
projects: []
cluster: null
EOF

sudo lxd init --preseed < lxd-init-config.yaml
```
#### 3. Deploy Ubuntu container for Build and shell in it
```
sudo lxc launch ubuntu:22.04/arm64 flexisip

sudo lxc shell flexisip
```
#### 4. Create the Belledonne Communications APT repository
```
cat > /etc/apt/sources.list.d/belledonne.list << EOF
# For Ubuntu 22.04 LTS
deb [arch=amd64] https://download.linphone.org/snapshots/ubuntu jammy stable # hotfix beta alpha
# For Ubuntu 24.04 LTS
deb [arch=amd64] https://download.linphone.org/snapshots/ubuntu noble stable # hotfix beta alpha
EOF
```
#### 5. Install Belledonne Communications PGP key for package signature checking
```
wget https://download.linphone.org/snapshots/ubuntu/pubkey.gpg -O - | sudo apt-key add -
```
#### 6. Install all required packages
```
apt install cmake
apt install build-essential
apt install doxygen

apt install python3-pip -y
pip3 install pystache

apt install yasm

apt install libpq-dev
apt install libsqlite3-dev
apt install libspeexdsp-dev libspeex-dev libopus-dev libgsm1-dev
apt install libxerces-c-dev
apt install libxml2-dev
apt install libnghttp2-dev
apt install libsnmp-dev snmp
apt install libmysqlclient-dev
apt install libjsoncpp-dev
apt install libgtest-dev
apt install libmariadb-dev libmariadb-dev-compat

apt install nlohmann-json3-dev
```
#### 7. Build and install hireddis
```
cd ~/
git clone https://github.com/redis/hiredis.git
cd ~/hiredis

# Checkout latest stable (>=1.1)
git checkout v1.1.0

# build and install
cmake -S . -B build
cmake --build build
cmake --install build

# symlink for cmake
sudo ln -s /usr/local/include/hiredis /usr/include/hiredis
```
#### 8. Build and install cpp-jwt
```
cd ~/
git clone https://github.com/arun11299/cpp-jwt.git
cd ~/cpp-jwt
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/local ..
make
sudo make install
```
#### 9. Build flexisip release
```
git clone https://gitlab.linphone.org/BC/public/flexisip --recursive -b 2.4.3

cd ~/flexisip

cmake ./build \
  -DCMAKE_INSTALL_PREFIX=/opt/belledonne-communications \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DSYSCONF_INSTALL_DIR=/etc \
  -DFLEXISIP_SYSTEMD_INSTALL_DIR=/usr/lib/systemd/system \
  -DCPACK_GENERATOR=DEB

make -C ./build -j$(nproc) package
```
After this steps there deb package would be in ~/flexisip folder.
