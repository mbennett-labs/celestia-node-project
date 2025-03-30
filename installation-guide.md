# Celestia Light Node Installation Guide

This guide provides step-by-step instructions for setting up a Celestia Light Node and monitoring dashboard on an Ubuntu VPS.

## Prerequisites

- Ubuntu 24.04 LTS or similar Linux distribution
- Minimum 2 CPU cores
- Minimum 4GB RAM
- At least 60GB storage
- Root or sudo access
- Open SSH access

## Node Installation

### 1. System Preparation

Update your system and install dependencies:

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install -y curl git jq build-essential wget
```

### 2. Install Go

```bash
# Download and install Go 1.21.1
wget https://go.dev/dl/go1.21.1.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.21.1.linux-amd64.tar.gz

# Add Go to your PATH
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source $HOME/.profile

# Verify installation
go version  # Should show go version go1.21.1 linux/amd64 or newer
```

### 3. Install Celestia Light Node

```bash
# Clone the repository
cd $HOME
git clone https://github.com/celestiaorg/celestia-node.git
cd celestia-node

# Install the celestia node
make build
sudo make install

# Verify installation
celestia version
```

### 4. Initialize the Light Node

```bash
# Create a directory for the light node
mkdir -p $HOME/.celestia-light-node

# Initialize the light node (for mainnet)
celestia light init --p2p.network celestia
```

This will generate a key for your node. Make sure to save the mnemonic phrase securely!

### 5. Create a Systemd Service

```bash
# Create a systemd service file
sudo tee /etc/systemd/system/celestia-light.service > /dev/null << EOF
[Unit]
Description=Celestia Light Node
After=network-online.target

[Service]
User=$USER
ExecStart=$(which celestia) light start --core.ip https://rpc-celestia.pops.one --p2p.network celestia
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable celestia-light
sudo systemctl start celestia-light

# Check status
sudo systemctl status celestia-light
```

### 6. Configure Resource Limits

To ensure the node doesn't consume excessive resources:

```bash
# Create a directory for the override file
sudo mkdir -p /etc/systemd/system/celestia-light.service.d/

# Create resource limits for the service
sudo tee /etc/systemd/system/celestia-light.service.d/override.conf > /dev/null << EOF
[Service]
CPUWeight=50
MemoryLimit=2G
IOWeight=50
EOF

sudo systemctl daemon-reload
sudo systemctl restart celestia-light
```

## Dashboard Installation

### 1. Install Node.js

```bash
# Install Node.js and npm
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify installation
node -v  # Should show v20.x or newer
npm -v   # Should show 9.x or newer
```

### 2. Create Dashboard Application

```bash
# Create project directory
mkdir -p ~/celestia-dashboard
cd ~/celestia-dashboard

# Initialize npm project
npm init -y

# Install required packages
npm install express ejs axios systeminformation node-cron
```

### 3. Create Dashboard Files

Create the `app.js` file as shown in the [dashboard.js](./dashboard/app.js) file in this repository.

Create the `views/dashboard.ejs` template as shown in the [dashboard.ejs](./dashboard/views/dashboard.ejs) file in this repository.

### 4. Create a Systemd Service for the Dashboard

```bash
# Create a systemd service file for the dashboard
sudo tee /etc/systemd/system/celestia-dashboard.service > /dev/null << EOF
[Unit]
Description=Celestia Node Dashboard
After=network.target

[Service]
User=$USER
WorkingDirectory=/home/$USER/celestia-dashboard
ExecStart=/usr/bin/node app.js
Restart=on-failure
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=celestia-dashboard
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable celestia-dashboard
sudo systemctl start celestia-dashboard

# Check status
sudo systemctl status celestia-dashboard
```

## Accessing the Dashboard

For security reasons, the dashboard is not directly exposed to the internet. Access it through an SSH tunnel:

```bash
# From your local machine (not on the VPS)
ssh -L 8080:localhost:3000 your_username@your_vps_ip
```

Then open your browser and navigate to:
```
http://localhost:8080
```

## Troubleshooting

### Node Issues

If your node isn't syncing:

```bash
# Check node status
sudo systemctl status celestia-light

# View logs
journalctl -u celestia-light -f

# Restart the node
sudo systemctl restart celestia-light
```

### Dashboard Issues

If your dashboard isn't working:

```bash
# Check dashboard status
sudo systemctl status celestia-dashboard

# View logs
journalctl -u celestia-dashboard -f

# Restart the dashboard
sudo systemctl restart celestia-dashboard
```

## Maintenance

### Updating the Node

```bash
cd ~/celestia-node
git pull
make build
sudo make install
sudo systemctl restart celestia-light
```

### Checking Node Syncing Status

```bash
curl -s http://localhost:26659/header/synced | jq
```
