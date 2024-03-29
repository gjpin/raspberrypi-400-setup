This 'guide' assumes that the desktop of Raspberry Pi OS is being used.
Tested on a Raspberry Pi 400.

# Base setup
To login to custom SSH port use: ssh pi@192.168.1.xx -p XXXXX

- Pi's home folder has more restricted permissions
- Sudo requires password
- UFW with incoming traffic blocked, except the custom SSH port
- SSH with custom port and password auth disabled
- fail2ban
- Apparmor (to be used with Docker profiles)
- 256MB of memory reserved for GPU

```
# Copy SSH public key to RPi (run ip -a to confirm IP)
sudo systemctl start ssh
[run from client] ssh-copy-id pi@192.168.1.xx

# Change pi password
sudo passwd pi

# Update Raspberry Pi OS
sudo apt update
sudo apt full-upgrade
sudo reboot

# Restrict default user's home directory so that other users/groups cannot read pi's home folder
sudo chmod 750 /home/*
sudo sed -i s/DIR_MODE=0755/DIR_MODE=0750/ /etc/adduser.conf
echo "HOME_MODE 0750" | sudo tee -a /etc/login.defs

# Make sudo require a password
sudo sed -i 's/NOPASSWD/PASSWD/g' /etc/sudoers.d/010_pi-nopasswd

# Disable SSH login with password, change SSH port and enable the service
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
sudo sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config
sudo sed -i 's/#Port 22/Port XXXXX/g' /etc/ssh/sshd_config
sudo systemctl enable ssh
sudo systemctl restart ssh

# Install and setup firewall (UFW)
sudo apt -y install ufw
sudo systemctl enable ufw
sudo systemctl start ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit XXXXX/tcp comment 'SSH port rate limit'
sudo ufw enable

# Setup fail2ban
sudo apt -y install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo tee -a /etc/fail2ban/jail.local << EOF
[ssh]
enabled  = true
port     = XXXXX
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime = -1
EOF

# Install apparmor and enable it
sudo apt -y install apparmor
sudo sed -i 's/rootwait/rootwait lsm=apparmor/g' /boot/cmdline.txt

# Setup automatic updates
sudo apt-get install -y unattended-upgrades apt-listchanges
sudo dpkg-reconfigure -plow unattended-upgrades

# Reserve 256MB of memory for GPU
# 4K 60Hz ref: https://www.raspberrypi.org/documentation/configuration/hdmi-config.md
# Memory reserve ref: https://www.raspberrypi.org/documentation/configuration/config-txt/memory.md
# HDMI configurations: https://www.raspberrypi.org/documentation/configuration/config-txt/video.md
sudo tee -a /boot/config.txt << EOF

gpu_mem=256
EOF

# Disabling SAP plugin (bluetooth)
sudo sed -i 's/bluetooth\/bluetoothd/bluetooth\/bluetoothd --noplugin=sap/g' /etc/systemd/system/bluetooth.target.wants/bluetooth.service
sudo systemctl daemon-reload
sudo service bluetooth restart

# Power on bluetooth on startup (should be the case by default, but it's not working)
sudo sed -i '/exit/d' /etc/rc.local
sudo tee -a /etc/rc.local << EOF
hciconfig hci0 up
systemctl restart bluetooth

exit 0
EOF
```

# Syncthing
```
sudo apt -y install syncthing
sudo systemctl enable syncthing@pi.service
sudo systemctl start syncthing@pi.service
sudo ufw allow from 192.168.1.0/24 to any port 22000 proto tcp comment 'syncthing'
sudo ufw allow from 192.168.1.0/24 to any port 21027 proto udp comment 'syncthing'
```

# Steam Link
```
sudo apt -y install steamlink
sudo tee -a /etc/xdg/lxsession/LXDE-pi/autostart << EOF
@steamlink
EOF
```

# Docker
```
# Install docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm get-docker.sh

# Manage Docker as a non-root user
sudo groupadd docker
sudo usermod -aG docker pi
newgrp docker

# Test Docker installation
docker run hello-world

# Install docker-compose
sudo apt -y install libffi-dev 
sudo pip3 install docker-compose

# Create new bridge network
docker network create web

# Allow incoming traffic on ports 80 and 443
sudo ufw allow 443/tcp
sudo ufw allow 80/tcp
```

# Misc
## Bluetooth connections
```
bluetoothctl
power on
agent on
scan on
# place PS4 controller in pairing mode
connect %MAC% / pair %MAC%
trust %MAC%
```
