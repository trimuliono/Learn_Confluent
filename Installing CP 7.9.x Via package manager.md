# Installing CP 7.9.x via package manager on debian/ubuntu (wsl2 - ubuntu 24)

1. Make a directory to store the Confluent public key used to sign Confluent packages in the APT repository.
```
sudo mkdir -p /etc/apt/keyrings
```
2. Download and install the Confluent public key.
```
wget -qO - https://packages.confluent.io/deb/7.9/archive.key | gpg \
--dearmor | sudo tee /etc/apt/keyrings/confluent.gpg > /dev/null
```
3. Add the Confluent repository to /etc/apt/sources.list.d referencing the location of the signing key.
```
CP_DIST=$(lsb_release -cs)
echo "Types: deb
URIs: https://packages.confluent.io/deb/7.9
Suites: stable
Components: main
Architectures: $(dpkg --print-architecture)
Signed-by: /etc/apt/keyrings/confluent.gpg

Types: deb
URIs: https://packages.confluent.io/clients/deb/
Suites: ${CP_DIST}
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/confluent.gpg" | sudo tee /etc/apt/sources.list.d/confluent-platform.sources > /dev/null
```
4. Update apt-get and install the entire Confluent Platform package.
- Confluent Platform:
  ```
  sudo apt-get update && sudo apt-get install confluent-platform
  ```
5. 














Source: https://docs.confluent.io/platform/7.9/installation/installing_cp/deb-ubuntu.html#systemd-ubuntu-debian-install
